---
title: "Connection Pool & Thread Pool"
date: 2026-01-06 00:29:00 +0900
categories: [spring, async-thread]
tags: [connection-pool, thread-pool, hikaricp]
---

## 스레드 풀과 DB 커넥션 풀

멀티 스레드 환경에서 스레드풀과 DB 커넥션 풀은 애플리케이션의 성능과 직결되는 핵심 요소이다.

`스레드 풀`은 작업 처리를 위해 스레드를 효율적으로 관리하는 방법이며, `DB 커넥션 풀`은 데이터베이스 연결을 재사용함으로써 연결 생성과 소멸에 드는 비용을 줄이는 기술이다. 이 두 풀은 애플리케이션의 성능에 직접적인 영향을 미치고, 적절히 관리하지 않으면 애플리케이션의 성능 저하, 다양한 문제가 발생할 수 있다.

## 스레드 풀과 DB 커넥션 풀의 상관관계

스레드 풀과 DB 커넥션 풀 사이에는 밀접한 상관관계가 있다.

1. 스레드 풀의 크기 > DB 커넥션 풀의 크기

   스레드 풀의 크기가 DB 커넥션 풀의 크기보다 크면 동시에 많은 쓰레드가 DB 커넥션을 요구하게 되지만, 사용가능한 커넥션이 제한되어있기 때문에 병목현상이 발생할 가능성이 있다.

2. DB 커넥션 풀의 크기 > 스레드 풀의 크기

   반대로 DB 커넥션 풀의 크기가 스레드 풀의 크기보다 크면 많은 DB 커넥션이 유휴 상태로 남게 되어 자원이 낭비될 수 있다. 왜나하면 동시에 처리할 수 있는 스레드의 수가 제한되어 있기 때문이다.

따라서 스레드 풀과 DB 커넥션 풀의 크기를 적절히 조절하는 것이 중요하다. 이를 통해 자원의 효율적인 사용과 애플리케이션의 성능 최적화를 달성할 수 있다.

## 스레드 풀 최적화 전략

스레드 풀의 최적화는 애플리케이션의 성능에 직접적인 영향을 미친다. 스레드 풀의 크기를 너무 작게 설정하면 스레드가 부족해 작업 처리가 지연될 수 있으며, 너무 크게 설정하면 자원 낭비와 `Context Switch` 비용이 증가할 수 있다.

스레드 풀의 최적화를 위해서 애플리케이션의 작업 특성을 분석하고, 스레드 풀의 사용 패턴을 모니터링 하는 것이 필요하다. → 이를 통해 스레드 풀의 크기를 동적으로 조절하거나, 스레드 풀의 구성을 최적화 할 수 있다.

## DB 커넥션 풀 최적화 전략

DB 커녁션 풀의 최적화 역시 애플리케이션의 성능에 중요한 영향을 미친다. DB 커넥션 풀의 크기가 너무 작으면 DB 연결 요청이 대기 상태에 빠질 수 있으며, 너무 크면 불필요한 자원 낭비가 발생할 수 있다.

DB 커넥션 풀의 최적화를 위해 DB 접근 패턴을 분석하고, 커넥션 풀의 사용 상태를 모니터링 하는 것이 필요하다. → 이를 통해 커넥션 풀의 크기를 동적으로 조절하거나, 커넥션 풀의 구성을 최적화 할 수 있다.

## **Connection Pool과 적절한 Size 설정하기 (with `HikariCP`)**

### Connection

유저가 보낸 요청을 처리하는 과정에서 서버(WAS)가 DB에 접근하기 위해서는 Connection 타입의 DB 연결을 위한 객체가 필요하다.

**Driver Manager**

JDBC는 DriverManager를 사용하면 Connection을 아래와 같이 가져올 수 있다.

```java
Connection connection = DriverManager.getConnection(DB_URL, User, Password);
```

**Connection은 생성 비용이 크다.**

하지만 이런 식으로 DB에 접근할 때마다 Connection을 생성하는 것은 비효율적이다. Connection은 생성하는데 비용이 많이 들기 때문이다.

```java
// 행을 insert하는 데 드는 시간 (괄호 안의 숫자는 비율을 나타낸다)
// 연결 비용은 3, 연결 종료 비용은 1로 꽤 큰 비중을 차지하는 것을 알 수 있다.

Connecting: (3)
Sending query to server: (2)
Parsing query: (2)
Inserting row: (1 × size of row)
Inserting indexes: (1 × number of indexes)
Closing: (1)
```

**커넥션 생성이 비용이 많이 드는 이유**

보통 DB와 연결할 때 TCP 통신을 하기 때문이다. TCP 통신은 커넥션의 안정성과 신뢰성 보장을 위해 연결 시 3-Way HandShake, 연결 종료 시 4-Way HandShake 과정을 거친다. 이 과정에서 통신 비용이 많이 소모되므로 커넥션을 계속 생성하는데에 비용이 많이 드는 것이다.

→ 그래서 Connection Pool이 필요하다. Connection을 생성하기 위한 비용을 줄이기 위해 Connection Pool을 사용한다. Connection을 미리 만들어 Pool에 보관하고, 필요할 때마다 커넥션을 꺼내 사용하면 커넥션을 생성하는데 드는 비용을 줄일 수 있다.

**DataSource**

DataSource는 DB나 파일과 같은 물리적 데이터 소스에 연결할 때 사용하는 인터페이스이다.

DataSource를 사용하면 Connection Pool을 활용할 수 있다. `DriverManager`가 아닌 `DataSource`를 사용하는 이유도 이 때문이다. 뿐만 아니라 애플리케이션 코드를 직접 수정하지 않고 properties로 DB 연결을 변경할 수 있고, 트랜잭션도 사용이 가능하다.

```java
JdbcDataSource dataSource = new JdbcDataSource();
dataSource.setURL(H2_URL);

// Connection을 새로 생성하지 않고, 이미 만들어진 Connection을 get으로 가져온다.
Connection connection = dataSource.getConnection(USER, PASSWORD);
```

### HikariCP

Spring Boot 2.0부터 HikariCP를 기본 DataSource로 채택하고 있다. HikariCP는 빠르고 간편하며 오버헤드가 없다.

**HikariCP Property**

- maximunPoolSize
  - Connection Pool이 가질 수 있는 최대 커넥션 개수.
  - Default : 10개
  - 유휴한 커넥션과 사용중인 커넥션을 모두 포함한 개수이다.
- connectionTimeout
  - Client가 Pool에서 Connection을 기다리는 최대 시간. 초과되면 SQLException이 발생한다.
  - Default : 30000ms
  - 최소 : 250ms
- maxLifetime
- keepaliveTime

[https://colour-my-memories-blue.tistory.com/15](https://colour-my-memories-blue.tistory.com/15)
