---
title: "DB + Server Timezone"
date: 2026-08-09 18:20:00 +0900
categories: [database, timezone]
tags: [timezone, mysql, java]
---

> 실제 서비스명, 도메인, 리소스 식별자 등은 보안을 위해 예시 값으로 바꿨습니다.
{: .prompt-info }

버그 : 벽시계의 출처가 3개(JVM 기본존, DB세션 존, 프론트)라서 같은 순간에 대해 다른 숫자가 생성되는 중. ⇒ 출처를 하나로 줄이기(UTC를 쓰라는 조언)

#### 서비스 A 코드의 현재 상태 (실제 확인)

| 위치 | 현재 | 문제 |
|---|---|---|
| `ClockConfig.java` | `Clock.systemDefaultZone()` | JVM 기본 존에 종속. 내 Mac=KST, ECS 컨테이너=UTC |
| `BaseTimeEntity` | `LocalDateTime` created/updated | 존 정보 없음. 어느 존의 벽시계인지 코드에 안 적혀 있음 |
| `docker-compose.yml` | MySQL `TZ: Asia/Seoul` | JVM(UTC 예정)과 불일치 |
| `application.yml` datasource url | `jdbc:mysql://.../appdb` (파라미터 0개) | `connectionTimeZone` 미지정 → 드라이버 기본 동작에 의존 |
| `{Notice, Posting, CardNews}` | `LocalDateTime.now()` 직접 호출 | Clock 주입 규칙을 깨는 3곳. 테스트 불가 + 존 종속 |
| `Session.java` | `isEntryWindowOpen(now, minutesBefore)` | **Zoom 입장 링크 노출 게이트.** 멘토님이 말한 "특정 시간 이후 공개" 로직 그 자체 |

Instant를 쓰는 파일, LocalDateTime을 쓰는 파일이 섞여 있음(각각 33개, 59개)

이벤트는 Instant.now(), 엔티티는 LocalDateTime → 같은 사실을 두 표현으로 저장 중

⇒ 결론 : ECS에서는 9시간 늦게 열린다.

ECS로 옮기는 순간, 신청 학생에게 Zoom링크가 9시간 늦게 열린다. 반대로 마이그레이션으로 들여온 기존 데이터가 UTC 기준이라면 같은 코드이니 9시간 일찍 줌 링크가 열린다. (지역에 따라 조기 노출)

### 설계

1. **런타임 시계의 출처를 UTC 하나로 고정한다. ⇒ 네군데 맞춰야 의미가 있다.**

```java
// ClockConfig.java
@Bean
public Clock clock() {
    return Clock.systemUTC();   // 벽시계의 유일한 출처
}
```

```yaml
# application.yml
spring:
  datasource:
    url: jdbc:mysql://${DB_HOST}:3306/appdb?connectionTimeZone=UTC&preserveInstants=true&characterEncoding=UTF-8
  jackson:
    time-zone: UTC
```

```docker
ENV TZ=UTC
ENV JAVA_TOOL_OPTIONS="-Duser.timezone=UTC ..."
```

```yaml
# docker-compose.yml — 로컬도 운영과 같은 조건으로. 지금 Asia/Seoul이라 로컬에서만 정상으로 보임
mysql:
  environment:
    TZ: UTC
```

1. LocalDateTime.now(), Instant.now() 바로 호출 금지. Clock 주입만.

```yaml
# .githooks/pre-commit
if git diff --cached -U0 -- '*.java' | grep -nE '^\+.*(LocalDateTime|LocalDate|Instant)\.now\(\)[^,]'; then
  echo "❌ 시간은 Clock 빈 경유로만 얻습니다: LocalDateTime.now(clock)"
  exit 1
fi
```

1. 기존 데이터 보정은 Flyway

`CONVERT_TZ`는 MySQL 타임존 테이블이 적재돼 있어야 동작하므로, Aurora에 적재 안 돼 있으면 `DATE_SUB(col, INTERVAL 9 HOUR)`로 간다.
