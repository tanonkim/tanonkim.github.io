---
title: "모니터링"
date: 2026-03-17 22:56:00 +0900
categories: [infra, monitoring]
tags: [hikaricp, mysql, apm]
---

{% raw %}
# **1. HikariCP 커넥션 풀 모니터링**

### 요약

`hikaricp_connections_pending`(대기 스레드 수)이 가장 중요한 지표이다. 이 값이 0이 아니면 커넥션 풀이 고갈 직전이므로 즉시 대응이 필요하다. `leak-detection-threshold`는 운영 환경에서 반드시 켜두기.

## 1.1 왜 커넥션 풀 모니터링이 필요한가

데이터베이스 커넥션은 비싼 자원이다. 하나의 커넥션을 생성하는데 TCP HandShake, 인증, 세션 초기화까지 20~50ms가 소요된다. 커넥션 풀은 이 비용을 줄이기 위해 미리 커넥션을 만들고 재사용하는 구조 → 풀의 상태를 모니터링 하지 않으면 다음의 장애가 발생한다.

- **커넥션 고갈**

  모든 커넥션이 사용 중이어서 새 요청이 connectionTimeout까지 대기하다 실패

- **커넥션 누수**

  커넥션을 빌려간 뒤 반환하지 않아 점진적으로 가용 커넥션이 줄어듬

- **Slow Query로 인한 풀 점유**

  느린 쿼리가 커넥션을 장시간 점유하여 다른 요청이 대기

### 1.2  HikariCP 핵심 설정 파라미터

```yaml
# application.yml
spring:
  datasource:
    hikari:
      pool-name: MyAppPool
      maximum-pool-size: 20          # 최대 커넥션 수
      minimum-idle: 5                # 최소 유휴 커넥션
      connection-timeout: 3000       # 커넥션 획득 대기 시간(기본 3000)
      idle-timeout: 600000           # 유휴 커넥션 유지 시간
      max-lifetime: 1800000          # 커넥션 최대 수명 (30분)
      leak-detection-threshold: 5000 # 5초 이상 미반환 시 누수 경고 로그
      validation-timeout: 5000       # 커넥션 유효성 검사 타임아웃
      register-mbeans: true          # JMX MBean 등록(모니터링 필수)
```

- 튜닝 가이드

| 파라미터 | 권장값 | 설명 |
| --- | --- | --- |
| maximum-pool-size | CPU코어 * 2 + 디스크 수 (보통 10~30)+ RDS Max Connection 미만 | connections = ((core_count × 2) + effective_spindle_count) |
| minimum-idle | maximum-pool-size와 동일하기 | HikariCP 공식 문서 권장. 트래픽 변동이 심하면 낮게 |
| connection-timeout | 3000~5000ms | 30초는 사용자 응답시간 초과, 빠르게 실패하는게 좋다 |
| leak-detection-threshold | 5000~10000ms | 운영환경에서 반드시 설정, 누수 로그가 반복되면 코드 수정 필요 |
| max-lifetime | DB의 wait_timeout보다 2-3분 짧게 | MySQL wait_timeout이 28800(8시간)이면 28500초 미만 |

DB의 스펙도 같이 고려해야 한다.

## 1.3 Spring Boot Actuator + Micrometer 메트릭 수집

**의존성 추가**

```yaml
<!-- pom.xml -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
```

**Actuator 설정**

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health, metrics, prometheus
  endpoint:
    health:
      show-details: always
  metrics:
    tags:
      application: my-app
```

### 수집되는 HikariCP 메트릭

HikariCP는 Micrometer와 자동 통합되어 다음 메트릭이 자동으로 수집된다.

```yaml
# 커넥션 풀 크기 관련
hikaricp_connections                    # 현재 총 커넥션 수
hikaricp_connections_active             # 현재 사용 중인 커넥션 수
hikaricp_connections_idle               # 현재 유휴 커넥션 수
hikaricp_connections_max                # 최대 풀 크기
hikaricp_connections_min                # 최소 풀 크기
hikaricp_connections_pending            # 커넥션 획득 대기 중인 스레드 수 ★ 핵심 지표


