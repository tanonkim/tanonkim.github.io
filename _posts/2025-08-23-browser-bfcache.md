---
title: "BFCache"
date: 2025-08-23 18:53:00 +0900
categories: [browser, cache]
tags: [browser, bfcache]
---

## BF 캐시란?

BFCache는 Backward/Forward 캐시의 약자로, 브라우저에서 렌더링된 페이지의 전체 스냅샷을 메모리에 보관하는 메커니즘이다. 즉. 뒤/앞으로 이동 시 웹페이지가 로딩 시간 거의 없이 즉시 렌더링된다. BFCache가 없으면 브라우저는 전체 웹페이지를 처음부터 로드해야 하므로, 서버에 다시 연결해 모든 정적 에셋을 다시 다운로드하고 모든 스크립트를 다시 실행해야 한다. 하지만 BFCache를 사용한다면 빠르고 원활하게 전환이 가능하다.

요약 : BF 캐시는 브라우저가 페이지의 상태를 메모리에 저장했다가, 사용자가 뒤로/앞으로 이동하면 그 페이지를 **다시 그리는 것이 아니라 메모리에서 복원**하는 방식이다.

## BFCache 동작 방식

BFCache는 전체 웹페이지의 스냅샷을 찍어 메모리에 저장하며, 반복 탐색 속도를 높이는데 자체 역할을 하는 HTTP 캐시와는 다르다.

그리고 실행되던 자바스크립트의 작업 큐에 대기중이던 작업은 일시중지된다. 즉 setInterval or setTimeout, Promise 등 실행이 예약된 코드는 BFCache에서 페이지가 제공될 때 다시 시작한다.

### 트랜잭션

indexedDB 트랜잭션을 다룰 때 동작은 더 까다로워지는 경향이 있다. 트랜잭션이 진행중일 때 사용자가 페이지에서 다른 곳으로 이동하면 사용자가 떠나기 전에 연결이 닫히지 않는 한 브라우저는 BFCache에 페이지를 캐시하지 않는다.

일반적으로 브라우저는 사용자의 편에서 합리적이라고 판단되는 경우에만 웹페이지를 BFCache에 캐시를 한다. 그리고 대부분의 브라우저는 모바일과 데스크탑에서 BFCache를 지원한다.

## BFCache와 호환이 되는 웹페이지

BFCache에서 제공되는 페이지의 수명 주기 이벤트를 알아두는 것이 중요하다.

페이지 load 이벤트는 BFCache에서 페이지가 로드될 때 발생하지 않는다. 따라서 pageshow 이벤트를 수신해야 하며, event.persisted 속성을 확인하여 BFCache에서 복원되었는지 확인할 수 있다.

```javascript
window.addEventListener('pageshow', function(event) {
  if (event.persisted) {
    // bfcache에서 페이지가 복원됨
  }
});
```

`unload` 이벤트 수신과 같은 일부 작업은 BFCache가 되지 못하도록 하므로, 이벤트를 추가하지 않아야 한다. (현재 레거시 API로 간주된다.)

## 테스트

크롬 개발도구를 열고 애플리케이션 탭으로 이동한 다음 Back/forward cache를 클릭하면 BFCache를 쉽게 테스트할 수 있다. 이 페이지에서 커다란 'Test back/forward cache 버튼'을 누르면 크롬이 자동으로 웹페이지로 이동했다가 다시 돌아와 페이지가 캐시에 진입했는지 여부를 표시한다.

![](/assets/img/2025-08-23/browser-bfcache-1.png)

### BFCache와 iframe

페이지에 iframe이 삽입된 경우 iframe 자체는 별도로 bfcache를 사용할 수 없다. iframe내의 url이 이동되면 이전 콘텐츠가 bfcache에 들어가지 않는다. 웹소켓도 마찬가지이다.

### Next.js의 Router Cache

> *Next.js has an in-memory client-side router cache that stores the RSC payload of route segments, split by layouts, loading states, and pages.*

공식문서에서는 Router cache를 “in-memory client-side router cache”라고 칭한다. 여기서 “in-memory”란 데이터를 하드 디스크와 같은 저장 장치가 아니라 메모리(RAM)에 저장한다는 개념으로 빠르게 데이터를 읽고 쓸 수 있다는 장점과 휘발된다는 단점이 있다.

Router cache는 라우트 세그먼트의 RSC Payload를 layouts, loading states, pages별로 분리해서 저장한다.

- layouts은 캐싱되며, 네비게이션할 때 재사용된다.
- loading states도 캐싱되며, 네비게이션할 때 재사용된다.
- pages는 기본적으로 캐시되지않지만, 브라우저 앞으로가기/뒤로가기 때는 재사용된다. (‘staleTimes’ 옵션을 통해 페이지 세그먼트 캐싱을 활성화할 수 있다.)

Next.js는 사용자가 이미 방문한 라우트 세그먼트를 캐시하고, 이동할 가능성이 있는 라우트 세그먼트들을 prefetch한다. 이로써 즉각적인 앞으로 가기/뒤로가기, 풀페이지 새로고침 없는 네비게이션이 가능하며, React와 브라우저의 상태를 보존한다. (브라우저의 BF cache와 다르지만 비슷한 효과를 낸다.)

| ❌ 리퀘스트 메모이제이션 | 없음 | `fetch` 요청을 캐싱해서 중복 호출을 막는 개념. BF 캐시랑 무관 |
| --- | --- | --- |
| ❌ 데이터 캐시 | 없음 | React Query나 Next.js의 `fetch` 캐시 등. 이것도 브라우저 메모리 복원과는 무관 |
| ❌ 풀라우트 캐시 | 없음 | 전체 페이지를 정적 HTML로 캐싱하는 개념. CSR에서는 BF 캐시랑 별 상관 없음 |
| ✅ **라우터 캐시** | **약간 관련 있음** | Next.js App Router에서 페이지 간 이동 시 데이터나 컴포넌트를 유지해주는 캐시. 하지만 BF 캐시는 어디까지나 브라우저가 결정하는 것이라, 직접적인 제어는 어려움 |

**결론**

- `unload` 이벤트 같은 방해 요소가 없고
- Next.js App Router를 사용해 CSR 중심 구조이기 때문이기에

**라우터 캐시**와 약간 관련은 있지만, **BF 캐시는 브라우저 주도 기능**이다.
