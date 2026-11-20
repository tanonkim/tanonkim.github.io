---
title: "Batch Insert 성능 개선 — rewriteBatchedStatements 옵션, Querydsl-SQL 비교하기"
description: "JPA saveAll, JDBC batch, Querydsl-SQL로 100만 행을 넣어 보고, rewriteBatchedStatements와 Querydsl 코드에서 차이의 원인을 찾은 기록"
date: 2026-11-20 10:30:00 +0900
categories: [database, mysql]
tags: [batch-insert, rewritebatchedstatements, jdbc, jpa, querydsl, spring-batch, mysql, jfr]
---

주간 랭킹 batch는 chunk마다 1,000행을 `ranking_snapshot_input`에 넣는다. 같은 행을 여러 방법으로 100만 행 넣어 보고, 왜 차이가 나는지를 Hibernate·Connector/J·Querydsl 코드와 MySQL 쪽에서 확인했다.

## 1. 실험 조건

| 항목 | 값 |
|---|---|
| 테이블 | `ranking_snapshot_input` — 7개 열, PK `id` AUTO_INCREMENT, unique `(snapshot_id, product_id, order_id)` |
| 단위 | chunk 1,000행 = 트랜잭션 하나 (모든 방법 같음) |
| 반복 | 방법마다 2만 행으로 예열 → 회차마다 방법들을 번갈아 3회, 평균 |
| 재는 것 | 쓰는 시간만 (행 만들기는 뺌) · MySQL이 받은 INSERT 문장 수 (`Com_insert`) · MySQL이 문장을 처리한 시간 (performance_schema) |
| MySQL | 8.4.10, Docker 기본 설정 (buffer pool 128MB, binlog ON, `sync_binlog=1`, `innodb_flush_log_at_trx_commit=1`) |
| 라이브러리 | Spring Data JPA 3.4.4 · Hibernate 6.6.11 · Spring Batch 5.2.2 · Connector/J 9.1.0 · Querydsl-SQL 5.1.0 |
| 앱 | JDK 21 · Mac (Apple Silicon, 12코어) · MySQL과 같은 PC (Docker Desktop) |

| 방법 | 쓰는 것 | rewriteBatchedStatements |
|---|---|---|
| 1번 | JPA `saveAll` | (상관없음, 이유는 4-1) |
| 2번 | Spring Batch `JdbcBatchItemWriter` | 끔 |
| 3번 | Spring Batch `JdbcBatchItemWriter` | 켬 |
| 4번 | Querydsl-SQL `addBatch` | 끔 / 켬 |
| 5번 | Querydsl-SQL `addBatch` + `setBatchToBulk(true)` | 끔 |
| 4번·5번 변형 | 원인을 확인하려고 4번·5번에서 하나씩만 바꾼 것 (4-5, 4-6) | |

## 2. 방법별 코드

**1번 JPA saveAll**

```java
transactionTemplate.executeWithoutResult(status -> repository.saveAll(chunk));
```

**2·3번 JdbcBatchItemWriter** — 점수 계산 step의 writer와 같다. 2번과 3번은 코드가 같고 연결 설정만 다르다.

```java
JdbcBatchItemWriter<InputRow> writer = new JdbcBatchItemWriterBuilder<InputRow>()
    .dataSource(dataSource)
    .sql("INSERT INTO ranking_snapshot_input (snapshot_id, product_id, ...) VALUES (?, ?, ?, ?, ?, ?, ?)")
    .itemPreparedStatementSetter((row, ps) -> { ps.setString(1, row.snapshotId()); /* ... */ })
    .build();
```

```yaml
# 3번 — HikariCP 연결 설정 (URL에 ?rewriteBatchedStatements=true 를 붙여도 같다)
data-source-properties:
  rewriteBatchedStatements: true
```

**4번 Querydsl-SQL addBatch**

```java
SQLInsertClause insert = queryFactory.insert(input);
for (InputRow row : chunk) {
    insert.columns(input.snapshotId, input.productId, /* ... */)
          .values(row.snapshotId(), row.productId(), /* ... */)
          .addBatch();
}
insert.execute();
```

**5번 Querydsl-SQL batchToBulk** — 4번에 한 줄을 더한다.

