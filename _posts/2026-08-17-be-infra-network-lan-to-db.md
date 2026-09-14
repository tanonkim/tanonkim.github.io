---
title: "네트워크(랜선에서 DB까지 - 서비스 A 관점)"
date: 2026-08-17 19:36:00 +0900
categories: [infra, network]
tags: [network, linux, tomcat]
---

> 실제 서비스명, 도메인, 리소스 식별자 등은 보안을 위해 예시 값으로 바꿨습니다.
{: .prompt-info }

**원문**

OSI 7계층의 최하단인 물리 신호부터 시작해서 리눅스 커널, 톰캣 스레드 풀, 스프링 부트, 그리고 DB 커넥션까지 트래픽이 흐르는 전체 과정을 5단계로 구조화한 스터디 커리큘럼 지도입니다. [1]<br>이 흐름대로 공부하시면 하드웨어, OS, 어플리케이션이 어떻게 유기적으로 맞물리는지 완벽하게 마스터할 수 있습니다.

:globe_with_meridians: 랜선에서 DB까지: 엔드투엔드(End-to-End) 트래픽 흐름 스터디 지도<br>[1단계: L1~L3 네트워크 인입] ➡️ [2단계: L4 & Linux 커널 큐] ➡️ [3단계: 파일 시스템 & FD] ➡️ [4단계: WAS 스레드 풀 & 스프링] ➡️ [5단계: DBCP & 데이터베이스]

:round_pushpin: 1단계: 네트워크 인입과 주소 바인딩 (OSI 1~3계층)

네트워크 관점 (L1~L3): 사용자의 요청이 물리 신호(L1)로 들어와, MAC 주소(L2)를 거쳐, IP 주소(L3)를 기반으로 내 서버의 네트워크 카드(NIC)에 도달합니다.<br>OS/서버 관점: 스프링 서버가 구동될 때 이미 리눅스 커널에 ServerSocket.bind() 시스템 콜을 날려 특정 IP와 포트(0.0.0.0:8080)를 선점(바인딩)하고 대기(Listen) 상태로 들어간 구조를 이해합니다.<br>핵심 스터디 키워드: 비트(Bit)와 패킷(Packet), IP 주소 체계, 포트 바인딩, netstat을 이용한 포트 점유 확인. [2]<br>:round_pushpin: 2단계: TCP 핸드셰이크와 커널 대기 큐 (OSI 4계층 & Linux 커널)

네트워크 관점 (L4): 클라이언트와 서버가 신뢰성 있는 연결을 위해 TCP 3-Way Handshake(SYN ➡️ SYN-ACK ➡️ ACK)를 수행합니다. [3]<br>OS/서버 관점: 트래픽이 폭발할 때 톰캣이 패킷을 미처 가져가지 못해도 리눅스 커널이 완충 작용을 해줍니다. 핸드셰이크 단계에 따라 두 개의 큐에 패킷이 쌓입니다.<br>SYN 큐: SYN 패킷을 받고 대기하는 공간 :arrow_right: 커널 파라미터 tcp_max_syn_backlog로 조절.<br>Accept 큐: 연결이 완료되어 톰캣이 가져가길 기다리는 공간 :arrow_right: 커널 파라미터 somaxconn 및 톰캣 accept-count로 조절.

핵심 스터디 키워드: TCP 3-Way Handshake, 커널 파라미터(sysctl), 연결 거절(Connection Refused) 장애.<br>:round_pushpin: 3단계: 파일 시스템과 소켓의 실체 (Linux OS 레벨)

OS/서버 관점: 리눅스는 "모든 것을 파일로 관리"합니다. 커널 Accept 큐에서 톰캣의 Selector 스레드가 요청을 꺼내오는 순간(accept() 발생), 하나의 네트워크 소켓이 생성되고 이는 파일 시스템의 File Descriptor(FD, 정수형 파일 번호)로 등록됩니다.<br>성능 및 장애 관점: 동시 접속자가 1만 명으로 늘어나면 FD도 1만 개가 열립니다. OS가 프로세스당 열 수 있는 파일 최대치(ulimit -n)를 낮게 잡아두면, 소켓을 더 이상 만들지 못해 Too many open files 에러가 터지며 서버가 다운됩니다.<br>핵심 스터디 키워드: File Descriptor(FD), 리눅스 파일 시스템, ulimit 제한, 소켓 고갈 및 fs.file-max.<br>:round_pushpin: 4단계: 톰캣 스레드 풀과 스프링 부트 (OSI 7계층 어플리케이션)

서버 관점: 이제 트래픽이 커널을 벗어나 자바 가상 머신(JVM)과 톰캣 영역으로 들어옵니다. 소켓에 실제 HTTP 데이터(L7)가 들어오면 톰캣의 Worker 스레드 풀이 움직입니다.<br>동작 원리: 톰캣은 미리 만들어둔 일꾼 스레드(max-threads, 기본 200개) 중 하나를 꺼내어 해당 소켓(FD)을 할당합니다. 이 스레드가 비로소 스프링의 DispatcherServlet을 깨우고, 우리가 짠 Controller :arrow_right: Service 로직을 한 줄 한 줄 실행합니다.<br>핵심 스터디 키워드: 톰캣 NIO 구조, Worker 스레드 풀, TaskQueue 대기열, Read Timeout.<br>:round_pushpin: 5단계: DB 커넥션 풀과 최종 데이터 처리 (End-to-End 완결)

