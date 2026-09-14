---
title: "Rendering Patterns 4 : Progressive Hydration, Streaming"
date: 2025-06-08 17:30:00 +0900
categories: [design-pattern, rendering]
tags: [design-pattern, hydration]
---

## 1. Progressive Hydration

### Hydration

서버에서 현재 탐색한 페이지에 대한 HTML이 생성된다. 서버가 정적 UI표현에 꼭 필요한 CSS나 JSON 데이터를 포함한 HTML컨텐츠를 만들고 나면, 즉시 클라이언트에 전송된다. 서버에서 마크업을 이미 만들었기 때문에 클라이언트는 내용을 파싱하고, 화면에 빠르게 출력한다. ⇒ FCP 시간이 빠르게 측정된다.

FCP가 빠르다고 하여 TTI가 빠른 것은 아니다. 아직 인터렉션을 가능하게 하는 자바스크립트가 로드되지 않았기 때문이다. 버튼은 보이긴 하지만, 동작하진 않는다.

자바스크립트 번들이 로드된 이후에 바인딩 되는데, 이 과정을 Hydration이라고 한다. React는 현재의 DOM 노드들을 확인하고, 각 노드를 대응하는 자바스크립트에 hydration한다.

사용자가 화면에 보이는 UI들과 인터렉션할 수 없을 때까지의 시간을 `불쾌한 골짜기`라고한다. 사용자는 해당 시간에 UI가 굳은 것 처럼 느낄 수 있다.

서버에서 받은 DOM 컴포넌트들이 완전히 hydration될 때까지는 시간이 걸린다. 컴포넌트가 hydrate가 가능해질때까지 자바스크립트 파일들은 로드되고 처리되며 실행한다.

### Progressive Hydration

앱이 한 번에 hydrate되도록 하는 대신, DOM노드마다 점진적으로 진행할 수 있다. 점진적 hydration은 위에서 언급했던 시간 동안에 개별 node별로 hydrate할 수 있도록 하여 최소 필요한 자바스크립트만을 요청하여 처리할 수 있게 한다.

앱을 점진적으로 hydration하여 덜 중요한 부분의 hydration을 지연할 수 있다. 이런 방식으로 페이지를 인터렉션 가능하게 만들기 위해 받아야 하는 자바스크립트 크기를 줄일 수 있고, 또 사용자에게 필요로 하는 노드만 hydration할 수 있다.

점진적 hydration은 SSR의 rehydration 과정에서 서버가 렌더링한 DOM 트리가 제거되고 다시 만들어질 때 발생할 수 있는 문제들을 겪지 않게 해줄 수 있다.

참고 : 점진적 hydration은 특정 조건에 만족하는 컴포넌트에 대해서만 hydrate할 수 있도록 한다.

![](/assets/img/2025-08-23/pattern-rendering-hydration-streaming-1.png)

![](/assets/img/2025-08-23/pattern-rendering-hydration-streaming-2.png)

### 점진적 hydration 구현하기

hydration은 클라이언트 측 React가 서버에서 렌더링 된 ReactDOM 컴포넌트를 인지하고, 이벤트 핸들러가 바인딩할 수 있게 해준다. 따라서 SSR앱에 클라이언트에서 활성화되니 CSR앱과 같이 연속성과 끊임없이 부드러운 사용성을 제공한다.

페이지 내의 모든 컴포넌트들이 hydration을 통해 인터랙트가 가능하기 위해 React의 코드들이 번들에 포함되어 클라이언트에 다운로드 된다. 인터랙트가 매우 중요한 SPA는 번들이 한번에 필요하겠지만, 대부분 인터랙트가 가능한 요소가 적은 정적 웹사이트는 모든 컴포넌트가 직접 즉시 인터랙트가 가능해질 필요가 없다. 이런 사이트에 각 컴포넌트를 위해 크기가 큰 React 번들을 전송하는 것은 오버헤드가 될 수 있다.

