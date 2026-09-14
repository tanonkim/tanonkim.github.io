---
title: "Performance Patterns 7 : Third Parties"
date: 2025-06-01 15:09:00 +0900
categories: [design-pattern, performance]
tags: [design-pattern, third-party]
---

### 요약

서드파티 리소스로 인해 사이트 속도가 느려질 수 있고, 최적화가 어려울 수 있다. 아래 사례를 참고하여 다양한 유형의 서드파티를 효율적으로 로드하거나, 지연할 수 있다.

서드파티 스크립트를 로드하기 위해 언제, 어떻게 구성하는지에 대한 템플릿 컴포넌트 형태로 제공하는 Next.js Script Component와 같은 프레임워크 수준의 컴포넌트 요소를 사용할 수도 있다. Partytown과 같은 실험적인 아이디어를 살펴보는 것도 좋다.

### 서드파티 리소스

서드파티 없이 단독으로 사용되는 현대의 웹사이트를 찾는건 어렵다. 21년도 웹 연감 보고서에 따르면 94%이상이 3P를 사용중이며, 그 중에 이미지와 자바스크립트는 3P 컨텐츠 중에서 가장 많은 역할을 하고 있다.

사이트에는 아래와 같은 3P 리소스들이 사용된다.

- 지도, 비디오, SNS, 채팅 서비스용 임베드
- 광고
- 구글 애널리틱스
- 데이터 시각화 등의 유틸리티 라이브러리
- 봇탐지를 위한 reCAPTCHA

3P 리소스는 웹사이트의 다양한 기능을 추가하는데 도움이 될 수 있지만, 다음의 경우 속도를 늦출 수 있다.

- 3P 도메인 서버에 대한 추가적인 `라운드 트립`이 발생한다.
- 무거운 자바스크립트를 실행하거나, 최적화되지 않은 이미지나 비디오로 리소스의 크기가 늘어난다.
- 개발자 입장에서 3P의 구현에 영향력이 없고, 3P의 구현에 휘둘린다.
- 페이지 내의 다른 중요한 리소스의 렌더링을 차단하고, Core Web Vitals에 악영향을 줄 수 있다.

### 3P 리소스의 성능 영향도 평가

여러 기술을 조합하여 3P가 우리의 웹사이트에 어떤 영향을 주는지 확인할 수 있다.

Lighthouse 감사는 Core Web Vitals에 영향을 주는 느린 3P스크립트를 식별하는데 도움이 된다.

- [메인 스레드를 블록하는 3P 스크립트에 대한 영향도를 줄임.](https://web.dev/third-party-summary/)
- [실행 시간이 긴 3P 스크립트의 실행 시간 단축.](https://web.dev/bootup-time/)
- [용량이 큰 3P 스크립트를 받는데 드는 네트워크 비용 감소.](https://web.dev/total-byte-weight/)
- WebPageTest의 워터폴 차트를 이용하여 [메인스레드를 차단하는 3P스크립트를 식별](https://nooshu.com/blog/2019/10/02/how-to-read-a-wpt-waterfall-chart/#third-party-blocking-javascript)하거나 [WTP side-by-side 비교를 이용하여 3P 태그의 영향을 측정](https://andydavies.me/blog/2018/02/19/using-webpagetest-to-measure-the-impact-of-3rd-party-tags/)한다.
- [Bundlephobia](https://bundlephobia.com/) 같은 사이트들은 npm 패키지를 번들에 추가할 때의 비용을 평가하는데 도움이 된다. [npm 패키지 검색](https://www.npmjs.com/package/)을 사용하여 패키지의 크기와 각 의존 모듈을 찾아볼 수 있다.

### 최적화 전략

3P 코드를 직접 수정하여 최적화할 수 없기에 다음의 두가지 선택지가 있다.

1. 대체하거나 제거
2. 로딩순서 최적화

   브라우저는 로딩 과정에서 1P와 3P 스크립트를 로드한다. 최적의 로딩 전략을 수립하려면 다양한 리소스들에 대해 브라우저에서 할당되는 우선순위, 페이지에서의 위치, 각 리소스의 가치 등을 고려해야 한다.

### 3P 스크립트의 효율적인 로드

`async`, `defer`를 사용하여 스크립트가 다른 컨텐츠를 차단하지 않도록 하기

적용대상 : 중요하지 않은 스크립트

자바스크립트의 다운로드 및 실행은 동기적이기 때문에 메인 쓰레드에서 HTML 파서나 DOM의 생성을 방해할 수 있다.

`<script>` 요소에서 async, defer를 사용하면 브라우저가 스크립트를 비동기적으로 다운로드 할 수 있다.

**`defer`** : HTML 구문 분석 중에 스크립트를 **병렬로 다운로드**하고 HTML 파싱(**분석**)**이 완료될 때까지 실행이 지연**된다. DOM 생성완료 전까지 스크립트의 실행을 지연할 수 있으므로, 항상 사용하는 것을 고려해야 한다.

**`async`** : HTML 구문 분석 중에 **병렬로 다운로드하며, 다운로드 즉시 메인 쓰레드를 블록하며 실행**된다. 모듈 스크립트가 로드될 경우 해당 스크립트와 의존 모듈 모두 defer 큐에서 실행된다. 페이지의 로드 프로세스에 실행되어야 하는 스크립트에는 async를 사용해야 한다. 예 : 페이지의 초기 데이터를 읽을 수 있는 분석 스크립트를 일찍 실행

```javascript
<script src="https://example.com/deferthis.js" defer></script>
<script src="https://example.com/asyncthis.js" async></script>
```

![](/assets/img/2025-06-01/pattern-performance-third-parties-1.png)

### 유의점

async 및 defer는 리소스의 로딩 우선순위를 낮추어 나중에 로드하게 된다. → 새로운 기능인 우선순위 힌트는 이 문제를 해결하는데 도움이 될 수 있다.

- 힌트

```javascript
리소스 힌트를 사용하여 필요 리소스에 대한 연결을 일찍 만들자
적용 대상: 중요한 스크립트, 글꼴, CSS 3P CDN 이미지
```

### dns-prefetch, preconnect

3P 리소스가 위치한 오리진에 연결하는 것은 각 서버마다 DNS 조회, 여러번의 라운드트립을 거쳐야 하기 때문에 느려질 수 있다. dns-prefetch, preconnect와 같은 리소스 힌트들을 사용하면 이런 리소스들에 대한 연결을 더 일찍 하도록 하여 설정에 필요한 시간을 줄일 수 있게 된다.

도메인에 해당되는 dns-prefetch는 DNS 조회가 일찍되어 관련 대기 시간을 줄일 수 있다.

매우 중요한 리소스에 대해서는 preconnect를 사용할 수 있다. preconnect를 사용하면 TCP 라운드트립과 TLS 협상을 처리하고, 추가로 DNS 조회도 진행하여 3P 리소스와의 연결을 시작하게 된다.

```javascript
<head>
  <link rel="preconnect" href="http://example.com" />
  <link rel="dns-prefetch" href="http://example.com" />
</head>
```