```java
insert.setBatchToBulk(true);
```

Querydsl-SQL을 쓸 때 알아 둘 것:
- **모듈이 따로 있다.** `querydsl-jpa`와 다른 `querydsl-sql` 모듈이다. `JPAQueryFactory` 대신 `SQLQueryFactory`를 쓴다.
- **Q클래스도 다르다.** JPA용 Q클래스는 쓸 수 없고, DB 스키마에서 만든 `RelationalPathBase`가 필요하다 (`querydsl-sql-codegen`).
- **연결은 `SpringConnectionProvider`로 받는다.** 그래야 Spring 트랜잭션에 참여하고, chunk가 실패하면 함께 rollback된다.

## 3. 결과 (100만 행, 3회 평균)

| 방법 | 초당 넣은 행 | 시간 | MySQL이 받은 INSERT 문장 | 보낸 데이터 |
|---|---:|---:|---:|---:|
| 1번 JPA saveAll | 5,669 | 176.4초 | 1,000,000 | 217MB |
| 2번 JDBC batch, rewrite 끔 | 6,697 | 149.3초 | 1,000,000 | 230MB |
| **3번 JDBC batch, rewrite 켬** | **58,008** | **17.2초** | **1,000** | **103MB** |
| 4번 Querydsl-SQL, rewrite 끔 | 6,506 | 153.7초 | 1,000,000 | 229MB |
| 4번 Querydsl-SQL, rewrite 켬 | 46,466 | 21.5초 | 1,000 | 103MB |
| 4번 변형: rewrite 켬 + `quote()` | 53,740 | 18.6초 | 1,000 | 103MB |
| 5번 Querydsl-SQL batchToBulk | 12,674 | 78.9초 | 1,000 | 104MB |
| 5번 변형: 100행씩 한 문장으로 | 23,740 | 42.1초 | 10,000 | 105MB |

- **두 분류로 나뉜다.** 1·2·4번(rewrite 설정 false)은 행마다 한 문장을 보내고 초당 6천 행 정도이다. 묶어 보내는 쪽은 초당 1만~6만 행이다.
- **rewrite 한 줄의 효과:** 약 9배 빨라지고, 보내는 데이터도 절반 아래로 준다. 행마다 반복되던 `INSERT INTO … (열 목록) VALUES`가 1000행에 한 번만 간다.
- **같은 문장 수인데 시간이 다르다.** 3·4·5번은 모두 chunk마다 한 문장인데 시간은 17초에서 79초까지 벌어진다. 이 차이는 MySQL이 아니라 앱 안에 있다 (4-5, 4-6). // 왜일까?
- **10만 행과 같은 결과다.** 10만 행에서도 3번은 초당 약 6만 행이었다. 이 크기에서는 테이블이 MySQL 메모리(128MB)를 넘어도 눈에 띄는 차이가 없었다.

## 4. 왜 이런 차이가 날까?

### 4-1. JPA saveAll — 행마다 INSERT가 바로 나간다

`saveAll`은 반복문이다. 행마다 `save`를 부르고, `save`는 엔티티가 새것이면 `persist`, 아니면 `merge`를 부른다 (`SimpleJpaRepository`).

```text
saveAll(entities) = for (e : entities) save(e)
save(e)           = isNew(e) ? persist(e) : merge(e)    // isNew: id가 null인가
```

- 우리 엔티티는 id가 비어 있어서 `persist`로 간다.
- id를 직접 채우는 엔티티면 `merge`로 가서, 행마다 SELECT가 하나 더 나간다. 흔히 말하는 "saveAll이 느리다"의 또 다른 경우다.

이유는 id 전략에 있다. Hibernate는 id를 INSERT가 만들어 주는 엔티티(IDENTITY)면 insert batch를 끈다.

- `InsertCoordinatorStandard` 주석 번역

```text
IDENTITY id
→ persist 순간 영속성 컨텍스트에 넣으려면 id가 필요하다
→ id는 INSERT를 해야 나온다
→ 트랜잭션 안이면 persist 하자마자 INSERT를 실행한다 (AbstractSaveEventListener)
→ 모아 둘 틈이 없다 = batch 불가
```