⇒ 점진적 hydration은 페이지가 로드될 때 앱의 일정 부분에 대해서만 hydrate를 가능하게 하여 이런 문제를 해결한다. 다른 부분들에 대해 따라 점진적으로 hydration이 된다.

![](/assets/img/2025-08-23/pattern-rendering-hydration-streaming-3.png)

사진 처럼 “You may also like”, “Other Content”는 나중에 필요에 따라 hydrate될 수 있다.

전체 앱이 한번에 초기화되는 대신, hydration 과정은 DOM 트리의 루트부터 시작되지만, 서버 렌더링 앱의 개별적인 부분들은 일정 기간동안만 활성화된다. Hydration 과정은 다양한 지점에서 정지될 수 있고, 각 부분들이 뷰포트에 들어오는 등의 조건을 만족할 때 재시작될 수 있다.

주목할 점은 hydration을 위한 리소스 역시 코드 스플리팅에 의해 지연될 수 있다. → 따라서 페이지를 인터랙트하게 만드는 데 필요한 자바스크립트의 양이 줄어든다.

### 점진적 hydration에 대한 요구사항

점진적 hydration의 기본 개념은 앱의 코드를 분할하고, 개별적으로 활성화하여 좋은 성능을 이끌어내는 것이다. 모두 점진적 hydratino 솔루션을 적용했을 때 사용자 경험에 어떤 영향을 줄 수 있는지 고려해야 한다.

화면의 각 부분에 대한 코드들을 화면에 보여지는 순서를 조절할 수는 없지만, 이미 로드된 코드 부분이 활성화되는 것이나, 사용자의 입력을 막게 될 수는 있다. 따라서 아래는 전체적인 점진적 hydration에 대한 요구사항이다.

1. 모든 컴포넌트에 대해 SSR을 사용한다.
2. 코드 스플리팅을 지원해 개별 컴포넌트 혹은 부분별로 코드를 나눌 수 있어야 한다.
3. 분할된 코드들에 대한 hydration 순서를 개발자가 정의할 수 있어야 한다.
4. 이미 hydrate된 코드들에 대해 사용자의 입력을 막으면 안된다.
5. hydration이 지연된 분할 코드에 대해 로딩 화면 등을 사용할 수 있어야 한다.

### 동시성 모드

React의 동시성 모드를 활용하면 위 요구사항을 모두 충족할 수 있다. 동시성 모드는 React가 여러 작업을 동시에 진행할 수 있게 해주고, 우선 순위에 따라 해당 작업을 전환할 수 있게 해준다. 교체할 때 부분적으로 렌더링 된 트리는 커밋(VDOM을 실제 DOM에 반영하는 동작)할 필요가 없기 때문에 React가 동일한 작업으로 다시 전환되었을 때 렌더링 작업을 이어 진행할 수 있다.

동시성 모드는 점진적 hydration을 구현하는데 사용될 수 있따. 사용자의 우선순위가 높은 작업이 들어올 경우 React는 hydration 작업을 일시적으로 멈추고, 사용자 입력을 받아들이는 작업으로 전환한다.

`lazy`, `Suspense` 는 로딩 상태를 변수로 제공한다. → 해당 변수를 사용해 분리된 코드들이 로딩중일 때 스피너를 화면에 보여줄 수 있다.

### 구현1. (라이브러리)

Preact와 Next.js를 사용하여 이를 구현한 예제를 살펴보자.

이 예제는 아래 라이브러리를 사용하여 부분 hydration을 구현하고 있다.

1. `pool-attendant-preact` : Preact x 를 활용하려 부분 hydration을 구현한 라이브러리
2. `next-super-performance` : 클라이언트 측 성능을 위해 사용하는 Next.js 플러그인

