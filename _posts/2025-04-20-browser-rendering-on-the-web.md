---
title: "웹에서 렌더링"
date: 2025-04-20 21:53:00 +0900
categories: [browser, rendering]
tags: [browser, ssr]
---

렌더링 접근 방식의 차이는 페이지 성능 관점에서 렌더링 할 때의 장단점을 보여준다.

용어

아래는 용어에 대한 정리이다.

### 렌더링

React, Next.js 실행 차이

`SSR` : 서버사이드 렌더링

서버에서 앱을 렌더링하여 클라이언트로 JavaScript가 아닌 HTML을 전달

`CSR` : 클라이언트 사이드 렌더링

브라우저에서 자바스크립트로 렌더링하는 방식

(JavaScript를 사용하여 DOM을 수정하여 브라우저에서 앱을 랜더링)

`Rehydration` (재수분 공급)

클라이언트에서 JavaScript로 뷰를 부팅하여 서버에서 랜더링된 HTML의 DOM 트리와 데이터를 재사용

hydration : 이미 렌더링 된 HTML은 정적인데, 여기에 JS와 상호작용이 가능하도록 제어권을 줌

re-hydration : hydration이 잘못 되거나, HTML구조가 바뀌어서 문제가 생기면 re-hydration or hydration missmatch 에러 발생

`사전 렌더링`

브라우저 요청에 사전에 렌더링이 완료된 HTML을 응답하는 렌더링 방식 → CSR의 단점 극복

(빌드 시 클라이언트 측 애플리케이션을 실행하여 초기 상태를 정적 HTML로 캡처)

### 성능

첫 바이트까지의 시간 : `TTFB`

링크를 클릭한 시점부터 새 페이지에 콘텐츠의 첫 번째 바이트가 로드되는 시점까지의 시간

콘텐츠가 포함된 첫 페인트 : `FCP`

요청된 콘텐츠가 표시되는 시간

다음 페인트에 대한 상호작용 : `INP`

페이지가 사용자 입력에 일관되게 빠르게 반응하는지 평가하는 항목

총 차단 시간 : `TBT`

페이지가 로드 중에 기본 쓰레드가 차단된 시간을 계산하는 INP의 프록시 측정항목

## 서버사이드 렌더링

서버 측 렌더링은 요창에 대한 응답으로 서버에서 페이지 전체 HTML을 생성한다. 이렇게 하면 브라우저가 응답을 받기 전에 렌더러가 데이터 가져오기/템플릿 생성을 처리하기에 클라이언트에서 데이터 가져오기 / 템플릿 생성을 위한 추가 왕복이 방지된다.

- 이미지 올려두기

서버사이드 렌더링은 일반적으로 빠른 FCP를 생성한다. 서버에서 페이지 로직을 실행하고 렌더링하면 클라이언트에 많은 JS를 전송하지 않아도 된다.

장점

→ TBT(총 차단 시간)이 줄어들고, 페이지 로드 중에 기본 쓰레드가 자주 차단되지 않아서 INP도 낮아질 수 있다.

→ 사용자가 상호작용이 더 일찍 실행될 기회가 많아진다.

단점

→ 서버에서 페이지를 생성하는데 시간이 걸리므로 페이지의 TTFB(첫 바이트까지의 시간)가 늘어날 수 있다.

**하이브리드 렌더링**

Netflix는 비교적 정적인 방문 페이지를 서버에서 렌더링하는 동시에 상호작용이 많은 페이지의 JS를 prefetching해 더 무거운 클라이언트 렌더링 페이지가 더 빨리 로드될 수 있도록 한다.

## 정적 렌더링