# 커넥션 획득/생성/사용 관련
hikaricp_connections_acquire_seconds    # 커넥션 획득까지 걸린 시간 (히스토그램)
hikaricp_connections_creation_seconds   # 새 커넥션 생성에 걸린 시간
hikaricp_connections_usage_seconds      # 커넥션 사용 시간 (빌려간~반환)


# 커넥션 생명주기
hikaricp_connections_timeout_total      # 타임아웃 발생 횟수 (누적)
```

### Prometheus에서 확인

```yaml
# 브라우저에서 직접 확인
GET http://localhost:8080/actuator/prometheus

# 특정 메트릭만
GET http://localhost:8080/actuator/metrics/hikaricp.connections.active
```

## 1.4 Grafana 대시보드 구성

### 핵심 패널 4가지

### **패널 1: 커넥션 풀 사용률 (가장 중요)**

```yaml
# 커넥션 풀 사용률 (%) - 80% 이상이면 경고
hikaricp_connections_active{pool="MyAppPool"} /
hikaricp_connections_max{pool="MyAppPool"} * 100
```

### 패널 2: Pending Threads - 장애 조기 경보

```yaml
# pending이 0이 아니면 즉시 확인 필요
hikaricp_connections_pending{pool="MyAppPool"}
```

### 패널 3 : 커넥션 획득 시간(p95, p99)

```yaml
# 커넥션 획득 시간 p95 (ms)
histogram_quantile(0.95,
  rate(hikaricp_connections_acquire_seconds_bucket{pool="MyAppPool"}[5m])
) * 1000

# p99
histogram_quantile(0.99,
  rate(hikaricp_connections_acquire_seconds_bucket{pool="MyAppPool"}[5m])
) * 1000
```

### 패널4 : 타임아웃 발생 추이

```yaml
# 분당 타임아웃 발생 수
rate(hikaricp_connections_timeout_total{pool="MyAppPool"}[5m]) * 60
```

**알림 규칙 (Grafana Alerting)**

```yaml
# 커넥션 풀 사용률 80% 초과 - Warning
- alert: HikariPoolHighUsage
  expr: hikaricp_connections_active / hikaricp_connections_max > 0.8
  for: 2m
  labels:
    severity: warning
  annotations:
    summary: "커넥션 풀 사용률 80% 초과"
    description: "{{ $labels.pool }} 풀의 사용률이 {{ $value | humanizePercentage }}입니다."

# Pending 스레드 발생 - Critical
- alert: HikariPoolPendingThreads
  expr: hikaricp_connections_pending > 0
  for: 30s
  labels:
    severity: critical
  annotations:
    summary: "커넥션 풀 대기 스레드 발생"
    description: "{{ $labels.pool }} 풀에 {{ $value }}개 스레드가 커넥션 대기 중입니다."

# 타임아웃 발생 - Critical
- alert: HikariPoolTimeout
  expr: rate(hikaricp_connections_timeout_total[5m]) > 0
  for: 0s
  labels:
    severity: critical
  annotations:
    summary: "커넥션 풀 타임아웃 발생"
```

## 1.5 커넥션 누수 탐지

### leak-detection-threashold 설정 시 로그 예시

```yaml
WARN  com.zaxxer.hikari.pool.ProxyLeakTask -
Connection leak detection triggered for com.mysql.cj.jdbc.ConnectionImpl@3a4b5c6d
on thread http-nio-8080-exec-7,
stack trace follows:
java.lang.Exception: Apparent connection leak detected
    at com.zaxxer.hikari.HikariDataSource.getConnection(HikariDataSource.java:128)
    at com.example.service.OrderService.createOrder(OrderService.java:45)
    at com.example.controller.OrderController.create(OrderController.java:30)
```

### 누수의 대표적 원인과 해결

```java
// ❌ 누수 발생 코드 - try-with-resources 미사용
public void badMethod() {
    Connection conn = dataSource.getConnection();
    PreparedStatement ps = conn.prepareStatement("SELECT ...");
    ResultSet rs = ps.executeQuery();
    // 예외 발생 시 conn.close()가 호출되지 않음!
    conn.close();
}