그래서 `hibernate.jdbc.batch_size`를 설정해도 **IDENTITY**면 소용없다. 이번 측정에서 1번이 rewrite가 켜진 연결을 썼는데도 문장 수가 100만인 이유도 이것이다. Hibernate가 `executeBatch`를 부르지 않으니 rewrite가 끼어들 자리가 없다. MySQL에는 SEQUENCE가 없어서, JPA로 묶으려면 id 전략부터 바꿔야 한다.

1번이 2번보다 18% 느린 건 Hibernate가 행마다 하는 일 때문이다. 엔티티 값을 복사하고, 영속성 컨텍스트에 등록하고, commit 때 1,000개를 변경 감지한다.

### 4-2. JdbcBatchItemWriter의 addBatch는 "모아 두기"일 뿐이다 (rewrite false)

`JdbcBatchItemWriter`는 1,000행을 `ps.addBatch()`로 모으고, `ps.executeBatch()`를 한 번 부른다. 여기까지 보면 한 번에 보낼 것 같다. 하지만 실제로 보내는 방식은 드라이버(Connector/J)가 정한다. `ClientPreparedStatement.executeBatchInternal`을 요약하면 이렇다.

```text
executeBatch()
  if (rewriteBatchedStatements 켬)
      INSERT … VALUES 문이면  → executeBatchWithMultiValuesClause   // 여러 VALUES 한 문장
      그 밖에 4개 이상이면    → executePreparedBatchAsMultiStatement // ;로 이은 여러 문장
  → executeBatchSerially                                            // 하나씩 실행
```

rewrite가 꺼져 있으면 `executeBatchSerially`(하나씩 순차적으로)로 간다. 모은 1,000개를 **반복문으로 하나씩 보내고**, 하나마다 MySQL의 응답을 기다린다. 왕복이 1,000번이다. `Com_insert`가 2번에서 행 수와 같게 나온 것이 그 증거다.

### 4-3. rewriteBatchedStatements는 무엇인가

옵션을 켜면 드라이버가 모은 행들을 다시 써서 한 문장으로 보낸다.

```sql
-- rewrite 끔: 문장 1,000개
INSERT INTO ranking_snapshot_input (...) VALUES ('weekly-…', 1, 1, 1, '2026-08-17 00:00:00', …);
INSERT INTO ranking_snapshot_input (...) VALUES ('weekly-…', 2, 2, 2, '2026-08-17 00:00:01', …);
...
-- rewrite 켬: 문장 1개
INSERT INTO ranking_snapshot_input (...) VALUES ('weekly-…', 1, 1, 1, …),('weekly-…', 2, 2, 2, …),…;
```

- **크기 제한:** 한 문장이 MySQL의 `max_allowed_packet`을 넘지 않게 드라이버가 나눈다 (`computeBatchSize`). 너무 크면 몇 문장으로 쪼개진다.
- **안 되는 경우:** `LAST_INSERT_ID()`를 쓰는 INSERT는 다시 쓰지 않는다.
- **트레이드오프:** 행마다 "몇 행이 바뀌었나"를 알 수 없다. 드라이버는 모든 칸을 `Statement.SUCCESS_NO_INFO`(-2)로 채운다. `JdbcBatchItemWriter`의 `assertUpdates`는 **0만** 잡으므로 통과하지만, **실제로는 아무것도 검사하지 못한다.**

### 4-4. DB 관점 — 시간은 어디에 쓰였나

2~5번은 드라이버 호출을 감싸 시간을 나눠 쟀고, MySQL이 문장을 받아 끝낼 때까지 쓴 시간은 performance_schema에서 확인

| 100만 행 (초)       | 전체 | MySQL INSERT 처리 | 왕복 + 드라이버 | 앱 (드라이버 밖) | commit |
|---------------------|---:|---:|---:|---:|---:|
| 2번 rewrite 옵션 끔 | 149.3 | 34.1 | **112.2** | 1.3 | 1.7 |
| 3번 rewrite 옵션 켬 | 17.2 | 7.9 | 5.6 | 2.0 | 1.8 |

