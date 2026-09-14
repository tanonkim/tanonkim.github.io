---
title: "HikariCP의 커넥션 maxLifetime과 MySQL wait timeout"
date: 2026-01-16 20:23:00 +0900
categories: [database, connection-pool]
tags: [hikaricp, mysql]
---

> 실제 서비스명, 도메인, 리소스 식별자 등은 보안을 위해 예시 값으로 바꿨습니다.
{: .prompt-info }

[HikariCP의 커넥션 maxLifetime과 MySQL wait timeout간의 관계 알아보기!](https://seongonion.tistory.com/157#MySQL%EC%9D%98%20wait%20timeout-1)

트랜잭션 타임아웃

### HikariCP의 connectinon max lifetime

HikariCp는 커넥션 풀링을 위해 사용자가 지정한 개수만큼 커넥션을 DB 서버와 미리 만들어 놓고, 커넥션이 필요한 상황마다 커넥션 풀에서 Idle(유휴상태)인 커넥션을 쓸 수 있도록 한다.

**maxLifetime**

커넥션이 커넥션 풀에서 살아있을 수 있는 최대 시간

`HikariCP`는 사용자가 설정한 커넥션 최대 생명주기 즉, `maxLifetime`만큼만 커넥션을 살려두고 해당 시간이 지난 커넥션은 제거(retire)시킨 후 새로운 커넥션을 맺는다.

→ 이런 동작 방식은 데이터베이스 서버의 최신 변경사항을 반영하고, 커넥션 유지 동안 발생할 수 있는 리소스의 누수를 막기 위한 선택이다.

```java
[REPLICA] DataSource Configuration
 ========================================
   JDBC URL        : jdbc:mysql://<db-endpoint>:3306/appdb?rewriteBatchedStatements=true&useUnicode=true&characterEncoding=utf8&validationQuery="select 1"
   Username        : app_user
   Driver          : com.mysql.cj.jdbc.Driver
   Max Pool Size   : 5
   Min Idle        : 2
   Idle Timeout    : 300000 ms
   Max Lifetime    : 1200000 ms
   Conn Timeout    : 5000 ms
   Leak Detection  : 30000 ms
```

### MySQL의 wait timeout

MySQL 서버는 애플리케이션 서버를 포함하여 여러 종류의 클라이언트들과 커넥션을 맺고 소통한다. 하지만 연결되는 클라이언트들이 많을수록 커넥션을 무한정하게 연결해두는 것은 리소스 낭비가 될 수 있다.

이런 리소스 효율화를 위해 MySQL은 wait timeout값을 사용한다. 클라이언트가 데이터베이스와 연결된 후 정해둔 wait timeout시간동안 아무런 작업을 하지 않는다면 MySQL 서버는 해당 커넥션을 닫아 가능성이 있는 리소스를 회수한다.

### MySQL과 HikariCP의 관계

데이터베이스 클라이언트의 입장에선 데이터베이스 상에 설정된 여러 값들을 고려하여 커넥션 풀 기술을 사용해야한다.

`HikariCP`의 `maxLifetime`과 MySQL의 `wait timeout` 값 또한 밀접한 관계가 있으며, 이를 제대로 이해하지 못한 채 잘못된 값을 설정하면 커넥션 획득 및 관리에 실패할 가능성이 있다.

일반적으로 문제는 **maxLifeTime이 wait timeout보다 더 길 때** 발생한다.

- 가정
  1. **wait time이 30**초로 설정된 MySQL 서버로부터 커넥션을 받아 HikariCP가 커넥션 풀을 생성
  2. ConnectionPool의 maxLifeTime : 60초

  ![](/assets/img/2026-01-16/be-db-hikaricp-maxlifetime-wait-timeout-1.png)

  가정) Connection Pool은 3개,

  의미) ConnectionPool의 maxLifeTime은 60초, 60초 동안 가지고 있다가 바꿔달라고 요청

  커넥션을 맺은지 30초가 넘었지만, 서버가 커넥션을 하나도 사용하지 않았다. → 설정한 wait timeout시간동안 커넥션으로 한 번도 요청이 들어오지 않았으므로, MySQL은 커넥션을 수거한다.

  **31초 경과 시..**

  MySQL은 커넥션 생성시간을 기준으로 wait timeout시간이 경과한 커넥션을 모두 회수하였지만, HikariCP의 ConnectionPool은 죽어있는 커넥션을 들고 있다. (사용 전에는 유효성 체크를 하지 않음)

  **60초 이전에..**

  60초가 경과학디 전에 서버에서 요청이 들어와 커넥션을 사용할 일이 생겼다. ConnectionPool은 갖고 있던 커넥션을 하나 쓰라고 주지만 **커넥션은 죽어있다.**

  → 결국 서버는 응답을 정상적으로 내려주지 못하거나, ConnectionPool이 다시 새로운 커넥션을 생성해 요청을 처리한다.