```javascript
import Teaser from './teaser'
import { withHydration } from 'next-super-performance'

const HydratedTeaser = withHydration(Teaser)

export default function Body() {
  return (
    <main>
      <Teaser column={1} />
      <HydratedTeaser column={2} />
      <HydratedTeaser column={3} />

      <Teaser column={1} />
      <Teaser column={2} />
      <Teaser column={3} />

      <Teaser column={1} />
      <Teaser column={2} />
      <Teaser column={3} />
    </main>
  )
}
```

2, 3번째로 렌더링 되는 `HydratedTeaser` 가 제일 먼저 hydrate된다.

### 구현2. dynamic import + Suspense + Lazy

**⚠️ 서버 컴포넌트는 클라이언트 번들에 포함되지 않으므로 hydration이 필요없다.**

클라이언트 컴포넌트 (use client)는 hydration의 적용 대상이다. ⇒ 필요한 부분만 클라이언트 컴포넌트로 만들어서 점진적 hydration으로 작동되게 한다.

서버 컴포넌트는 서버에서만 렌더링되고, hydration이 되지 않는다. ⇒ TTI === FCP

예1) client 컴포넌트만 hydration의 적용 대상

```javascript
// app/page.tsx
import MyHeavyComponent from './MyHeavyComponent';

export default function Page() {
  return (
    <main>
      <h1>Hello World</h1>
      <MyHeavyComponent />
    </main>
  );
}

// app/MyHeavyComponent.tsx
"use client"; // 클라이언트 컴포넌트만 hydration됨

export default function MyHeavyComponent() {
  return <div>I'm interactive!</div>;
}
```

예2) Next.js에서 동적 import + Suspense

```javascript
import dynamic from 'next/dynamic';

const MyHeavyComponent = dynamic(() => import('./MyHeavyComponent'), {
  ssr: false, // 서버 사이드 렌더링은 하지 않음 -> hydration이 클라이언트에서만 발생
  loading: () => <p>Loading...</p>,
});

export default function Page() {
  return (
    <main>
      <h1>Hello World</h1>
      <MyHeavyComponent />
    </main>
  );
}
```

사용자가 스크롤하거나 컴포넌트를 보고 있을 때 해당 컴포넌트가 로드된다. → 실제로도 progressive hydration처럼 작동한다.

예3) React에서 점진적 hydration 사용

```javascript
import { Suspense, lazy } from 'react';

const MyHeavyComponent = lazy(() => import('./MyHeavyComponent'));

export default function App() {
  return (
    <div>
      <h1>Hello World</h1>
      <Suspense fallback={<p>Loading...</p>}>
        <MyHeavyComponent />
      </Suspense>
    </div>
  );
}
```

React.lazy + Suspense는 hydration 시점을 유연하게 분리시켜 progressive하게 동작하게 한다.

### **점진적 hydration - 장단점**

점진적 hydration은 hydration 자체의 비용을 최소화하며 서버사이드 렌더링과 클라이언트 사이드 hydration을 제공한다.

**장점**

1. 코드 스플리팅

   개별 컴포넌트들에 대해 코드가 분리되어 지연로딩되어야 하기 때문에 코드 스플리팅은 점진적 hydration에 필수적인 기능이다.

2. 자주 사용되지 않는 요소를 지연로딩

   정적이거나 뷰포트를 벗어나 있거나 자주 사용되지 않는 컴포넌트들은 지연 로딩의 대상이다. 이런 컴포넌트를 hydration하는 코드 역시 페이지를 로드하는 시점에 받을 필요가 없다.

3. 번들 사이즈 감소

   코드 스플리팅으로 인해 번들 사이즈가 감소한다. 페이지를 로드할 때의 코드량이 줄어들기에 FCP와 TTI간의 시간 차가 줄어든다.

**단점**

hydration은 페이지 로드와 함께 페이지 내의 모든 요소들이 인터랙션이 가능해야 하는 동적인 앱에는 적합하지 않다. 사용자가 어떤 부분을 인터렉션을 먼저 할 지 알 수 없기 때문이다.