서버/DB 관점: 비즈니스 로직을 수행하던 스프링 스레드는 DB 조회를 위해 외부 DB 서버와 통신해야 합니다. 매번 소켓을 열면 대규모 트래픽에서 포트가 고갈되므로, 미리 맺어둔 DB 소켓 묶음인 DBCP(HikariCP)에서 커넥션을 빌려옵니다.<br>도미노 병목 시나리오: DB 쿼리가 느려지면 HikariCP 커넥션(maximumPoolSize, 기본 10개)이 고갈됩니다 :arrow_right: 스프링 스레드(200개)가 DB 연결을 받으려고 줄 서서 대기합니다 :arrow_right: 톰캣 스레드가 모두 묶여 신규 요청을 못 받습니다 :arrow_right: 커널 Accept 큐와 FD가 무한정 쌓이다가 폭발합니다 :arrow_right: 전체 서버가 마비됩니다.<br>핵심 스터디 키워드: HikariCP 설정, Connection Timeout, TIME_WAIT 소켓 재사용(tcp_tw_reuse).

:bulb: 이 커리큘럼으로 스터디를 시작할 때의 장점

이 구조로 스터디를 진행하면 "스프링 yml 설정 한 줄이 리눅스 커널과 네트워크 선로에 어떤 나비효과를 불러오는가?"를 인프라 관점으로 이해하게 됩니다. 단순 코드 구현을 넘어 대규모 트래픽을 아키텍처 레벨에서 다루는 시야가 열립니다.<br>이 흐름 중에서 가장 먼저 세부적으로 파고들어 가고 싶은 '몇 단계'가 있으신가요?

1~2단계의 리눅스 커널 큐 설정과 TCP 핸드셰이크 상세 분석<br>3~4단계의 리눅스 FD 제한 풀기와 톰캣 스레드 풀 최적화 매핑<br>5단계 중심의 HikariCP 고갈로 인한 전체 서버 다운 시뮬레이션<br>스터디의 첫 번째 깊은 탐구 주제를 골라보세요!

목적: "스프링 yml 한 줄이 리눅스 커널과 네트워크 선로에 어떤 나비효과를 부르는가"를 우리 서비스 실제 값으로 추적한다.<br>원칙: 개념은 짧게, 우리 값 · 확인 명령 · 위험 신호를 길게. 읽는 문서가 아니라 장애 때 여는 문서로 쓴다.<br>대상 스택: `service-a-server` — Java 21 · Spring Boot 4.1 · Tomcat · 가상 스레드 ON · HikariCP · RDS MySQL 8 single-AZ · ECS Fargate 뒤 ALB

### 전체 흐름

**네트워크 흐름**

```yaml
① 클라이언트가 SYN 전송
        ▼
② 커널: SYN 큐에 등록, SYN+ACK 응답        ← 앱은 아직 아무것도 모름
        ▼
③ 클라이언트 ACK 도착 → 커널이 Accept 큐로 이동
        ▼
④ 앱이 accept() 호출 → 커널이 큐에서 꺼내줌
        ▼
⑤ 연결 소켓 생성 + FD 번호 발급             ← 커널 → 앱 소유권 이전
        ▼
⑥ 앱이 read(fd)로 HTTP 요청 읽기
        ▼
⑦ 처리 후 write(fd)로 응답
        ▼
⑧ close(fd) → FD 회수
```

```yaml
손님 ──▶ [ 문 ] ──▶ [ 대기 벤치 ] ──▶ 직원이 안내 ──▶ [ 테이블 ]
             ▲            ▲                  ▲               ▲
         리스닝 소켓    Accept 큐          accept()      연결 소켓 = FD
         
         

동시에 받는 수      성격
  [사용자]
     │
  [ALB]                    ~무제한          그냥 다 밀어넣는다
     │
─────┼──────────────────────────────────  여기부터 우리 태스크
     │
  커널 Accept 큐              100          ★ 대기실. 넘치면 조용히 드롭
     │
  FD (소켓 = 파일)             ?           ★ 하드 상한. 닿으면 전면 장애
     │
  Tomcat max-connections     8192          ★ 우리가 실제로 여는 문의 크기
     │
  가상 스레드                 무제한         ← 아무것도 막지 않는다
     │
  힙 1.5GB                   (암묵)         ★ 넘으면 프로세스 즉사
     │
  HikariCP                     10          ★ 진짜 처리 창구
     │
  RDS max_connections         170          여유 있음
```

### 1. 소켓 - 리스닝 소켓, 연결 소켓

|  | **리스닝 소켓** (listening socket) | **연결 소켓** (connected socket) |
|---|---|---|
| 개수 | 서버당 **1개** | **연결마다 1개** (1만 명 접속 = 1만 개) |
| 하는 일 | 받기만 한다. 데이터 못 주고받음 | 실제 데이터를 주고받음 |
| 만들어지는 때 | 서버 기동 시 1번 | `accept()` 할 때마다 |

- **4-tuple**

```yaml
( 출발지 IP , 출발지 포트 , 목적지 IP , 목적지 포트 )
   1.xx.xx.xx   51234       10.xx.xx.xx   8080     
   1.xx.xx.xx   51235       10.xx.xx.xx   8080     
   9.xx.xx.xx   51234       10.xx.xx.xx   8080     
```