// ✅ 올바른 코드 - try-with-resources
public void goodMethod() {
    try (Connection conn = dataSource.getConnection();
         PreparedStatement ps = conn.prepareStatement("SELECT ...");
         ResultSet rs = ps.executeQuery()) {
        // 예외가 발생해도 자동으로 close됨
    }
}

// ✅ Spring @Transactional 사용 시에는 프레임워크가 관리
@Transactional
public void springMethod() {
    // Spring이 커넥션 획득/반환을 자동 관리
    repository.findById(id);
}
```

### 커스텀 누수 모니터링 코드

```java
@Component
@Slf4j
public class ConnectionPoolMonitor {

    private final HikariDataSource dataSource;

    @Scheduled(fixedRate = 10000) // 10초마다
    public void monitor() {
        HikariPoolMXBean poolMXBean = dataSource.getHikariPoolMXBean();

        int active = poolMXBean.getActiveConnections();
        int idle = poolMXBean.getIdleConnections();
        int total = poolMXBean.getTotalConnections();
        int waiting = poolMXBean.getThreadsAwaitingConnection();

        log.info("[HikariCP] active={}, idle={}, total={}, waiting={}",
                active, idle, total, waiting);

        // 경고 조건
        if (waiting > 0) {
            log.warn("[HikariCP] ⚠️ {}개 스레드가 커넥션 대기 중!", waiting);
        }

        double usageRate = (double) active / dataSource.getMaximumPoolSize();
        if (usageRate > 0.8) {
            log.warn("[HikariCP] ⚠️ 풀 사용률 {:.1f}% - 커넥션 부족 위험",
                    usageRate * 100);
        }
    }
}
```

# **2. DB Lock Wait, Deadlock, Slow Query 모니터링**

요약

`erformance_schema.data_lock_waits`와 `information_schema.innodb_trx`를 조인하면 "누가 누구를 블로킹하는지" 실시간으로 볼 수 있다. Deadlock 예방의 핵심은 모든 서비스에서 테이블/행 접근 순서를 통일하는 것다.

## 2.1 Lock의 종류

```java
Lock 계층 구조:
├── Table-level Lock
│   ├── Intention Shared Lock (IS)    - 행 공유 잠금 의도
│   └── Intention Exclusive Lock (IX) - 행 배타 잠금 의도
├── Row-level Lock
│   ├── Record Lock (S/X)             - 특정 인덱스 레코드 잠금
│   ├── Gap Lock                      - 인덱스 레코드 사이 간격 잠금
│   └── Next-Key Lock                 - Record + Gap (기본 잠금 모드)
└── Auto-increment Lock               - AUTO_INCREMENT 값 할당 시
```

### Lock Wait이 발생하는 시나리오

```java
-- 트랜잭션 A (먼저 실행)
BEGIN;
UPDATE orders SET status = 'SHIPPED' WHERE order_id = 100;
-- order_id=100에 X Lock 획득, 아직 COMMIT하지 않음

-- 트랜잭션 B (이후 실행)
BEGIN;
UPDATE orders SET status = 'DELIVERED' WHERE order_id = 100;
-- ⚠️⚠️⚠️ X Lock 대기 (Lock Wait) ⚠️⚠️⚠️
-- innodb_lock_wait_timeout (기본 50초) 초과 시:
-- ERROR 1205 (HY000): Lock wait timeout exceeded; try restarting transaction
```

## Lock Wait 모니터링

### 실시간 Lock 대기 확인 쿼리

```sql
-- ★ 현재 Lock을 기다리는 트랜잭션 확인 (MySQL 8.0+)
SELECT
    r.trx_id AS waiting_trx_id,
    r.trx_mysql_thread_id AS waiting_thread,
    r.trx_query AS waiting_query,
    r.trx_wait_started AS wait_started,
    TIMESTAMPDIFF(SECOND, r.trx_wait_started, NOW()) AS wait_seconds,
    b.trx_id AS blocking_trx_id,
    b.trx_mysql_thread_id AS blocking_thread,
    b.trx_query AS blocking_query,
    b.trx_started AS blocking_started
FROM information_schema.innodb_trx r
JOIN performance_schema.data_lock_waits w
    ON r.trx_id = w.REQUESTING_ENGINE_TRANSACTION_ID
