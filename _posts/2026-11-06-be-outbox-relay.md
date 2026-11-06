---
title: "commit은 됐는데 이벤트가 없다 — AFTER_COMMIT 발행을 Outbox relay로 바꾸며 직접 따져 본 것들"
description: "Transactional Outbox에 relay를 직접 붙이며 정한 것, 재 본 것, 자동으로 복구하지 못하는 것"
date: 2026-11-06 16:40:00 +0900
categories: [kafka, event-driven]
tags: [kafka, transactional-outbox, relay, shedlock, idempotency, debezium]
---

## 1. 왜 relay가 필요했나

지난주에는 주문 확정을 Kafka 이벤트로 내보냈다. 흐름은 이랬다.

1. 주문 DB save (확정 트랜잭션 commit)
2. `@TransactionalEventListener(AFTER_COMMIT)`에서 `kafkaTemplate.send`

2번에서 유실되면 어떻게 될까. 실제로 만들어 봤다. 기존 코드의 새 DB에서 주문 두 건을 확정하고, 한 건은 commit 뒤 send 직전에 멈췄다.

| | 주문 A (send 직전 중단) | 주문 B (정상) |
|---|---|---|
| 주문 | CONFIRMED | CONFIRMED |
| 결제 · 재고 예약 | SUCCESS · CONSUMED | SUCCESS · CONSUMED |
| 발행 기록 `published_at` | **null** | 있음 |
| 구매 활동 | **0행** | 1행 |

주문은 확정됐고 돈도 나갔는데, 확정됐다는 사실은 broker 어디에도 없다. **복구할 방법이 없다.** 이유는 주문 DB 저장과 Kafka send가 서로 다른 트랜잭션이기 때문이다. commit이 끝난 뒤의 send는 그 바깥이라, 실패하면 다시 보낼 무언가가 없다.

그래서 이렇게 바꿨다.

```java
// 같은 트랜잭션
주문 확정 save()
order_outbox에 이벤트 save()   // PENDING
```

이제 "보낼 사실"이 주문과 함께 DB에 원자적으로 남는다. 대신 **그 행을 읽어서 보낼 누군가가 반드시 필요해진다.** 그게 relay다. 아니면 outbox 테이블에 CDC를 붙여, CDC가 알아서 메시지를 보내게 할 수도 있다.

## 2. relay냐 CDC냐

CDC(Debezium)를 쓰면 binlog를 읽어 보내니, 아래에 나오는 고민 상당수가 사라진다. 그래도 relay를 직접 만들었다.

- Debezium은 Kafka Connect 클러스터, MySQL binlog 설정, 커넥터 운영까지 따라온다. 하지만 러닝 커브(팀단위일수록 더욱 더)와 운영 비용이 크다.
- CDC가 알아서 해 준다는 말이 **구체적으로 무엇을 대신해 주는지**는, 그 일을 직접 해 봐야 비교할 수 있다고 봤다.

코드를 쓰는 비용이 많이 낮아진 지금은 같은 흐름에 Debezium을 붙여 차이를 재 보는 것이 다음 할 일이다.

## 3. relay를 직접 짜며 답해야 했던 질문들

relay는 outbox에서 read 후 Kafka에 send가 전부인 것 같지만, 실패할 여지가 곳곳에 있다.