정적 렌더링은 빌드 시 발생한다. 이는 더 빠른 FCP와 더 낮은 **TBT**/**INP**를 제공한다. 서버사이드 렌더링과는 달리 HTML을 서버에서 동적으로 생성할 필요가 없으므로 TTFB도 일관되게 빠르다.

일반적으로 정적 렌더링은 각 URL에 대해 별도의 HTML파일을 미리 생성하는 것을 의미한다.

**장단점**

장점

빌드타임에서 미리 사전 렌더링을 함 → HTML을 서버에서 생성할 필요가 없으므로 빠르다

단점

빌드 타임 이후에는 페이지를 새롭게 생성하지 않는다 (최신 데이터 반영은 어렵다)

모든 URL에 대해 개별 HTML 파일을 생성해야 한다. URL을 미리 예측할 수 없거나 고유 페이지가 많은 사이트의 경우 어려울 수 있다.

**코드(Next.js)**

- page router

  ```javascript

  // getServerSideProps -> getStaticProps
  export const getStaticProps = async (context: GetServerSidePropsContext) => {
      const { id } = context.params!;

      const book = await fetchOneBook(Number(id));

      return {
          props: { book },
      };
  };

  export default function Page({ book }: InferGetServerSidePropsType<typeof getStaticProps>) {
  ```

- app router

  App Router는 Static이거나 Dynamic 둘 중 하나이다.

  1. `generateStaticParams`

     ```javascript
     // book/[id]/page.tsx

     // export const dynamicParams = false; // 명시된 값, 즉 1,2,3 이외에는 404 페이지로 보냄

     // 서버 측에 풀라우트 캐시로서 잘 저장됨
     export function generateStaticParams() {
         return [{ id: '1' }, { id: '2' }, { id: '3' }]; // parameter에 해당하는 값을 빌드 타임에 정적으로 생성
     }

     export default async function Page({ params }: { params: { id: string | string[] } }) {
         const response = await fetch(`${process.env.NEXT_PUBLIC_SERVER_URL}/book/${params.id}`);

         if (!response.ok) {
             if (response.status === 404) {
                 notFound();
             }
             return <div>오류가 발생했습니다.</div>;
         }
     ```

     ![](/assets/img/2025-04-20/browser-rendering-on-the-web-1.png)

  2. `dynamic = ‘force-static’`

     페이지를 강제로 static하게 한다.

     ```javascript
     export const dynamic = 'force-static'; 
     // Dynamic -> Static : 검색 결과가 나오지 않음, 이유 : Dynamic 페이지 -> Static 페이지 강제로 변환
     // static으로 되면 searchParams가 빈 값으로 고정됨 -> 검색해도 값이 나오지 않음

     export default async function Page({
         searchParams,
     }: {
         searchParams: {
             q?: string;
         };
     }) {
         const response = await fetch(
             `${process.env.NEXT_PUBLIC_SERVER_URL}/book/search?q=${searchParams.q}`,
             { cache: 'force-cache' } // 데이터 캐시 최대한 활용
         );
     ```

### 서버사이드 렌더링과 정적 렌더링 비교

- 서버사이드 렌더링은 동적 특성으로 인해 컴퓨터 오버헤드 비용이 발생할 수 있어 모든 경우 적합한 솔루션이 아니다.
  - SSR은 조기에 플러시하지 않거나, TTFB를 지연시키지 않거나, 전송되는 데이터를 두배로 늘리지 않는다.
  - React에서 `renderToString()` 동기 및 단일 쓰레드라서 느릴 수 있다. (최신 React DOM API는 스트리밍을 지원한다.)
- 서버사이드를 올바르게 실행하려면 캐싱을 하거나, 메모리 사용량을 관리하고, 메모이제이션 기법을 사용해야 한다.
  - 콘텐츠를 더 빨리 표시하는 SSR로 인해 작업량이 반드시 줄어드는 것은 아닐 수 있다.
- 서버사이드 렌더링은 각 URL에 대해 HTML을 동적으로 생성하는데, 정적 렌더링된 콘텐츠를 제공하는 것 보다 느릴 수 있다.
  - 서버사이드 렌더링과 HTML 캐싱을 통해 렌더링 시간을 줄일 수 있다.
  - 서버사이드 렌더링의 장점은 정적 렌더링보다 더 많은 실시간 데이터를 가져올 수 있고, 요청에 응답할 수 있다.

## 클라이언트 사이드 렌더링

클라이언트 렌더링은 JavaScript를 사용하여 브라우저에서 페이지를 직접 렌더링하는 것을 의미한다. 모든 로직, 데이터 fetching, 템플릿, 라우팅 등은 클라이언트에서 처리한다. 결과적으로 더 많은 데이터가 서버에서 사용자한테 전달된다.

단점

→ 클라이언트 사이드 렌더링은 모바일에서 빠르게 실행하고 유지하기 어려울 수 있다.

애플리케이션이 커지면서 JavaScript 양이 늘어나는 경향이 있어 페이지의 **INP**에 영향을 줄 수 있다. 페이지의 콘턴츠를 렌더링하기 전에 새로운 JS라이브러리, 폴리필, 서드파티 코드가 추가되면 더 느려질 수 있다.

CSR을 사용하고 JavaScript 번들에 의존하는 환경은 TBT/INP를 낮추기 위해 적극적인 코드 분할을 고려하고, JavaScript를 지연 로드하여 필요할 때만 사용자에게 필요한 항목만 제공해야 한다.

- 상호작용이 없는 환경은 서버사이드 렌더링, 정적 렌더링이 솔루션이 될 수 있다.

## 리하이드레이션

하이드레이션 : HTML로만 된 렌더링 페이지에 JS를 연결(이벤트 등 상호작용을 하게 해줌) → TTI라고 부름

재수화(리하이드레이션)은 서버사이드 렌더링과 클라이언트 사이드 렌더링을 결합한다. CSR, SSR 모두 실행하여 두가지 절충점을 완화하려는 접근 방식이다.

리하이드레이션을 사용한 서버사이드 렌더링의 주요 단점은 FCP가 개선되더라도, TBT/INP에 상당히 부정적인 영향을 미칠 수 있다. 서버사이드에서 렌더링된 페이지는 상호작용이 가능한 것 처럼 보일 수 있지만, 클라이언트 사이드 스크립트가 실행되고, 이벤트 핸들러가 연결될 때까지 실제로 입력에 응답할 수 없다.

## 서버사이드 및 점진적 리하이드레이션

스트리밍 서버사이드 렌더링을 이용하면 브라우저가 점진적으로 렌더링할 수 있는 청크 단위로 HTML을 전송할 수 있다. 이러면 사용자에게 마크업을 더 빠르게 제공하여 FCP 속도를 높일 수 있다.

- React에서 동기식 `renderToString()`에 비해 `renderToPipeableStream()`에서 스트림이 비동기식은 데이터흐름 제어 매커니즘(백프레셔)가 잘 처리되는 의미이다.

**React의 점진적 재수화**

> Suspense로 감싸진 컴포넌트는 다른 부분과 독립적으로 hydration될 수 있다.

이는 전체 애플리케이션이 아닌 특정 부분만 먼저 상호작용 가능한 상태가 될 수 있다는 것을 의미한다. (비동기)

1. 이 접근 방식을 사용하면 전체 애플리케이션을 한번에 초기화하는 일반적인 접근 대신 서버사이드 렌더링 애플리케이션의 개별 부분이 시간이 지남에 따라 부팅된다.

   → 이러면 페이지를 대화형(상호작용 가능)으로 만드는데 필요한 JavaScript의 양을 줄일 수 있다.

   이유 : 페이지의 우선순위가 낮은 부분의 클라이언트 측 업그레이드를 지연하여 기본 쓰레드를 차단하지 않도록 할 수 있다.

React는 이 상태에서 더 이상 렌더링 작업을 진행하지 않고, 현재의 폴백(Fallback) 상태를 유지한다. ⇒ 이는 서버에서 렌더링 된 HTML을 유지하면서 클라이언트에서 필요한 데이터가 준비될 때까지 기다리는 전략이다.

```javascript
// 구조
<Suspense key={searchParams.q} fallback={<SkeletonUi/>}>
		<AsyncComponent/>
</Suspense>
```

**부분 재수화**

페이지의 개별 부분(구성요소, 뷰/트리)을 분석하고 상호작용이 적거나 반응이 없는 부분을 식별하는 점진적 재수화의 확장이다. JavaScript코드는 비활성 참조 및 장식용 기능으로 변환되어 클라이언트 측 공간을 거의 0으로 줄인다.

**삼변형 렌더링**

![](/assets/img/2025-04-20/browser-rendering-on-the-web-2.png)

삼변형 렌더링을 사용하면

1. 초기 탐색 또는 JS가 아닌 탐색에 스트리밍 서버사이드 렌더링을 사용하고,
2. 서비스 워커가 설치된 이후에 탐색을 위한 HTML 렌더링을 처리하도록 할 수 있다.

이렇게 하면 캐시된 구성요소와 템플릿을 최신 상태로 유지하고, 동일한 세션에서 새 뷰를 렌더링하기 위한 SPA 스타일 탐색을 설정할 수 있다.

서버/클라이언트/서비스 워커 간 템플릿 공유 및 라우팅 코드를 공유할 수 있는 경우에 적합하다.

Isomorphic JS