- **2번은 시간의 75%가 왕복이다.** 문장 100만 개가 하나씩 오가며 문장당 약 0.11ms를 기다린다. MySQL이 실제로 일한 시간은 23%뿐이다.
- **MySQL 작업도 줄어든다.** 같은 100만 행인데 MySQL 처리 시간이 34.1초에서 7.9초로 준다. 행 쓰기와 인덱스 검사는 그대로이고, 파싱·문장 시작과 끝 처리가 1,000회에서 1회로 줄기 때문이다.
- **commit은 같다.** 둘 다 chunk마다 한 번 commit하므로 1.7~1.8초로 같다.

| MySQL이 하는 일 | 행마다 한 문장 (1·2·4번, rewrite 끔) | chunk마다 한 문장 (3·4번 rewrite 켬, 5번) |
|---|---|---|
| 행 쓰기, unique 인덱스 검사, binlog 행 데이터 | 1,000회 | 1,000회 (같음) |
| 왕복, 파싱, 문장 시작과 끝 처리 | **1,000회** | **1회** |
| commit (redo·binlog fsync) | 1회 | 1회 (같음) |

이번 왕복(문장당 약 0.11ms)은 Docker Desktop의 포트 전달까지 포함한 값이다. 앱과 DB가 다른 서버면 왕복이 더 길어질 수 있고, 그만큼 2번의 손해가 커진다.

### 4-5. Querydsl-SQL addBatch — 같은 JDBC batch인데 왜 순수 JDBC Batch + rewrite보다 25% 느린가

Querydsl의 `addBatch`는 행을 쌓기만 한다. `execute` 때 `AbstractSQLInsertClause.createStatements`가 이렇게 한다.

```text
for (쌓인 행 하나하나)
    INSERT SQL 글자를 처음부터 다시 만든다 (SQLSerializer)
    그 글자로 "같은 문장의 PreparedStatement"를 찾는다 (HashMap)
    값을 넣고 JDBC addBatch()
→ PreparedStatement.executeBatch()
```

끝에서 JDBC `executeBatch`를 부르므로, 묶일지는 4-2·4-3의 rewrite 규칙을 그대로 따른다. 그래서 4번은 rewrite 설정에 따라 2·3번과 똑같이 갈린다.

rewrite를 켜도 3번보다 느린 이유를 세 단계로 좁혔다.

1. **MySQL 실행은 같다.** 문장 수(1,000), 보낸 데이터(103MB), MySQL 처리 시간(7.9초 대 8.0초)이 모두 같다. 차이는 드라이버 밖, 즉 Querydsl 안이다 (3번 2.0초, 4번 7.0초).
2. **JFR CPU 샘플로 보면** 4번 CPU의 50%가 Querydsl이고, 그중 36%가 `SQLTemplates.requiresQuotes`다. 행당 메모리 할당도 3번의 5.8KB보다 훨씬 많은 27.5KB이고, 대부분이 정규식 내부 객체(`Pattern`, `boolean[]`, `int[]`)다.
3. **코드로 보면** Querydsl은 SQL을 만들 때 테이블·컬럼 이름마다 "따옴표로 감싸야 하나"를 검사한다. 이 검사가 이름마다 `String.matches`를 두 번 부르고, `String.matches`는 부를 때마다 정규식을 새로 컴파일한다. **행마다 SQL을 다시 만드니** 1,000행 chunk 하나에 정규식 컴파일이 16,000번이다.

**확인:** 이 검사만 건너뛰게 하면 된다. 이름을 늘 따옴표로 감싸는 `quote()`를 켜면 Querydsl이 검사 없이 감싼다.

```java
new Configuration(MySQLTemplates.builder().quote().build());   // `ranking_snapshot_input` (`snapshot_id`, …)
```

1. quote() 없이
```
insert into ranking_snapshot_input (snapshot_id, product_id, order_id, quantity, confirmed_at, created_at, updated_at)
values ('weekly-20260817-r1', 1, 1, 1, '2026-08-17 00:00:00', '2026-08-24 01:00:00', '2026-08-24 01:00:00'), ...
```
2. quote() 적용
```
insert into `ranking_snapshot_input` (`snapshot_id`, `product_id`, `order_id`, `quantity`, `confirmed_at`, `created_at`, `updated_at`)
values ('weekly-20260817-r1', 1, 1, 1, '2026-08-17 00:00:00', '2026-08-24 01:00:00', '2026-08-24 01:00:00'), ...
```