| 질문 | 이번에 한 것                                                                                                                                                                                 | Debezium이면                                                             |
|---|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------|
| offset은 어떻게 하나 | relay는 Kafka offset이 아니라 **outbox 행의 상태**(PENDING → SENDING → PUBLISHED, id 순)를 자기 위치로 쓴다. broker가 돌려준 partition · offset은 추적용으로 행에 남긴다                     | 커넥터가 binlog 위치를 자기 offset으로 관리한다                          |
| 발송이 실패하면 | 그 행만 backoff(10초 × 3^(n-1), 상한 5분, full jitter)로 미룬다. 3번 실패하면 `BLOCKED`로 멈추고 ERROR 로그를 남긴다 → 원인을 고친 뒤 개발자 개입 + 재개                                     | 커넥터가 재시도한다. 대신 커넥터가 멈추면 모든 이벤트가 멈춘다           |
| 소비할 때 실패하면, DLQ에 들어가면 | consumer는 3회 재시도 뒤 DLT로 보낸다. DLT를 읽어 DB에 옮기고 관리자 API로 재처리하는 방식으로 정했다(아직 구현 안 함)                                                                       | **똑같이 남는다**                                                        |
| broker가 eventId 중복을 확인해 주나 | **안 해 준다.** idempotent producer는 한 producer 세션 안의 재시도만 걸러 낸다. relay가 다시 보낸 것은 새 record다. 중복은 consumer의 eventId unique가 거른다                                | **똑같이 남는다** (Debezium도 at-least-once)                             |
| 순서는 | key = 주문 ID → 같은 주문은 같은 partition, 그 안에서는 보낸 순서. 다른 주문 사이 순서는 partition이 여러 개라 보장되지 않는다                                                               | partition 사이 순서는 똑같이 없다                                        |
| 이벤트가 DB에 많이 쌓이면 | relay 조회는 `(status, next_attempt_at, id)` 인덱스로 차례가 된 행만 읽는다. 5만 행 중 대기 100행일 때 101행만 읽었다. 저장 공간은 보존 7일, PUBLISHED만 지운다 (백업해서 NAS, S3 등에 보관) | binlog에서 읽으니 **넣자마자 지워도 된다.** 테이블 증가 문제 자체가 없다 |
| 파티셔닝 기준은 | `created_at` 범위 파티셔닝을 검토했지만, MySQL은 모든 unique 키에 파티션 컬럼을 요구해 `event_id` unique와 부딪힌다. 보류                                                                    | 위 이유로 거의 필요 없다                                                 |
| relay만의 고민 | 주기 · 묶음 크기 · 겹침 방지(ShedLock) · 보내다 죽은 행(lease)                                                                                                                               | **Debezium이 정말로 없애 주는 것이다**                                   |

"Debezium이면 이 고민 안 해도 된다"는 말은 절반만 맞았다. polling · 주기 · 잠금 · lease · 테이블 증가는 사라지지만, consumer 중복 제거 · DLT · 순서는 그대로 남는다.

## 4. relay 한 회차 그리고, 보내다 죽었을 때

relay 한 회차의 실행 흐름은 이렇다.

1. 보낼 차례가 된 행(`PENDING`, 또는 기한이 지난 `SENDING`)을 id 순서로 최대 200개 고른다.
2. 행마다 짧은 트랜잭션으로 `SENDING`으로 바꾸고 commit → 트랜잭션 밖에서 send.
3. broker 응답(ack)을 받으면 다시 짧은 트랜잭션으로 `PUBLISHED`와 partition · offset을 기록한다.

send를 트랜잭션 안에 넣지 않은 이유 : 넣으면 "ack는 왔는데 commit이 실패"했을 때 Kafka에는 있고 DB는 안 보냈다고 믿는다. 응답을 기다리는 동안 DB 연결과 행 잠금도 쥐고 있게 된다. 외부 시스템과 소통은 다른 트랜잭션으로.

`SENDING`을 둔 대가 : 보내다 프로세스가 죽으면 행이 `SENDING`에 남는다. 다시 고르는 규칙이 없으면 누락이 STATUS만 바뀌어 생긴다. 그래서 `SENDING`으로 바꿀 때 기한을 같이 적는다(지금 + 1분, lease :  SENDING 표시의 유효 기한). 기한이 지나면 relay가 그 행을 다시 고른다. 기한은 relay의 응답 대기(35초)보다, 그 대기는 producer의 timeout 시간(`delivery.timeout.ms` 30초)보다 길게 잡았다.

<br>

**이 지점에서 실제로 멈춰본 테스트**

broker가 ack를 준 뒤, `PUBLISHED`를 저장하기 전.

| 순서 | 일어난 일 |
|---|---|
| 1 | 상품 7×2, 상품 8×1 주문 확정 → outbox PENDING 1행 |
| 2 | relay가 보냄 → broker partition 2 · **offset 2** → 구매 활동 저장 |
| 3 | `PUBLISHED` 저장 트랜잭션 실패 → 프로세스 종료. 행은 `SENDING` |
| 4 | 새 JVM으로 relay 재시작 → lease가 지날 때까지 42.7초 기다림 → 같은 eventId · 같은 payload를 다시 보냄 → partition 2 · **offset 3** |
| 5 | consumer: "같은 사실의 재전달 — no-op" |

| 결과 | 값 |
|---|---|
| outbox | 같은 eventId 1행, 최종 PUBLISHED |
| broker | 같은 eventId의 record **2개** (offset 2, 3) |
| 구매 활동 | 상품 7×2, 8×1 그대로 |
| 재고 · 포인트 · PG 호출 | 확정 직후와 같음 |

**relay는 누락 대신 중복을 택한다.**

같은 사실이 두 번 갈 수 있고, 그 중복을 걸러 결과를 한 번으로 만드는 것은 consumer다. 주문은 확정 직후에 이미 맞고, 구매 활동은 몇 초 늦게라도 결국 같은 값에 도착한다(최종적 일관성).