넷 중 하나라도 다르면 다른 연결이다. (참고 : 포트 고갈은 내가 다른 서버로 연결을 걸 때 생긴다.)

- **3번의 요청**

서버는 커널에 시스템 콜을 3번 날린다. (시스템 콜 : 프로그램이 커널에게 이거 해주세요~ 부탁하는 공식 창구)

```yaml
### 커널
① socket()   "소켓 하나 만들어 주세요"       → 식당 자리를 하나 임대
② bind()     "0.0.0.0:8080 을 제 걸로"    → 그 자리에 간판을 걸고 주소를 등록
③ listen()   "손님 받겠습니다"          → 문을 열고, 대기 벤치를 놓는다
```

0.0.0.0 : 이 서버가 가진 모든 IP라는 와일드 카드<br>listen()이 중요한 이유 : 호출이 실행되는 순간 커널 안에 큐(대기 벤치)가 생긴다.<br>이 셋이 끝나면 accept()를 호출하고, 손님이 올 때까지 아무것도 안한다.

참고

- listen()까지는 커널영역, accept()부터 애플리케이션 영역

**용어**

- **바인딩(bind)**: 소켓에 "나는 이 IP의 이 포트다"라고 주소를 붙이는 것.
- **`Address already in use`**: 다른 프로세스가 이미 그 포트를 bind 해놨다는 뜻.
- **리슨(listen)**: 연결 요청을 받을 준비 완료. 이때 큐가 만들어짐.
- **accept()**: 완성된 연결 하나를 커널에서 꺼내오는 것.

### 2. 커널의 두 큐

왜 큐가 두개인가 : SYN큐 : 인사 진행 중(3-way handshake) + Accept큐 : 앱이 가져가기만 하면 되는 큐

```yaml
클라이언트                        서버
    │ ──── SYN ──────────────▶ │   "연결하고 싶습니다"
    │ ◀─── SYN + ACK ───────── │   "좋습니다, 저도 준비됐어요"
    │ ──── ACK ──────────────▶ │   "확인했습니다"
                                   → 이제부터 연결 성립
```

세번의 인사 → 전부 커널이 진행

|  | SYN 큐 | Accept 큐 |
|---|---|---|
| 담긴 것 | **반쯤 열린(half-open)** 연결 | **완성된** 연결 |
| 크기 조절 | `net.ipv4.tcp_max_syn_backlog` | `min( listen()의 backlog 인자 , net.core.somaxconn )` |
| 차는 이유 | 3-way handshake가 안끝남 : 네트워크 문제 | 앱이 accept()를 못따라감 → 느림 |

`somaxconn` : Accept큐 크기는 앱이 요청한 값과 커널 상한 중 더 작은 값. 예) backlog를 1000으로 요청해도 커널 somaxconn이 128이면 실제 큐는 128

- 큐가 넘친다면

1. SYN큐가 넘칠 때

새 SYN을 말없이 버린다. → 클라이언트는 답이 없으니 SYN을 재전송하고.. 결국 timeout

이걸 악용하는게 SYN Flood 공격. SYN만 잔뜩 보내고, ACK를 안보내면 큐가 반쯤 열린 상태로 가득차서 정상 손님이 못 들어옴 ⇒ SYN쿠키가 방어책. 리눅스에서 `net.ipv4.tcp_syncookies`는 보통 기본으로 켜져 있다.

1. Accept큐가 넘칠 떄

인사는 끝났는데, 자리가 없다면 커널은 : 마지막 ACK를 못본척 한다 (기본 동작)

Accept큐 오버플로우는 서버 애플리케이션 로그에 한 줄도 남지 않는다. → 앱이 그 연결 조차 몰라서 이 층이 무서운 이유

- Connection refused : 포트에 리스닝 소켓이 없을 때.
- 큐 오버플로우 : 리스닝 소켓은 멀쩡히 있다. → 조용히 있다가 타임아웃 발생

### 3. FD (File Discriptor)

리눅스는 모든 것을 파일로 관리한다. → 인터페이스를 하나로 통일하기 위해

```yaml
read(3, buf, 1024);    // 3번이 텍스트 파일이면 → 파일에서 읽기
read(3, buf, 1024);    // 3번이 소켓이면       → 네트워크에서 받기
read(3, buf, 1024);    // 3번이 파이프면       → 다른 프로세스에서 받기
```

read/write/close

```yaml
Accept 큐 (커널)
     │
     │  앱이 accept() 호출
     ▼
연결 소켓 생성 + FD 번호 발급   ←── 여기가 두 층의 경계
     │
     ▼
앱이 그 FD로 read/write
```

accept()의 반환값이 FD(번호표)

FD를 소비하는 것들

```yaml
FD를 소비하는 것들:
  · 클라이언트 연결 소켓        (접속자 수만큼)
  · 리스닝 소켓                (1개)
  · DB · Redis · 외부 API 연결  (전부 소켓)
  · 열어놓은 로그 파일
  · 프로그램이 읽는 라이브러리 파일 (자바면 JAR 수백 개)
  · epoll 인스턴스, eventfd 같은 커널 객체
  · 파이프
```

상한에 닿으면 — 2종류 에러

| 에러 메시지 | errno | 뜻 |
|---|---|---|
| `Too many open files` | **EMFILE** | **이 프로세스**가 자기 한도를 다 씀 |
| `Too many open files in system` | **ENFILE** | **시스템 전체**가 한도를 다 씀 |