| 100만 행 | 시간 | 드라이버 밖 | Querydsl의 CPU 몫 | 행당 할당 |
|---|---:|---:|---:|---:|
| 3번 JDBC | 17.2초 | 2.0초 | — | 5.8KB |
| 4번 Querydsl | 21.5초 | 7.0초 | 50% | 27.5KB |
| 4번 변형: Querydsl + `quote()` | 18.6초 | 3.6초 | 18% | 8.9KB |

(Querydsl의 CPU 몫과 행당 할당은 30만 행 × 3회를 JFR로 따로 잰 값이다.)

3번보다 더 들던 앱 시간의 약 70%가 사라진다. 남은 차이(3번보다 8%)는 행마다 SQL을 다시 만들고 그 글자로 HashMap을 찾는 비용이다.

### 4-6. Querydsl-SQL batchToBulk — 한 문장으로 묶었는데 오히려 4.6배 느리다 - O(n²) 

`setBatchToBulk(true)`면 Querydsl이 쌓인 행을 `VALUES (…), (…), …` 한 문장으로 만들고, `executeUpdate`를 한 번 부른다. 드라이버의 rewrite가 필요 없다. MySQL이 받는 SQL도 3번과 같다. 그런데 78.9초가 걸렸고, 그중 60.7초가 드라이버 밖이었다.

- **JFR로 보면** CPU의 92%가 `LinkedList.get`이다. `AbstractSQLClause.setParameters`가 부른다.
- **코드로 보면** Querydsl은 값마다 "어느 컬럼의 값인지"를 `LinkedList`에 담아 두고, 값을 넣을 때 `get(i)`로 하나씩 꺼낸다. `LinkedList.get(i)`는 매번 리스트를 처음(또는 끝)부터 따라가므로 **O(n²)**이다. 1,000행이면 값이 7,000개라 문장 하나에 약 1,200만 번을 따라간다.
- **확인:** 한 문장을 100행(값 700개)으로 줄이면, O(n²) 몫은 1/10이 되어 CPU 몫이 약 53%로 떨어져야 한다. 측정값은 48%였다.

**MySQL 처리 시간도 더 길었다** (5번 12.8초, 3번 7.9초). SQL은 같다(SQL 문제는 아님). 5번은 chunk마다 약 60ms를 혼자 쓰는 동안 MySQL이 대기한다. 3번에 chunk 사이 60ms 텀만 넣어 봤더니, 같은 SQL인데 MySQL 처리 시간이 두 배가 됐다 (10만 행, 0.89초 → 1.72초, 3회 모두). 문장이 쉬지 않고 들어오면 MySQL도 빠르고, 틈이 있으면 문장마다 콜드 상태에서 시작한다. 

## 5. 쿼리가 하나로 묶였나 확인하는 법

시간만 보면 착각하기 쉽다. MySQL이 받은 INSERT 문장 수를 직접 세는 게 확실하다.

```sql
SHOW GLOBAL STATUS LIKE 'Com_insert';   -- 넣기 전후 차이
```

chunk 1,000행을 넣었는데 차이가 1이면 묶인 것이고, 1,000이면 묶이지 않은 것이다. 다만 5번처럼 문장 수가 1이어도 느릴 수 있으니, 문장 수는 "묶였나"만 알려 준다.

## 6. 정리

| 상황 | 쓸 것 | 이유 |
|---|---|---|
| JPA 엔티티, IDENTITY id | 대량 insert에는 `saveAll`을 쓰지 않는다 | Hibernate가 batch를 끈다. `batch_size`로 해결되지 않는다 |
| JDBC batch (`JdbcBatchItemWriter`, `JdbcTemplate.batchUpdate`) | `rewriteBatchedStatements=true` | 설정 한 줄로 8.7배. 켰는지는 `Com_insert`로 확인 |
| Querydsl-SQL | `addBatch` + rewrite + `MySQLTemplates.builder().quote()` | `quote()` 없이는 이름마다 정규식 컴파일로 25% 느리다 |
| Querydsl-SQL `batchToBulk` | 큰 chunk에는 쓰지 않는다 | Querydsl 5.1.0은 값 개수의 제곱으로 느려진다 |