## 5. relay의 주기에 대한 고민

relay의 주기에 대한 고민이 들었다. `@Scheduled(fixedDelay = 1s)`, 앞 회차가 끝나고 1초 뒤에 다음 회차를 돈다. 1초마다 DB를 읽는 게 부담이지 않을까.

실제로 broker 3대 · `acks=all` 환경을 띄우고, 보낼 행이 없을 때 relay가 DB에 보내는 문장을 60초 동안 셌다(`performance_schema`).

| relay가 1초에 보내는 것 | 횟수 |
|---|---|
| 차례가 된 행 조회 (인덱스 range, 평균 0.44ms) | 0.98회 |
| ShedLock 잠금 UPDATE + 해제 UPDATE | 0.98 + 0.98회 |
| COMMIT · `SET autocommit` | 1.97 · 3.93회 |
| **합계** | **약 9문장, DB 시간 약 7.7ms** |

1초 동안 DB가 relay 때문에 일하는 시간이 7.7ms다. 연결도 잠깐 빌렸다 돌려준다(측정 내내 Hikari active 0). 주기를 0.1초로 줄이면 한가할 때 지연은 줄지만 문장이 초당 90개가 된다. 주기 1초가 정하는 것은 **한가할 때의 지연(최대 1초)과 한가할 때의 DB 비용**이고, 1초로 충분했다.

그런데 주기가 처리량까지 정하지는 않았다.

## 6. TPS 100건을 넘어 300건까지 버티려면

부하 가정은 확정 초당 100건이었다. 그만큼 outbox 행을 넣어 봤다.

| | 처음 (한 행씩 ack를 기다림) | 바꾼 뒤 |
|---|---|---|
| 2,000행이 한꺼번에 쌓였을 때 다 보내는 시간 | 35.7초 (**초당 56건**) | 17.9초 (**초당 112건**) |
| 초당 100건 × 30초 — 가장 많이 쌓인 양 · 가장 오래 기다린 이벤트 | 1,119행 · 15.1초 | **351행 · 3.0초** |
| 초당 300건 × 30초 | 6,990행 · 92.9초 | 4,200행 · 23.3초 |

처음 구조로는 100건도 못 따라갔다. 원인은 주기가 아니었다.

- **한 행씩 ack를 기다렸다.** 행 하나에 약 12ms(트랜잭션 2개 + broker 왕복)가 걸려 초당 56건이 상한이었다.
- **꽉 찬 회차 뒤에도 1초를 쉬었다.** 200건을 다 골랐다는 건 아직 밀려 있다는 뜻인데도.

그래서 두 가지를 바꿨다.

```
전:  1번 행 [SENDING으로 변경 → 보내기 → ack 기다림 → 결과 기록] → 2번 행 [...] → …
후:  1번 행 [SENDING으로 변경 → 보내기] → 2번 행 [SENDING으로 변경 → 보내기] → … → 200번 행
     → 1번 ack 받아 결과 기록 → 2번 ack 받아 결과 기록 → …
```

- 회차의 행을 먼저 모두 보내고, ack는 id 순서로 기다린다. 같은 순간에 보낸 record를 producer가 묶어 전송해서 broker 왕복이 겹친다. 같은 주문의 순서는 같은 partition 안에서 보낸 순서로 지켜진다.
- 꽉 찬 회차면 쉬지 않고 바로 다음 회차를 돈다. 단, 한 번에 30초까지만 돈다(ShedLock 잠금 기한 5분 안).

대가로 "실패하면 멈춘다"의 뜻이 달라졌다. 전에는 앞 행이 실패하면 뒤 행을 보내지 않았다. 지금은 뒤 행이 이미 함께 나갔으므로, 뒤 행은 자기 결과대로 기록하고 실패한 행만 재시도로 미룬다. send 호출 자체가 바로 실패하는 경우(broker 정보를 아예 못 얻음)에만 그 뒤 행을 보내지 않고 멈춘다.

이제 100건은 버티지만 300건은 추가로 테스트가 필요하다. 남은 병목은 행마다 트랜잭션 2개의 COMMIT(로컬 Docker MySQL에서 한 번에 약 3ms)이다. 300건을 위한 선택지는 이렇다.

