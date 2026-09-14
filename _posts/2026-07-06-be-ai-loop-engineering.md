---
title: "루프 엔지니어링"
date: 2026-07-06 17:54:00 +0900
categories: [ai, ai-engineering]
tags: [loop-engineering, ai-agent]
---

> 실제 서비스명, 도메인, 리소스 식별자 등은 보안을 위해 예시 값으로 바꿨습니다.
{: .prompt-info }

[루프 엔지니어링 — '프롬프트하는 나'를 시스템으로 대체하는 법](https://www.youtube.com/watch?v=A7gwGNsL6y4)

핵심 : 중첩

랠프 위검 루프 - 가장 단순한 원형

- 단일/수직
- 상태는 디스크에
- 새 컨텍스트 - 파일을 다시 읽어 다음 할 일을 스스로 고름

하네스 엔지니어링 - 에이전트를 둘러싼 공학 분야

- 파일로 저장해서 관리
- 여러 에이전트들이 유기적으로
- 0~100까지 어떤건 하고, 어떤건 하지 마라, 이건 해\~\~

루프 엔지니어링 - 가장 높은 추상화 (프롬프팅 조차도 AI에게)

- 나를 대체하는 시스템
- act → observe → decide → repeat
- 하네스보다 한층 위

### 프롬프트하는 나 자신을 시스템으로 대체함

맨 마지막에만 사람 검증을 한다면 그게 loop 엔지니어링

prompt → work → human → prompt → work → human ….

⇒

prompt → work →prompt → work →prompt → work →prompt → work…. ⇒ human

![](/assets/img/2026-07-06/be-ai-loop-engineering-1.png)

예) 경쟁사 a-b테스트 하는것 처럼 우리도 a-b테스트 할 수 있도록 해줘 : looping

프로젝트 - loop적용 시 효율이 안나올 수 있을 수 있다. (프로젝트 스코프가 다르다.)

사이드플젝/mvp/pmf에 적용하는 작은 프로젝트 - 루프 iterate할 만큼 어려운 작업이 아닐 수 있다.

⇒ max or ultracode

적절한 사용 예 : 엄청 큰 코드 내에서 feature를 적용할 때 (W사 예시), 컨텐츠 생산 시

![](/assets/img/2026-07-06/be-ai-loop-engineering-2.png)

### 루프 성립 2대 조건

- 트리거
  - 언제 작업할 것인가
- **검증** 가능한 목표
  - 루프의 완료 조건
    - 테스트가 완료될 때까지
    - Lighthouse 스코어가 90점 이상이 될 때까지
    - Critical 점수가 없을 때까지

### Cron ↔ Roop

크론잡 : 상태를 보지 않는다.

루프의 에이전트 : 상태를 본다

상태를 보고 에이전트를 본다 (5개나 fail했네, 5개 보자~) with Fable5**(오랜 시간의 오케스트레이터와 함께)**

- 기타

  openclaw, hermes agent

서비스 A를 Looping했으면 어땠을까

루프 → with fable5

### Outer Loop

작은 setup 하나로 개선을 한다 (좋으면 개선, 나쁘면 버림)

![](/assets/img/2026-07-06/be-ai-loop-engineering-3.png)

### 루프의 6개 빌딩 블록

![](/assets/img/2026-07-06/be-ai-loop-engineering-4.png)

프롬프트 예시

> 보안상 이미지를 생략했습니다.

Generator(코드 생성), Evaluator(검증)이 무조건 달라야 함

[https://github.com/jha0313/harness_framework](https://github.com/jha0313/harness_framework)

### 언제 루프 만들까?

![](/assets/img/2026-07-06/be-ai-loop-engineering-6.png)

반복된다 > lighthouse/test/build

**검증 만들기, 만약 채점표가 주관적이라면 - 검증부터 만들어야 함**

Opus 비용이 필요없을 때는 Sonnet/Haiku를 적용