---

## 2. Streaming SSR

서버 렌더링을 할 때 현재 탐색된 웹페이지를 위한 전체 HTML을 만들지 않고, 서버에서 앱의 컨텐츠를 스트리밍처리하도록 하여 여러 조각으로 나눌 수 있다. Node의 Stream 모듈은 응답에 데이터 스트림을 포함시킬 수 있고 클라이언트에게 지속적인 데이터를 전달할 수 있다는 의미이다. 클라이언트가 나눠진 코드들을 받으면 렌더링을 시작한다.

React의 `renderToPipeableStream`을 사용하면 앱을 여러 부분으로 나누어 전송할 수 있게 된다. 서버로부터 받은 DOM노드들에 대해 `hydrate`를 호출하여 각각에 대응하는 이벤트 핸들러를 바인딩해 인터렉트 가능하도록 만들 수 있다.

⇒ 데이터를 수신하면서도 UI를 그릴 수 있기에 좋은 성능을 이끌어낼 수 있다.

![](/assets/img/2025-08-23/pattern-rendering-hydration-streaming-4.png)

![](/assets/img/2025-08-23/pattern-rendering-hydration-streaming-5.png)

### React

예) 다수의 데이터를 렌더링하는 React App

```javascript
import React from "react";
import facts from "../data"; // 100개 데이터 존재

const App = () =>
  facts.map((fact, i) => (
    <div className="card">
      <p key={i}>{fact.fact}</p>
    </div>
  ));

export default App;

// facts 형태
// {
//  fact:
//    "Tests done by the Behavioral Department of the Musuem of Natural History conclude that while a dog's memory lasts about 5 minutes, a cat's recall can last as long as 16 hours.",
//  length: 175
// },
```

App 컴포넌트를 렌더링하기 위해 renderToString을 사용한다면 사용자는 앱이 데이터를 로드하고 메타데이터를 처리하기 전에 모든 데이터를 받을 때 까지 기다려야 한다. 이를 처리하기 위해 renderToPipeableStream을 사용하면 App 컴포넌트의 분리된 코드를 받자마자 곧바로 데이터 로드 및 처리를 시작할 수 있다.

### 컨셉

점진적 hydration과 같이 스트리밍도 렌더링 매커니즘으로 SSR 성능 향상을 위해 사용할 수 있다. 이름 그대로 스트리밍은 노드 서버에서 HTML을 여러 조각으로 나누어 생성되는대로 클라이언트에 전달한다.

페이지의 마크업 사이즈가 크더라도 HTML의 일부를 미리 받아 그릴 수 있어 TTFB가 상대적으로 감소한다. 메이저 브라우저들은 스트림된 컨텐츠나 부분적인 응답을 파싱하고 렌더링하여 FP, FCP까지의 시간을 감소시킨다.

스트리밍은 네트워크 신호에 따라 유연하게 동작한다. 네트워크가 막혀 더 이상 추가적인 바이트를 전송할 수 없게 되면 렌더러는 신호를 받아 상태가 정리될 때까지 스트리밍을 멈춘다. 따라서 서버는 적은 메모리를 사용하며, I/O 컨디션에 따라 유동적으로 동작한다. → 이로 인해 Node.js 서버는 여러 요청을 동시에 처리할 수 있고, 무거운 요청이 가벼운 요청의 응답을 방해하는 것을 막을 수 있다.

### React에서 스트리밍 하기

**참고** : Next.js의 App Router 내부적으로 `renderToPipeableStream`를 사용해 Streaming SSR, SSG를 지원한다.