요약 : HikariCP는 커넥션이 살아있다고 생각하지만 **MySQL은 이미 wait timeout으로 끊어버림**

결론 : `HikariCP`의 공식문서에서는 커넥션 풀의 `maxLifetime`값을 데이터베이스에 설정된 커넥션 관련 타임 설정보다 더 짧게 설정하도록 권고하고 있다.

### **wait timeout이 경과하기 전에 커넥션에 대해 지속적으로 헬스체크를 하면 되지 않나?**

ySQL서버에서 설정한 `wait timeout`이 경과하기 전에 SELECT 1과 같은 핑 쿼리를 지속적으로 날려 데이터베이스에서 커넥션을 회수 할 수 없도록 할수도 있다. → 이렇게 하지 않는다, 유휴 상태의 커넥션에 대한 테스팅은 불필요한 쿼리를 발생시키고 데이터베이스에 설정된 타임아웃 값을 무력화하며 네트워크 인프라 팀의 통제를 빼앗는다는 것이다.

![](/assets/img/2026-01-16/be-db-hikaricp-maxlifetime-wait-timeout-2.png)

### 궁금한 점 : HTTP에서의 Broken Pipe

*"HikariCP maxLifetime와 HTTP keep-alive는 같은 원리일까?"*

SpringBoot에서 broken pipe 에러가 발생했는데, mysql과 HikariCP 설정을 확인할 줄 알았으나 시니어가 keep alivetime을 확인했다.

| 계층 | 네 글 | keep-alive |
|---|---|---|
| 서버 | MySQL | Tomcat / Nginx |
| 클라이언트 | HikariCP | 브라우저 / API 서버 |
| 문제 | 죽은 DB 커넥션 재사용 | 죽은 HTTP 커넥션 재사용 |

- Broken Pipe

`Broken pipe`는 **서버가 클라이언트에게 데이터를 보내려는데, 이미 연결이 끊긴 상태**일 때 말한다.

즉, "보낼 상대가 이미 전화를 끊었는데, 계속 말하려고 한 상황" 같은 느낌

주요 원인:

- 클라이언트가 먼저 연결 종료
- 중간에 로드밸런서 / 프록시가 연결 종료
- 타임아웃으로 소켓이 닫힘

HTTP keep-alive : **하나의** **TCP 연결을 재사용**해서 여러 요청을 처리하는 기능

예)  이렇게 서로 다르면 누군가는 먼저 끊고, 누군가는 살아있다고 착각 → Broken pipe 발생

| 구간 | keep-alive |
|---|---|
| 클라이언트 | 30초 |
| 로드밸런서 | 60초 |
| SpringBoot(Tomcat) | 120초 |

- Spring Boot에서 체크할 것들

### ① Tomcat keep-alive 설정

```yaml
server:
	tomcat:
		keep-alive-timeout:60000# ms
		max-keep-alive-requests:100

server.
	tomcat.
			keep-alive-timeout = -1 # 기본값
```

### ② Nginx / ALB / LB 설정

```
keepalive_timeout60;
```

### ③ 클라이언트 설정

- HTTP client timeout
- 모바일 / 브라우저 / API 서버