장애가 났는데, 로그가 텅 비어있다 → FD 고갈의 전형적인 모습

- FD leak

close()를 하지 않으면 FD가 회수되지 않는다. 손님은 갔는데, 번호표는 돌려받지 못한 상태

```yaml
① 클라이언트가 SYN 전송
        ▼
② 커널: SYN 큐에 등록, SYN+ACK 응답        ← 앱은 아직 아무것도 모름
        ▼
③ 클라이언트 ACK 도착 → 커널이 Accept 큐로 이동
        ▼
④ 앱이 accept() 호출 → 커널이 큐에서 꺼내줌
        ▼
⑤ 연결 소켓 생성 + FD 번호 발급             ← 커널 → 앱 소유권 이전
        ▼
⑥ 앱이 read(fd)로 HTTP 요청 읽기
        ▼
⑦ 처리 후 write(fd)로 응답
        ▼
⑧ close(fd) → FD 회수
```

### 두 층을 하나로 잇기

```yaml
① 클라이언트가 SYN 전송
        ▼
② 커널: SYN 큐에 등록, SYN+ACK 응답        ← 앱은 아직 아무것도 모름
        ▼
③ 클라이언트 ACK 도착 → 커널이 Accept 큐로 이동
        ▼
④ 앱이 accept() 호출 → 커널이 큐에서 꺼내줌
        ▼
⑤ 연결 소켓 생성 + FD 번호 발급             ← 커널 → 앱 소유권 이전
        ▼
⑥ 앱이 read(fd)로 HTTP 요청 읽기
        ▼
⑦ 처리 후 write(fd)로 응답
        ▼
⑧ close(fd) → FD 회수
```

두 층은 독립적이지 않다. 한쪽이 막히면 다른 쪽으로 역류한다.

Accept()큐가 넘친다면 증상의 원인은 거의 항상 앱 쪽에 있음

| 클라이언트가 보는 것 | 가장 유력한 층 | 확인 명령 |
|---|---|---|
| **즉시** `Connection refused` | 프로세스가 죽었거나 bind 안 됨 (0부) | `ss -ltn` 에 그 포트가 있나 |
| **한참 매달리다** 타임아웃 | Accept 큐 오버플로우 (1부) | `netstat -s | grep listen` 증가 여부 |
| 연결은 되는데 응답이 느림 | 앱 처리 지연 (오늘 범위 밖) | 앱 지표 |
| 산발적으로 연결 실패 + 로그 없음 | FD 고갈 (2부) | `ls /proc/<PID>/fd | wc -l` vs `ulimit -n` |
| 장애인데 **로그가 텅 빔** | FD 고갈 유력 (2부) | 위와 동일 |

## 서비스 A 관점

### 전제

일반적인 "랜선→DB" 커리큘럼은 **4단계에서 톰캣 워커 스레드 200개**를 전제한다. 우리는 다르다.

```yaml
# api/src/main/resources/application.yml
spring:
  threads:
    virtual:
      enabled: true      # Java 21 가상 스레드
```

가상 스레드 때문에 고정 워커 스레드 풀을 사용하지 않는다.

**바뀌는점 3가지 (가상 스레드 사용)**

|  | 일반 커리큘럼(플랫폼 스레드) | 서비스 A(가상 스레드) |
|---|---|---|
| 동시 처리 상한 | `server.tomcat.threads.max` = 200 | **`server.tomcat.max-connections` = 8192**(기본값, 우리는 미설정) |
| 초과분이 가는 곳 | 톰캣 내부 `TaskQueue` → `accept-count` → 커널 Accept 큐 | 사실상 큐가 없음. 8192까지 그냥 다 받는다 |
| 백프레셔(요청 거절) 주체 | **스레드 풀이 자연스럽게 막아준다** | **막는 게 없다.** DB 풀 고갈이 유일한 방어선 |

고정 스레드 풀은 병목이 아닌 보호장치. 200개만 처리하고, 나머지는 큐에서 기다리게 해서, DB가 감당할 수 있는 만큼만 흘려보냈다. 가상 스레드를 켜면 그 보호막이 사라지고, 8192개의 요청이 동시에 HikariCP 커넥션 10개를 향해 달려든다.

**보호막**

```yaml
hikari:
  connection-timeout: 3000     # 3초 안에 커넥션 못 얻으면 즉시 실패
```

서비스 A의 설계는 줄 세우기가 아닌, 빨리 실패시키기.

### 현재 구조

```yaml
[사용자]
   │  HTTPS 443
   ▼
[ALB]  internet-facing · idle_timeout 60s · dereg_delay 90s
   │  ALB SG → App SG (8080만)
   ▼
[Fargate Task]  1 vCPU / 2GB · 2~6개 · Spot 100% · stopTimeout 120s
   │  ├─ Linux 커널: SYN 큐 → Accept 큐 → FD(소켓)
   │  ├─ Tomcat NIO: Acceptor → Poller → 가상 스레드(요청당 1개)
   │  ├─ JVM: heap 75% ≈ 1.5GB · G1GC · ExitOnOutOfMemoryError
   │  └─ HikariCP: 10개 고정 풀
   │  App SG → RDS SG (3306만)
   ▼
[RDS MySQL 8]  db.t4g.small · single-AZ · wait_timeout 3600s
```

