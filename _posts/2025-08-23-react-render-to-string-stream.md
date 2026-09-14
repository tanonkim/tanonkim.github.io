---
title: "React- renderToString, renderToReadableStream"
date: 2025-08-23 16:36:00 +0900
categories: [react, react-study]
tags: [react, ssr]
---

## 1. renderToString

`renderToString`은 React 트리를 HTML 문자열로 렌더링한다. **서버**에서 renderToString을 호출하여 HTML로 렌더링할 수 있다.

```javascript
import { renderToString } from 'react-dom/server';

const html = renderToString(<App />);
```

### 사용 방법

```javascript
import { renderToString } from 'react-dom/server';

// The route handler syntax depends on your backend framework
app.use('/', (request, response) => {
  const html = renderToString(<App />);
  response.send(html);
});
```

### 언제 사용되는가

- 서버사이드 렌더링을 할 때

  예) Next.js의 내부 구현, Express.js 등의 Node 서버에서 SSR을 구현할 때

- 초기 로딩속도 개선 및 SEO 최적화가 필요할 때

  HTML을 클라이언트에 미리 전달하면 브라우저가 페이지를 빠르게 렌더링 가능함

  검색 엔진 크롤러가 HTML 콘텐츠를 읽을 수 있음

  ⇒ 로딩 화면 대신 실제 콘텐츠를 먼저 보여줄 수 있음

### 특징

1. `renderToString`은 문자열을 즉시 반환하므로, 로딩 중인 콘텐츠를 스트리밍하는 것을 지원하지 않는다.

   → Node.js를 사용하는 경우 renderToPipeableStream을 사용하기

   → 최신 엣지 런타임에서 WebStream을 사용하는 경우 `renderToReadableStream` 사용하기

   **예) Next.js SSR 구현**

   ```javascript
   async function renderToString(element: React.ReactElement) {
     const renderStream = await ReactDOMServerPages.renderToReadableStream(element)
     await renderStream.allReady
     return streamToString(renderStream)
   }

   ...
         const html = await renderToString(
             <Body>
               <ErrorDebug />
             </Body>
         )
             ... 중략
   ```

2. 클라이언트에서 `react-dom/server`을 호출하면 불필요하게 번들 크기가 커지므로 피해야 한다.

   → 브라우저에서 컴포넌트를 HTML로 렌더링해야 하는 경우 `createRoot`를 사용하고 DOM에서 HTML을 읽도록 한다.

   ```javascript
   import { createRoot } from 'react-dom/client';
   import { flushSync } from 'react-dom';

   const div = document.createElement('div');
   const root = createRoot(div);

   flushSync(() => {
     root.render(<MyIcon />);
   });
   console.log(div.innerHTML); // 예를 들어, "<svg>...</svg>
   ```

3. Suspense를 완벽하게 지원하지 않는다.

   → 컴포넌트가 lazy와 함께 정의도니 데이터를 가져오는 등 일시중단(Suspend)될 때 콘텐츠가 resolve될 때까지 기다리지 않는다. 가장 가까운 `<Suspense>`을 찾아 fallback 프로퍼티를 HTML에 렌더링한다.

   ⇒ 그러면 lazy되는 내용은 클라이언트코드가 로드될 때까지 나타나지 않는다.

## 2. renderToReadableStream

renderToReadableStream은 Readable WebStream을 이용해 React 트리를 구한다.

```javascript
import { renderToReadableStream } from 'react-dom/server';

async function handler(request) {
  const stream = await renderToReadableStream(<App />, {
    bootstrapScripts: ['/main.js']
  });
  return new Response(stream, {
    headers: { 'content-type': 'text/html' },
  });
}
```

클라이언트에서 hydrateRoot를 호출해 서버에서 생성된 HTML을 상호작용가능하도록 만든다. Promise를 리턴한다.

`<App />`은 root컴포넌트, [’/main.js’]은 bootstrap `<script>`경로로 두 가지가 제공되어야 한다.

root 컴포넌트는 최상위 html 태그를 포함한 문서를 제공해야 한다.

예) root 컴포넌트

```javascript
export default function App() {
  return (
    <html>
      <head>
        <meta charSet="utf-8" />
        <meta name="viewport" content="width=device-width, initial-scale=1" />
        <link rel="stylesheet" href="/styles.css"></link>
        <title>My app</title>
      </head>
      <body>
        <Router />
      </body>
    </html>
  );
}
```

예) script 태그

```javascript
<!DOCTYPE html>
<html>
  <!-- ... 사용자가 직접 작성한 컴포넌트의 HTML ... -->
</html>
<script src="/main.js" async=""></script>
```