| 순서 | 방법 | 기대 | 대가 |
|---|---|---|---|
| 1 | relay 한 대 그대로, DB TX를 회차 단위로 묶는다. 200건 SENDING 변경을 TX 1개로, 결과 기록을 TX 1개로. COMMIT이 400번에서 2번이 된다 | 건당 DB 비용이 크게 줄어 300건을 넘을 것으로 예상(측정 안 함) | 결과 기록 TX가 실패하면 그 회차 200건이 모두 SENDING으로 남고 lease 뒤 다시 나간다. 중복은 consumer가 거르지만 broker record는 그만큼 늘어난다 |
| 2 | relay 여러 개, 주문 ID로 나눈다 (`order_id % N`마다 ShedLock 잠금 하나) | 대략 N배 | 같은 주문은 항상 같은 relay가 보내 같은 주문 안의 순서는 지켜진다. N을 바꾸면 담당이 바뀌는 순간 순서가 깨질 수 있고, DB 연결 · 조회도 N배 |
| 3 | relay 여러 개, 행 단위로 가져간다 (`SELECT … FOR UPDATE SKIP LOCKED`) | N배 | 같은 주문의 이벤트를 서로 다른 relay가 동시에 보낼 수 있어 같은 주문 안의 순서도 깨진다 |
| 4 | CDC (Debezium) | binlog를 읽어 보내 relay의 DB 비용이 없어진다 | Kafka Connect 운영 |

"처리량이 모자라면 relay를 늘린다"가 첫 수가 아닌 이유가 표에 있다. 병목이 relay 개수가 아니라 relay 한 대 안의 DB 비용이라, 늘리기 전에 1번이 먼저다. 그리고 relay를 몇 대 쓰든 partition이 여러 개인 이상 다른 주문 사이의 순서는 보장되지 않는다. 늘리면 오히려 같은 주문 안의 순서까지 신경 쓸 것이 생긴다.

## 7. 자동으로 복구하지 못하는 것

relay가 혼자 해결하지 못하고 개발자가 개입하는 경우가 남는다.

- **BLOCKED.** broker 3대를 모두 멈추고 주문을 넣어 봤다. 같은 eventId로 재시도하다 **101초 뒤 세 번째 실패에서 BLOCKED**가 됐다. broker를 되살려도 BLOCKED는 그대로다. 원인을 고친 뒤 관리 포트로 `POST /actuator/outbox/{eventId}`를 부르면 PENDING으로 돌아가고, 다음 회차(1초 안)에 PUBLISHED가 됐다. 실패 종류를 나누지 않아서, 잠깐 멈춘 broker 때문에도 BLOCKED가 생길 수 있다. 그래서 BLOCKED를 감지하는 수단(ERROR 로그, 목록 SQL)이 같이 있어야 한다.
- **BLOCKED를 Kafka DLT로 보내지 않은 이유.** broker 장애로 생긴 BLOCKED는 DLT로도 보낼 수 없다. 보내야 또 장애난다. 보내는 쪽의 실패는 원본(SoT)인 DB에 둔다.
- **PUBLISHED ≠ 소비 완료.** consumer를 강제 종료하고 주문을 넣으면, outbox는 PUBLISHED인데 구매 활동은 없다. broker까지는 갔으니 relay를 다시 돌릴 일이 아니라 consumer 쪽을 봐야 한다. 둘을 구분하는 SQL(PUBLISHED인데 소비 완료 기록이 1분 넘게 없는 행)을 운영 문서에 두었다.
- **consumer의 DLT.** 지금은 DLT를 읽는 consumer가 없다. 들어간 record는 7일 뒤 사라진다. DB로 옮겨 관리자가 재처리 · 폐기하는 방식으로 정했다.

## 8. Outbox가 새로 보장한 것, consumer가 계속 책임지는 것

| Outbox + relay가 새로 보장한 것 | consumer가 계속 책임지는 것 |
|---|---|
| 확정된 주문의 사실은 DB에 남고, 결국 broker에 간다 (at-least-once) | 같은 사실이 여러 번 와도 결과를 한 번으로 만든다 (eventId 완료 기록) |
| 확정이 rollback되면 사실도 남지 않는다 (같은 트랜잭션) | 소비 실패의 재시도와 DLT |
| 보내는 쪽 실패는 DB에서 찾고 재개할 수 있다 (BLOCKED) | 받은 순서와 늦게 도착한 사실의 처리 |

Outbox는 "보낸다"를 보장할 뿐 "한 번만 처리된다"를 보장하지 않는다. 그건 처음부터 consumer의 몫이었고, relay를 붙이면서 더 분명해졌다.

## 9. 남은 것

- 300건: 1번(회차 단위 트랜잭션)을 구현하고 측정까지.
- CDC: Debezium을 같은 흐름에 붙여서 relay와 비교해보기