### **1단계 — 네트워크 인입과 주소 바인딩 (L1~L3)**

물리신호(L1) → 프레임/MAC(L2) → IP 패킷(L3) → NIC 도착<br>서버 쪽에서는 기동 시점에 이미 bind() + listen() 시스템 콜로 0.0.0.0:8080을 선점하고 대기

서비스 A에서는

Fargate awsvpc 네트워크 모드라서 태스크마다 전용 ENI(NIC)를 받는다. → 포트가 이미 사용중이라는 충돌 없고 다른게 생김

1. ALB 타깃 그룹이 target_type = “ip”이다. 인스턴스가 아니라, 태스크의 ENI IP를 직접 등록한다.
2. 태스크가 교체되면 IP가 바뀐다. → 고정 IP를 어디에도 적으면 X
3. SG 규칙을 IP가 아닌, 앞 SG를 참조한다. referenced_security_group_id → IP가 바뀌어도 규칙이 따라온다.
4. 태스크는 프라이빗 서브넷에 있고, 아웃바운드는 fck-nat 인스턴스를 거친다. 이 NAT가 유휴 커넥션을 끊는 지점.

확인 명령

```yaml
# 서비스의 태스크 ENI IP (ALB 타깃과 일치해야 함)
aws ecs list-tasks --cluster my-cluster --service-name my-api \
  --query 'taskArns[]' --output text | xargs -n1 \
  aws ecs describe-tasks --cluster my-cluster --tasks \
  --query 'tasks[].attachments[].details[?name==`privateIPv4Address`].value'

# ALB 타깃 헬스
aws elbv2 describe-target-health --target-group-arn <TG_ARN>

# 로컬에서 리스닝 확인 (Mac은 lsof, 리눅스는 ss)
lsof -nP -iTCP:8080 -sTCP:LISTEN
ss -ltnp 'sport = :8080'
```

**위험 신호**

- ALB 타깃이 unhealthy → /actuator/health/readiness가 200을 맞추지 못함 (Flyway 마이그레이션 지연, DB 연결 실패일 가능성이 높음)
- startPeriod : 60s안에 기동이 안끝나면 컨테이너가 계속 죽고 살아난다. JVM + Flyway 마이그레이션 시간 실측 필요

**키워드**

`bind`/`listen` 시스템 콜, `awsvpc` vs `bridge`, ENI, 타깃 그룹 `ip` vs `instance`, SG 참조 체인, `ss`/`netstat` 읽는 법

### 2단계 — TCP 핸드셰이크와 커널 대기 큐 (L4)

개념

```yaml
Client                          Server
  │ ──────── SYN ─────────────▶ │  ← SYN 큐에 등록 (net.ipv4.tcp_max_syn_backlog)
  │ ◀────── SYN-ACK ──────────  │
  │ ──────── ACK ─────────────▶ │  ← Accept 큐로 이동 (min(somaxconn, backlog))
                                │  ← 앱이 accept()로 꺼내감
```

**두 개의 큐**

앱의 자원이 다 차서 accept()를 못해도 커널이 Accept 큐에서 완충해준다. 그 큐마저 차면 커널이 SYN을 조용히 드롭하고, 클라이언트는 타임아웃 또는 Connection refused를 본다.

애플리케이션 로그에는 아무것도 남지 않는다. → 가장 취약한 점

서비스 A에서는

- Spring에서 Accept 큐 깊이를 정하는건 server.tomcat.accept-count(기본 100). 미 설정
- 실제 큐 깊이는 min(accept-count, net.core.somaxconn). 커널 값이 더 작으면 앱 설정은 의미가 없다.
- Fargate에서는 sysctl을 마음대로 바꾸지 못한다. ECS 태스크 정의의 systemControls로 일부 net.*만 지정할 수 있다.
- ALB가 앞에 있으므로 클라이언트는 ALB 노드. ALB는 커넥션을 재사용하므로(keep-alive) SYN 폭주 가능성은 직접 노출보다 훨씬 낮다. 대신 ALB ↔ 태그 keep-alive 정합이 중요해진다.

**왜 Tomcat keep-alive(70s) > ALB idle(60s)인가?**

앱보다 ALB가 먼저 끊어야 한다.

앱이 먼저 끊으면 ALB가 아직 살아있다고 착각하는 커넥션에 요청 → RST 받고 → 502 Bad Gateway

재현이 안되는 502의 전형적인 원인

```yaml
좋음:  앱 keep-alive 70s  >  ALB idle 60s   → ALB가 먼저 정리, 502 없음
나쁨:  앱 keep-alive 50s  <  ALB idle 60s   → 앱이 먼저 끊음, 502 레이스
```

**확인명령**

```yaml
# 커널 큐 파라미터 (컨테이너 안에서 — 현재는 진입 불가, 아래 체크리스트 참고)
sysctl net.core.somaxconn net.ipv4.tcp_max_syn_backlog

# Accept 큐가 넘치고 있는지 (Recv-Q = 대기 중, Send-Q = 큐 최대치)
ss -ltn | grep 8080
#   Recv-Q 가 Send-Q 에 근접 → 앱이 accept 를 못 따라가고 있다

# 큐 오버플로우 누적 카운터 (증가하면 드롭 중)
netstat -s | grep -iE "listen queue|SYNs to LISTEN|overflowed"
```

**위험신호**