1. renderToPipeableStream

   `renderToPipeableStream`는 React 18+에서 추가된 API로, SSR을 Node.js 스트림(`pipe`) 형태로 출력할 수 있게 한다. 이 덕분에 HTML이 **조금씩** 렌더링되어 클라이언트로 전송되면서 사용자 경험을 개선할 수 있다 → (점진적 hydration).

   참고 : node.js환경에서만 사용이 가능하다.

   ```javascript
   import express from "express";
   import React from "react";
   import { renderToPipeableStream } from "react-dom/server";
   import App from "./App";

   const app = express();

   app.get("*", (req, res) => {
     const { pipe, abort } = renderToPipeableStream(
       <App />,
       {
         onShellReady() {
           // HTML shell이 준비되면 헤더 설정
           res.statusCode = 200;
           res.setHeader("Content-Type", "text/html");

           // React의 스트림을 응답으로 연결
           pipe(res);
         },
         onShellError(error) {
           // shell 렌더링 중 에러가 발생하면 fallback 처리
           res.statusCode = 500;
           res.send("<h1>Something Wrong</h1>");
         },
         onError(error) {
           console.error(error);
         }
       }
     );

     // 일정 시간 이후 abort 시도 가능 (예: 5초)
     setTimeout(abort, 5000);
   });

   app.listen(3000, () => {
     console.log("Server listening on http://localhost:3000");
   });


   // App.tsx
   import React, { Suspense } from "react";
   import MyComponent from "./MyComponent";

   export default function App() {
     return (
       <html>
         <head>
           <title>Streaming SSR Example</title>
         </head>
         <body>
           <Suspense fallback={<p>Loading...</p>}>
             <MyComponent />
           </Suspense>
         </body>
       </html>
     );
   }
   ```

   - 동작흐름
     1. 렌더링 시작

        `renderToPipeableStream(<App />, options)`로 React 앱 렌더링

        내부에 Suspense 컴포넌트가 있으면 fetch 등의 비동기 작업이 끝날때 까지 부분적으로 렌더링

     2. HTML Shell 준비

        onShellReady()에서 HTML shell이 준비되면 res.setHeader() 설정 후 pipe(res)로 스트리밍 전송

     3. 점진적 렌더링

        Suspense Boundary가 준비될 때마다 추가로 HTML 전송

     4. Hydration

        점진적으로 hydration을 수행해 상호작용이 가능

     5. 에러 처리

일반적인 SSR과 스트리밍에 대해 TTFB/FMP 차이

![](/assets/img/2025-08-23/pattern-rendering-hydration-streaming-6.png)

### Streaming SSR 장단점

**장점**

1. 성능 향상

   서버 렌더링 시작 직후, 클라이언트에 즉시 부분적인 데이터가 전송되기 때문에 TTFB가 SSR보다 좋다. 페이지의 크기와 상관없이 일관적으로 유지되며, 클라이언트는 HTML을 즉시 파싱 가능하기에 FP와 FCP역시 빨라진다.

2. 네트워크 상태에 덜 영향을 받음

   네트워크 상태가 좋지 않은 경우에서도 응답성을 유지할 수 있음

3. SEO 지원

   스트리밍 되어 렌더링 된 컨텐츠는 크롤러에서도 읽어들일 수 있어 SEO에도 좋음

**단점**

1. 마이그레이션의 어려움 (React v.19)

   renderToString()으로 구현된 부분을 renderToPipeableStream()으로 교체하기가 어려운 상황이 존재한다. (SSR 스트리밍이 함께 잘 동작하지 않는 상황이 존재한다.)

   1. SSR 처리된 분할 코드를 받기 전 다른 이유로 문서에 서버에서 만든 마크업을 미리 추가해야 하는 프레임워크를 사용하는 경우, 예) 특정 프레임워크가 어떤 CSS를 사용해야 하는지 동적으로 결정하여 페이지에 확인
   2. `renderToStaticMarkup`은 페이지 템플릿을 만들고, `renderToString`은 데이터가 포함된 동적 컨텐츠를 생성한다. renderToStaticMarkup의 인자에서는 문자열이 올 것으로 기대하기에 renderToString의 컴포넌트를 사용하면 동작하지 않는다.
