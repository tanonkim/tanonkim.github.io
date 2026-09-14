---
title: "Dockerizing"
date: 2026-07-11 11:45:00 +0900
categories: [infra, docker]
tags: [docker, dockerfile, spring-boot]
---

> 실제 서비스명, 도메인, 리소스 식별자 등은 보안을 위해 예시 값으로 바꿨습니다.
{: .prompt-info }

### 1. api(Spring Boot)를 멀티스테이지 Docker 이미지로 만든다.

**목표 3가지**

1. 작고 안전한 런타임 이미지 (빌드 도구는 최종 이미지에 안 남김)
2. 빌드 캐시 최적화 (의존성 레이어 분리 → 소스만 바뀌면 빠른 재빌드)
3. 시크릿 0 / 비루트 / graceful 종료 — 운영 안전

### 2. Dockerfile

```java
  ARG JAVA_VERSION=21                                    
  
  # ① 버전 ARG(기본값)

  ###### Build stage — JDK로 부트JAR 생성 ######
  FROM amazoncorretto:${JAVA_VERSION}-al2023-jdk AS build
  WORKDIR /workspace
  RUN dnf install -y findutils && dnf clean all          # gradlew가 xargs 요구

  # ② 레이어 캐싱: 변경 빈도 낮은 것부터 COPY
  COPY gradlew ./
  COPY gradle ./gradle
  RUN chmod +x ./gradlew
  COPY settings.gradle build.gradle ./
  COPY service/build.gradle service/
  COPY api/build.gradle     api/
  COPY batch/build.gradle   batch/
  RUN ./gradlew --no-daemon :api:dependencies            
  
  # ③ 의존성만 먼저 → 레이어 고정

  COPY . .                                               
  
  
  # ④ 소스는 마지막(자주 바뀜)
  RUN ./gradlew --no-daemon :api:bootJar -x test
  RUN cp "$(ls api/build/libs/*.jar | grep -v -- '-plain.jar' | head -n1)" /workspace/app.jar

  ###### Run stage — headless 런타임(JDK 없음) ######
  FROM amazoncorretto:${JAVA_VERSION}-al2023-headless AS run
  ENV TZ=Asia/Seoul
  ENV JAVA_OPTS="-XX:InitialRAMPercentage=75.0 -XX:MaxRAMPercentage=75.0 -XX:+UseG1GC
  -XX:+ExitOnOutOfMemoryError"
  RUN dnf install -y curl tzdata shadow-utils --allowerasing   # 헬스체크 curl + 계정생성 도구
  RUN groupadd --system --gid 1000 servicea && \                  
  
  
  # ⑤ 비루트 계정
      useradd --system --uid 1000 --gid servicea --home-dir /app --shell /sbin/nologin servicea
  WORKDIR /app
  COPY --from=build /workspace/app.jar /app/app.jar            
  
  
  # ⑥ 산출물(JAR)만 복사
  USER servicea
  EXPOSE 8080
  HEALTHCHECK --interval=30s --timeout=5s --start-period=60s --retries=3 \
      CMD curl -fsS http://localhost:8080/actuator/health/readiness || exit 1   # ⑦
  ENTRYPOINT ["sh", "-c", "exec java $JAVA_OPTS -jar /app/app.jar"]              # ⑧ exec → SIGTERM 전파
```

### 개념

1. ②③④ 레이어 캐싱

   Docker는 레이어 단위 캐시. 의존성을 소스보다 위에 두면 코드만 코쳤을 때 무거운 :dependencies를 재실행 안함 → 빌드 수 분 단축

2. 비루트(USER servicea)

   컨테이너 탈취 시 권한 최소화, 루트로 도는 컨테이너는 보안 사고 시 피해가 큼

3. 산출물만 목사

   -plain.jar(의존성이 없는 얇은 JAR) 제외하고 실행가능 부트 JAR만 app.jar로 정규화

4. HEALTHCHECK

   /actuator/health/readiness로 컨테이너 건강 판정. ECS/ALB가 이 신호로 라우팅 결정

5. exec java …

   shell이 아니라 java가 PID 1이 됨. → SIGTERM이 JVM에 직접 전달 : server.shutdown=graceful 동작

### 체크리스트

- 멀티스테이지: build(JDK) → run(headless) 분리
- 시크릿/환경변수는 이미지에 안 넣음: **.dockerignore**가 **.env**·프로파일 yml 차단. 운영값은 ECS Task Definition의 secrets(Secrets Manager)로 주입 (이미지엔 하드코딩 0)
- ${JAVA_VERSION} ARG 기본값(21) 확인
- 비루트·헬스체크·graceful 포함

### 검증

- 빌드 컨텍스트 = 레포 루트 (루트 gradlew·settings.gradle 필요)

  `docker build -f api/Dockerfile -t my-api:local .`