- `netstat -s`의 "times the listen queue of a socket overflowed"가 **증가한다** → Accept 큐 포화. 4·5단계 병목이 여기까지 역류한 것.
- ALB 메트릭 `TargetConnectionErrorCount` 증가.
- ALB 메트릭 `HTTPCode_ELB_502_Count` 증가 → keep-alive 정합 문제 의심.

**스터디 키워드**

3-Way Handshake, SYN 큐 / Accept 큐, `somaxconn`, `tcp_max_syn_backlog`, `accept-count`, backlog 계산식, ALB idle timeout, 502 레이스, `ss -ltn`의 Recv-Q/Send-Q 의미

### 3단계 - 파일 디스크립터와 소켓 (Linux OS)

리눅스는 모든 것을 파일로 다룬다. `accept()`가 성공하는 순간 소켓 하나가 만들어지고 **File Descriptor(정수 번호)** 로 등록된다. 동시 접속 1만이면 FD 1만 개다.

프로세스 당 FD 상한에 닿으면 `Too many open files`가 터진다. 이때 새 소켓뿐 아니라, 로그파일, JAR 읽기, DB 커넥션까지 전부 못연다. → 전면 장애

서비스 A에서 FD를 쓰는 것들

```yaml
FD 총량 ≈ (ALB↔앱 keep-alive 커넥션 수)
        + (HikariCP 커넥션 10)
        + (Redis 커넥션)
        + (S3/SQS HTTPS 커넥션)
        + (로그·JAR·클래스패스 파일)
        + (에폴 인스턴스 등 커널 객체)
```

`ulimits`를 태스크 정의에 지정하지 않음. Fargate 플랫폼 기본값을 그대로 쓰는 중이고, 그 값이 얼마인지 확인된 바 없다. → 체크리스트 항목이다.

한 가지 다행인 점: HikariCP가 커넥션을 10개로 고정하므로 DB 쪽 FD는 폭증하지 않는다. 커넥션 풀의 존재 이유 중 하나가 정확히 이것이다(5단계 TIME_WAIT과 함께).

### 파일 디스크립터와 소켓

FD고갈 : 동시 접속 1만명일 때

서비스 A에서는 훨씬 낮은 트래픽에서도 닿을 수 있음 : 가상스레드 때문에

고정 스레드 풀 200개짜리 서버는 동시에 살아있을 수 있는 소켓 수를 스레드 풀이 간접적으로 눌러준다. 우리는 max-connections [기본값] 8192까지 커넥션을 그냥 받는다. FD 상한이 동시성 상한이다.

```yaml
FD 상한(ulimit -n)  ≥  max-connections(8192)
                       + HikariCP(10)
                       + Redis / S3 / SQS 커넥션
                       + 로그·JAR·클래스패스 파일 (수백 개)
                       + epoll/eventfd 등 커널 객체
```

서비스 A는 태스크 정의에 ulimits를 지정하지 않음. Fargate 플랫폼 버전에 따라 달라져 왔는데, 컨테이너 안에서 직접 읽기.

만약 소프트 리밋이 1024라면, 8192개를 받겠다고 선언해놓고 1024번째 소켓에서 죽는다. 설정 두 개가 서로 모순인 상태로 배포되어 있는 것이고, 이건 부하 테스트 없이는 절대 드러나지 않는다.

**확인명령**

```yaml
# --- 컨테이너 진입 (ECS Exec) ---
aws ecs execute-command --cluster my-cluster \
  --task <TASK_ID> --container <CONTAINER_NAME> \
  --interactive --command "/bin/sh"
# 안 되면: 서비스에 enableExecuteCommand + 태스크 역할에
# ssmmessages:CreateControlChannel/CreateDataChannel/OpenControlChannel/OpenDataChannel 필요

# --- FD 상한 (이게 핵심) ---
cat /proc/1/limits | grep -i 'open files'
#   Max open files   1024    1048576    files   ← 앞이 soft, 뒤가 hard
ulimit -n            # 셸 기준 soft

# --- 현재 열린 FD 개수 ---
ls /proc/1/fd | wc -l

# --- FD가 뭘로 채워져 있는지 (소켓 / 파일 / epoll 비율) ---
ls -l /proc/1/fd | awk '{print $NF}' | sed 's/[0-9]\+//g' | sort | uniq -c | sort -rn | head

# --- 시스템 전체 상한 ---
cat /proc/sys/fs/file-max
cat /proc/sys/fs/file-nr     # 할당됨 / 미사용 / 최대
```

위험 신호

| 신호 | 어디서 보이나 | 의미 |
|---|---|---|
| `Too many open files` | 앱 로그 | 이미 늦었다. 소켓·로그·DB 커넥션 전부 못 연다 |
| `java.net.SocketException: Too many open files` | Tomcat Acceptor 스레드 | accept() 자체가 실패 → 커널 Accept 큐가 역류 (2단계로 전파) |
| `process.files.open` 이 계단식으로 우상향 후 안 내려옴 | Actuator/CloudWatch | **FD 릭.** 응답을 안 닫는 HTTP 클라이언트, 안 닫는 InputStream |
| 트래픽은 평평한데 FD만 증가 | 위와 동일 | 릭 확정. 트래픽 연동이면 정상 |
| 재시작하면 잠깐 괜찮아짐 | 배포 이력 | 릭의 전형적 증상 |