JOIN information_schema.innodb_trx b
    ON b.trx_id = w.BLOCKING_ENGINE_TRANSACTION_ID;
```

### Performance Schema 기반 Lock 모니터링

```sql
-- 현재 걸려있는 모든 Data Lock 확인
SELECT
    OBJECT_SCHEMA,
    OBJECT_NAME,
    LOCK_TYPE,
    LOCK_MODE,
    LOCK_STATUS,
    LOCK_DATA,
    OWNER_THREAD_ID
FROM performance_schema.data_locks
ORDER BY OBJECT_SCHEMA, OBJECT_NAME;
```

### Lock Wait 모니터링 자동화 스크립트

```sql
#!/bin/bash
# lock_monitor.sh - 1분마다 실행 (crontab: * * * * * /path/to/lock_monitor.sh)

THRESHOLD_SECONDS=10
MYSQL_CMD="mysql -u monitor -p'password' -h 127.0.0.1"

RESULT=$($MYSQL_CMD -e "
SELECT COUNT(*) as cnt
FROM information_schema.innodb_trx r
JOIN performance_schema.data_lock_waits w
    ON r.trx_id = w.REQUESTING_ENGINE_TRANSACTION_ID
WHERE TIMESTAMPDIFF(SECOND, r.trx_wait_started, NOW()) > ${THRESHOLD_SECONDS};
" -s -N)

if [ "$RESULT" -gt 0 ]; then
    # 상세 정보를 Slack/PagerDuty로 전송
    DETAILS=$($MYSQL_CMD -e "
    SELECT r.trx_mysql_thread_id, r.trx_query,
           TIMESTAMPDIFF(SECOND, r.trx_wait_started, NOW()) as wait_sec,
           b.trx_mysql_thread_id as blocker
    FROM information_schema.innodb_trx r
    JOIN performance_schema.data_lock_waits w
        ON r.trx_id = w.REQUESTING_ENGINE_TRANSACTION_ID
    JOIN information_schema.innodb_trx b
        ON b.trx_id = w.BLOCKING_ENGINE_TRANSACTION_ID
    WHERE TIMESTAMPDIFF(SECOND, r.trx_wait_started, NOW()) > ${THRESHOLD_SECONDS};")

    curl -X POST "$SLACK_WEBHOOK" \
        -H 'Content-type: application/json' \
        -d "{\"text\": \"🔒 Lock Wait Alert!\\n${DETAILS}\"}"
fi
```

## 2.3 DeadLock 모니터링

### DeadLock

두 트랜잭션이 서로가 가진 Lock을 기다리는 교착 상태. InnoDB는 자동으로 감지, 한쪽을 롤백한다.

```sql
트랜잭션 A                          트랜잭션 B
─────────                         ─────────
BEGIN;                            BEGIN;
UPDATE accounts SET balance=100   UPDATE orders SET total=500
WHERE id=1; (Lock 획득)            WHERE id=200; (Lock 획득)
    │                                  │
    ▼                                  ▼
UPDATE orders SET total=600      UPDATE accounts SET balance=200
WHERE id=200; (⏳ B의 Lock 대기)   WHERE id=1; (⏳ A의 Lock 대기)
    │                                  │
    └──────── 💥 DEADLOCK ─────────────┘

→ InnoDB가 비용이 낮은 트랜잭션을 선택해 롤백
→ ERROR 1213 (40001): Deadlock found when trying to get lock
```

### **마지막 Deadlock 정보 확인**

```sql
SHOW ENGINE INNODB STATUS\G
```

```sql
------------------------
LATEST DETECTED DEADLOCK
------------------------
2024-01-15 14:35:22 0x7f8b4c0a1700
*** (1) TRANSACTION:
TRANSACTION 28401, ACTIVE 2 sec starting index read
mysql tables in use 1, locked 1
LOCK WAIT 3 lock struct(s), heap size 1136, 2 row lock(s)
MySQL thread id 145, OS thread handle 140234571234048, query id 89123
UPDATE accounts SET balance = 200 WHERE id = 1

*** (1) HOLDS THE LOCK(S):    ← 이 트랜잭션이 가진 Lock
RECORD LOCKS space id 50 page no 3 n bits 72 index PRIMARY of table `mydb`.`orders`
lock_mode X locks rec but not gap

*** (1) WAITING FOR THIS LOCK TO BE GRANTED:    ← 이 Lock을 기다리는 중
RECORD LOCKS space id 48 page no 4 n bits 80 index PRIMARY of table `mydb`.`accounts`
lock_mode X locks rec but not gap

*** (2) TRANSACTION:
TRANSACTION 28400, ACTIVE 3 sec starting index read
*** (2) HOLDS THE LOCK(S):
RECORD LOCKS space id 48 page no 4 ... table `mydb`.`accounts`
*** (2) WAITING FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 50 page no 3 ... table `mydb`.`orders`

*** WE ROLL BACK TRANSACTION (1)    ← 트랜잭션 1이 롤백됨
```

### DeadLock 로그를 파일로 기록

```sql
# my.cnf
[mysqld]
innodb_print_all_deadlocks = ON    # 모든 Deadlock을 에러 로그에 기록
log_error = /var/log/mysql/error.log
```

### DeadLock 예방 전략 (코드 레벨)

- 전략 1: 테이블/행 접근 순서를 항상 동일하게

  ServiceA: accounts → orders<br>ServiceB: orders → accounts  ← Deadlock 가능!

  ```java
  // ✅ 좋은 예: 팀 전체 컨벤션으로 접근 순서 통일
  @Transactional
  public void transfer(Long fromId, Long toId, BigDecimal amount) {
      // ID 순으로 정렬하여 항상 작은 ID부터 Lock 획득
      Long firstId = Math.min(fromId, toId);
      Long secondId = Math.max(fromId, toId);

      Account first = accountRepository.findByIdForUpdate(firstId);   // SELECT ... FOR UPDATE
      Account second = accountRepository.findByIdForUpdate(secondId);
      // 이제 Deadlock 불가능
  }
  ```

- 전략 2: 트랜잭션을 최대한 짧게

  ```java
  @Transactional
  public void processOrder(Long orderId) {
      // ❌ 트랜잭션 안에서 외부 API 호출 금지!
      // Order order = orderRepository.findById(orderId);
      // externalPaymentApi.charge(order);  ← 네트워크 지연 동안 Lock 유지
      // order.setStatus("PAID");

      // ✅ 외부 호출은 트랜잭션 밖에서
      PaymentResult result = externalPaymentApi.charge(orderId);  // 트랜잭션 밖

      updateOrderStatus(orderId, result);  // 짧은 트랜잭션
  }
  ```

- 전략 3: Deadlock 발생 시 재시도 로직

  ```java
  @Retryable(
      value = DeadlockLoserDataAccessException.class,
      maxAttempts = 3,
      backoff = @Backoff(delay = 100, multiplier = 2)
  )
  @Transactional
  public void updateWithRetry(Long id, String status) {
      orderRepository.updateStatus(id, status);
  }
  ```

## 2.4 Slow Query 모니터링

### 설정

```java
# my.cnf
[mysqld]
slow_query_log = ON
slow_query_log_file = /var/log/mysql/slow.log
long_query_time = 1              # 1초 이상 걸리는 쿼리 기록
log_queries_not_using_indexes = ON  # 인덱스 미사용 쿼리도 기록
min_examined_row_count = 1000    # 1000행 이상 스캔한 쿼리만 (노이즈 감소)
log_slow_extra = ON              # MySQL 8.0.14+ : 추가 정보 기록
```

### 로그 분석

```java
# mysqldumpslow로 패턴별 집계
mysqldumpslow -s t -t 10 /var/log/mysql/slow.log

# 출력 예시:
# Count: 127  Time=3.42s (434s)  Lock=0.00s (0s)  Rows=1.0 (127)
#   SELECT * FROM orders WHERE user_id = N AND status = 'S'
#
# → 이 패턴의 쿼리가 127번 실행, 평균 3.42초, 총 434초 소요
# → user_id + status 복합 인덱스 추가 검토

# pt-query-digest (Percona Toolkit) - 더 상세한 분석
pt-query-digest /var/log/mysql/slow.log --limit 20
```

### **Spring Boot에서 Slow Query 감지**

```java
// 방법 1: HikariCP의 connectionTestQuery 활용은 불가 → P6Spy 사용

// datasource-proxy를 이용한 쿼리 시간 측정
@Configuration
public class DataSourceConfig {

    @Bean
    public DataSource dataSource(DataSourceProperties properties) {
        HikariDataSource original = properties.initializeDataSourceBuilder()
                .type(HikariDataSource.class)
                .build();

        return ProxyDataSourceBuilder.create(original)
                .name("slow-query-detector")
                .listener(new SlowQueryListener() {
                    {
                        setThreshold(1000); // 1초
                        setThresholdTimeUnit(TimeUnit.MILLISECONDS);
                    }

                    @Override
                    protected void onSlowQuery(ExecutionInfo execInfo,
                                                List<QueryInfo> queries,
                                                long elapsedTime) {
                        String query = queries.stream()
                                .map(QueryInfo::getQuery)
                                .collect(Collectors.joining("\n"));

                        log.warn("Slow Query ({}ms): {}", elapsedTime, query);

                        // 메트릭 수집
                        meterRegistry.counter("db.slow_query",
                                "query_hash", String.valueOf(query.hashCode()))
                                .increment();
                    }
                })
                .build();
    }
}
```

### EXPLAIN 분석 실전

```sql
- 느린 쿼리 발견 시 EXPLAIN으로 분석
EXPLAIN ANALYZESELECT o., u.name
FROM orders o
JOIN users u ON o.user_id = u.id
WHERE o.status = 'PENDING'AND o.created_at > '2024-01-01'ORDER BY o.created_at DESCLIMIT 20;
```

# **3. WhaTap APM을 통한 트레이싱**

요약

주문 API가 200ms → 5초로 느려진 시나리오를 Step별로 추적하는 과정. 프로파일에서 `UPDATE inventory` 쿼리가 4,950ms Lock Wait을 일으킨 것을 발견하고, 근본 원인(두 서비스가 같은 테이블 직접 접근)과 단기/중기/장기 해결책(Redis 재고 선점 코드 포함)까지 정리.

### 설치

```sql
# 1. WhaTap Java Agent 다운로드
wget https://repo.whatap.io/agent/whatap.agent-2.x.x.jar

# 2. whatap.conf 설정
cat > whatap.conf << EOF
license=발급받은-라이선스키
whatap.server.host=13.124.11.223/13.209.172.35
app_name=order-service
app_process_name=java

# 트레이싱 상세 설정
profile_step_normal_count=2000     # 프로파일 스텝 최대 수
profile_http_header_enabled=true    # HTTP 헤더 수집
profile_http_parameter_enabled=true # HTTP 파라미터 수집 (개인정보 주의)
profile_sql_param_enabled=true      # SQL 바인드 파라미터 수집
profile_method_enabled=true         # 메서드 프로파일링

# Slow 기준값
trace_active_transaction_slow_time=5000  # 5초 이상 = Slow
trace_active_transaction_very_slow_time=10000  # 10초 이상 = Very Slow

# HikariCP 모니터링
profile_connection_open_enabled=true
dbcp_pool_enabled=true
EOF

# 3. JVM 옵션에 Agent 추가
java -javaagent:/path/to/whatap.agent-2.x.x.jar \
     -Dwhatap.config.path=/path/to/whatap.conf \
     -jar my-application.jar
```

### WhaTap 트레이싱 개념

```sql
하나의 HTTP 요청 (Transaction) -- 트랜잭션 프로파일 구조
│
├── [STEP 1] HTTP Start ─────────────── 0ms
│   ├── URL: POST /api/v1/orders
│   ├── Client IP: 10.xx.xx.xx
│   └── Header: Content-Type: application/json
│
├── [STEP 2] Method ─────────────────── 2ms
│   └── OrderController.createOrder()
│
├── [STEP 3] Method ─────────────────── 3ms
│   └── OrderService.createOrder()
│
├── [STEP 4] DB Connection ──────────── 5ms
│   ├── Pool: HikariPool-1
│   ├── Connection 획득 시간: 2ms
│   └── Active: 8 / Max: 20
│
├── [STEP 5] SQL ────────────────────── 7ms
│   ├── SELECT * FROM users WHERE id = ?
│   ├── Param: [12345]
│   ├── Rows: 1
│   └── Elapsed: 3ms ✅
│
├── [STEP 6] SQL ────────────────────── 12ms
│   ├── SELECT * FROM products WHERE id IN (?, ?, ?)
│   ├── Param: [101, 102, 103]
│   ├── Rows: 3
│   └── Elapsed: 5ms ✅
│
├── [STEP 7] SQL ────────────────────── 18ms
│   ├── INSERT INTO orders (user_id, total, status) VALUES (?, ?, ?)
│   ├── Param: [12345, 150000, 'PENDING']
│   └── Elapsed: 8ms ✅
│
├── [STEP 8] HTTP Call ──────────────── 30ms      ★ 외부 호출
│   ├── POST https://payment-api.example.com/charge
│   ├── Status: 200
│   └── Elapsed: 2800ms 🔴 ← 여기서 느림!
│
├── [STEP 9] SQL ────────────────────── 2835ms
│   ├── UPDATE orders SET status='PAID', payment_id=? WHERE id=?
│   ├── Param: ['pay_abc123', 67890]
│   └── Elapsed: 12ms ✅
│
├── [STEP 10] SQL ───────────────────── 2850ms
│   ├── INSERT INTO order_items (order_id, product_id, qty) VALUES ...
│   └── Elapsed: 15ms ✅
│
└── [STEP 11] HTTP End ──────────────── 2870ms
    ├── Status: 201 Created
    └── Total Elapsed: 2870ms 🔴

→ 전체 2870ms 중 2800ms가 외부 결제 API 호출
→ 병목 지점: STEP 8 (외부 HTTP 호출)
```

## **예시: 주문 생성 API가 갑자기 느려진 경우**

상황

```sql
서비스: 이커머스 주문 서비스
증상: POST /api/v1/orders 의 평균 응답시간이 200ms → 5초로 급증
시점: 2024-01-15 14:30 ~ 15:00
영향: 주문 성공률 92% → 45% 하락
```

1. 히트맵에서 이상 패턴 식별
2. 트랜잭션 프로파일 상세 분석

   느린 트랜잭션 클릭 → 프로파일 상세를 확인

3. 멀티 트랜잭션 연계 분석

   ```sql
   [order-service]  POST /api/v1/orders ── 5,240ms 🔴
       │
       ├── [DB] MySQL:3306
       │   ├── SELECT users      3ms  ✅
       │   ├── SELECT products   8ms  ✅
       │   ├── INSERT orders    12ms  ✅
       │   └── UPDATE inventory  4,950ms  🔴  ← Lock Wait
       │
       ├── [inventory-service]  PUT /api/internal/stock ── timeout ❌
       │   │   (order-service가 DB 직접 접근과 서비스 호출 두 경로 존재)
       │   └── [DB] MySQL:3307
       │       └── UPDATE inventory  — 이 쿼리가 Lock을 잡고 있음
       │
       └── [notification-service]  POST /api/notify ── not reached
           └── (주문 실패로 호출되지 않음)

   ```

4. 해결방안 시나리오

   [즉시 조치]

   1. SELECT ... FOR UPDATE NOWAIT 사용

      ⇒ Lock 즉시 실패로 빠른 재시도 가능

   2. connection-timeout을 3초로 축소

      ⇒ 풀 고갈 시 빠르게 실패

   [구조 개선] - 중기

   1. inventory 접근을 단일화

      ⇒ 분산 Lock 문제 제거

   2. 재고 차감을 별도 트랜잭션으로 분리

      ⇒ 주문 트랜잭션의 Lock 유지 최소화

   [아키텍처 개선] - 장기

   1. Redis를 이용한 재고 선점 방식

      ⇒ DECR 명령으로 atomic 차감, DB 부하 제거

   2. 이벤트 기반 비동기 처리

      ⇒ 주문 접수 후 재고 차감은 이벤트로 동기 처리
{% endraw %}