File Descriptor, `ulimit -n` soft/hard, `/proc/<pid>/limits`, `fs.file-max` vs `fs.nr_open`, ECS 태스크 정의 `ulimits`, Fargate에서 못 바꾸는 것들, epoll 인스턴스도 FD다, Micrometer `FileDescriptorMetrics`

### 톰캣 NIO와 가상스레드(L7)

```yaml
[커널 Accept 큐]
      │
      │  accept()
      ▼
 Acceptor 스레드 (플랫폼 스레드, 1개)
      │  소켓을 받아 Poller에 등록만 하고 즉시 복귀
      ▼
 Poller 스레드 (플랫폼 스레드, 1~2개 · epoll)
      │  "이 소켓에 읽을 데이터가 도착했다" 감지
      ▼
 Executor  ←── 여기가 우리가 바꾼 지점
      │
      ├─ 일반 커리큘럼: ThreadPoolExecutor (max-threads 200) — 재사용
      └─ service-a:    VirtualThreadExecutor — 요청마다 새 가상 스레드 1개, 그냥 버림
```

Acceptor와 Poller는 여전히 플랫폼 스레드이고, 개수가 1~2개라는 점.

가상스레드를 켜도 이 둘은 변경되지 않는다. → 여기가 막히면 앞단 전체가 멈춘다.

가상스레드가 바꾸는 것

```yaml
spring:
  threads:
    virtual:
      enabled: true
```

1. Tomcat 프로토콜 핸들러의 Executor → VirtualThreadExecutor
2. Spring의 applicationTaskExecutor → 가상스레드
3. server.tomcat.threads.max / threads.min-spare → 무시됨

⇒ max-connections가 8192로 올라감

**1. 캐리어 스레드 1개 : ecs 인스턴스가 1vCPU**

블로킹 I/O만 하면 문제없음. 가상 스레드가 알아서 언마운트 되니까.

문제는 pinning. Java21에서 synchronized 블록 안에서 블로킹을 하면 가상 스레드가 캐리어를 놓지 못한다. → 캐리어가 1개인데 하나가 고정되면 애플리케이션 전체가 선다.

TODO

`[미확인]` MySQL Connector/J는 커넥션 내부에 `synchronized`가 있어 JDBC 호출 중 pinning을 유발하는 것으로 알려져 있다. 우리는 이 조합(Java 21 + Connector/J + 가상 스레드 + 1 vCPU)을 정확히 밟고 있다. 부하 테스트 전에 pinning 여부부터 측정해야 한다. (참고 : **Java 24부터 synchronized pinning**이 제거됨)

**2.jstack이 안통한다.**

가상 스레드는 기존 스레드 덤프에 안 나온다.

```yaml
# Java 21+ 가상 스레드까지 포함한 덤프
jcmd 1 Thread.dump_to_file -format=json /tmp/vt.json
# HikariCP 대기 중인 가상 스레드가 몇 개인지 여기서 센다
```

**3.지표대시보드**

`tomcat.threads.busy` / `tomcat.threads.config.max`는 ThreadPoolExecutor를 전제한 지표다. `[미확인]` 가상 스레드에서는 0 또는 -1로 나오거나 아예 사라진다.

대체 지표:

| 봐야 할 것 | 지표 |
|---|---|
| 진행 중 요청 수 | `http.server.requests.active` (LongTaskTimer) |
| 실질 커넥션 수 | `process.files.open` (3단계) 또는 JMX `Catalina:type=ThreadPool,name="http-nio-8080"` → `connectionCount` 커스텀 게이지 |
| 진짜 병목 | `hikaricp.connections.pending` (5단계) |

**4.keep-alive 기본값이 ALB와 정확히 동률이다.**

앞에서 "앱 70s > ALB 60s"를 원칙으로 세웠다. 그런데 우리는 `[기본값]` 상태다.

- `server.tomcat.keep-alive-timeout` 미설정 → `connection-timeout` 값을 따라감
- `server.tomcat.connection-timeout` 미설정 → 톰캣 기본 **60초** `[기본값]`
- ALB idle timeout = **60초** `[확정]`

```yaml
server:
  tomcat:
    keep-alive-timeout: 70000        # ALB idle 60s 보다 확실히 김
    max-keep-alive-requests: -1      # 요청 수로 인한 커넥션 종료 제거
    max-connections: 2000            # 8192는 우리 자원 대비 과대 (아래 산술 참고)
    accept-count: 100
  shutdown: graceful
spring:
  lifecycle:
    timeout-per-shutdown-phase: 25s
```

**5.max-connections를 낮춰야 하는 이유**

`max-connections`는 스로틀이 아니라 OOM 방어선이다.

가상 스레드는 스레드를 안 쌓는다. 대신 **진행 중 요청 객체를 쌓는다.** 요청 하나당 가상 스레드 스택 + 요청/응답 버퍼 + 도메인 객체가 힙에 산다.

힙은 2GB의 75% ≈ **1.5GB** `[확정]`. 8192개 동시 요청이 각각 100KB만 잡아도 800MB다. `-XX:+ExitOnOutOfMemoryError` `[확정]`이 걸려 있으니, **OOM = 태스크 즉사**다. 그리고 죽은 태스크의 트래픽은 남은 태스크로 몰린다 → 연쇄.

스터디 키워드

Tomcat NIO Acceptor/Poller/Executor, `VirtualThreadExecutor`, 캐리어 스레드와 `jdk.virtualThreadScheduler.parallelism`, pinning과 JEP 491, `jcmd Thread.dump_to_file`, `http.server.requests.active`, keep-alive 정합과 502 레이스, `max-keep-alive-requests`, ThreadLocal → ScopedValue

### **5단계 — HikariCP와 RDS MySQL (종착지)**

커넥션 풀은 성능 최적화가 아닌 상한선

TCP커넥션 하나를 여는 비용(3Way handshake + TLS + MySQL 인증)이 쿼리 자체보다 비싸다. → 그래서 미리 열어두고 빌려 쓴다.

풀 크기가 곧 DB에 걸리는 부하의 하드 리밋이다.

서비스 A에서는

```yaml
hikari:
  maximum-pool-size: 10            # [기본값]
  connection-timeout: 3000         # [확정] 우리가 명시함
```

1. OSIV

false로 하기

true가 기본값

1. 커넥션 수명 정합

| 값 | 우리 상태 | 정합 |
|---|---|---|
| Hikari `max-lifetime` | `[기본값]` 1800s (30분) |  |
| RDS `wait_timeout` | `[확정]` 3600s (1시간) | ✅ 1800 < 3600 — 앱이 먼저 버린다. 정상 |
| Hikari `keepalive-time` | `[기본값]` 0 = 비활성 | ⚠️ 유휴 커넥션이 조용히 죽어도 모른다 |
| Hikari `leak-detection-threshold` | `[기본값]` 0 = 비활성 | ⚠️ 커넥션 릭을 못 잡는다 |

원칙 : 항상 앱이 먼저 끊는다.

```yaml
hikari:
  max-lifetime: 1500000            # 25분. wait_timeout(1h)보다 확실히 짧게
  keepalive-time: 60000            # 1분마다 유휴 커넥션 생존 확인
  leak-detection-threshold: 5000   # 5초 넘게 안 반납하면 스택 로그
  validation-timeout: 1000
```

1. DB 트래픽은 NAT를 안거친다. 그러나 외부 호출은 거친다.

RDS 트래픽은 같은 VPC 안이라 NAT를 안 탄다. 여기서 헷갈리면 안 된다.

NAT를 타는 건 S3/SQS/외부 API다.

1. db.t4g.small의 실제 한계

```
MySQL 기본 max_connections = {DBInstanceClassMemory/12582880}
db.t4g.small = 2 GiB = 2,147,483,648 bytes
  → 2147483648 / 12582880 ≈ 170개
```

태스크 6개 × 풀 10개 = **60개** + 마이그레이션/모니터링. `[확정]` 커넥션 수는 여유 있다.

진짜 병목은 커넥션이 아니라 t4g의 CPU 크레딧과 스토리지 IOPS다.

1. single-AZ + 헬스체크가 만드는 최악의 조합

```yaml
readiness에 db 포함  → DB 다운 → 전 태스크 unhealthy → ALB가 전부 내림
                     → 트래픽 0 → DB 복구돼도 헬스 통과까지 추가 지연
                     → ECS가 태스크를 계속 교체 (교체 폭풍)

readiness에 db 제외  → DB 다운 → 태스크는 살아있음 → 3초 만에 500 응답
                     → DB 복구되는 순간 즉시 정상화
```

1. Fargate Spot 100% × single-AZ

`[확정]` Spot 100%. 회수 통지는 SIGTERM + **120초**, `stopTimeout`도 120초. 그런데 앞단에 `deregistration_delay 90s`가 있다.

```yaml
ALB 드레이닝 90s  +  graceful shutdown  ≤ 120s 안에 끝나야 함
```

`[미확인]` ECS가 드레이닝 완료를 기다린 뒤 SIGTERM을 보내는지, 동시에 보내는지에 따라 여유가 완전히 달라진다. Spot 100% 환경에서 dereg_delay 90s는 길다. 30s 정도로 줄이는 걸 검토해야 한다(정상 배포 시 502를 안 만드는 선에서).

### 관측·재현, 그리고 지금 모르는 것들

재현 시나리오 (스터디용 실습)

1. 의도적으로 느린 엔드포인트 — `SELECT SLEEP(2)` 하나 만들고 k6/vegeta로 부하
   - 관찰: `hikaricp.connections.pending`이 언제 올라가는지, 3초 실패가 언제 시작되는지
   - 산수 검증: 풀 10 ÷ 2초 = **초당 5건**이 상한. 실제로 그런지 확인
2. Accept 큐 오버플로우 — 위 상태에서 `max-connections`를 50으로 낮추고 부하
   - 관찰: `/proc/net/netstat`의 `ListenOverflows` 증가, ALB `TargetConnectionErrorCount`
3. FD 상한 — `ulimits` nofile을 200으로 낮춘 태스크 정의로 배포 후 부하
   - 관찰: `Too many open files`가 **어디서 먼저 터지는지** (소켓? 로그?)
4. pinning — `Djdk.tracePinnedThreads=full` 켜고 JDBC 부하
   - 관찰: pinning 스택 발생 여부. 발생하면 병렬도 1 vs 8 처리량 비교
5. keep-alive 경합 — 앱 keep-alive를 50s(< ALB 60s)로 낮추고 저부하로 방치
   - 관찰: 502가 산발적으로 뜨는지. 뜬다면 현재 60s 동률도 같은 위험
