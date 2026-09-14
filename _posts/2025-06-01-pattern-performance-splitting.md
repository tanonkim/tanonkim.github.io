---
title: "Performance Patterns 3 : splitting"
date: 2025-06-01 00:20:00 +0900
categories: [design-pattern, performance]
tags: [design-pattern, code-splitting]
---

## 라우트 경로에 따른 코드 스플리팅

라우팅 경로 기반 코드 스플리팅을 사용하면 특정 경로에서만 필요한 리소스를 로드하여 사용하도록 할 수 있다. React 의 **`Suspense`** 혹은 **`loadable-components`** 와 **`react-router`**를 함께 사용하여 현재 라우팅 경로에 해당하는 컴포넌트를 동적으로 로드할 수 있다.

```javascript
import React, { lazy, Suspense } from "react";
import { render } from "react-dom";
import { Switch, Route, BrowserRouter as Router } from "react-router-dom";

const App = lazy(() => import(/* webpackChunkName: "home" */ "./App"));
const Overview = lazy(() =>
  import(/* webpackChunkName: "overview" */ "./Overview")
);
const Settings = lazy(() =>
  import(/* webpackChunkName: "settings" */ "./Settings")
);

render(
  <Router>
    <Suspense fallback={<div>Loading...</div>}>
      <Switch>
        <Route exact path="/">
          <App />
        </Route>
        <Route path="/overview">
          <Overview />
        </Route>
        <Route path="/settings">
          <Settings />
        </Route>
      </Switch>
    </Suspense>
  </Router>,
  document.getElementById("root")
);

module.hot.accept();
```

## Bundle Splitting

웹 앱을 빌드할 때 Webpack이나 Rollup과 같은 번들러들은 앱의 소스 코드를 받고 이를 하나 이상의 `번들`로 만들어 준다. 사용자가 웹 사이트에 접속하면 만들어진 번들들이 다운로드고 로드되며 사용자의 화면에 데이터를 출력하게 된다.

V8과 같은 JS엔진들은 로드되는 동안 사용자가 요청한 데이터에 대해 구문을 분석하고 컴파일 할 수 있다. 최신 브라우저들은 코드들을 성능 좋고 빠르게 구문 분석하고 컴파일하도록 발전했지만, 요청한 데이터와 관련된 코드에 대한 로딩 시간과 실행 시간은 아직 개발자가 직접 최적화해야 한다. 이 실행 시간을 가급적 짧게 유지하여 메인 쓰레드를 블록하지 않도록 해야 한다.

최신 브라우저들은 만들어진 번들들을 **`스트리밍`** 할 수 있지만, 사용자의 화면에 첫 픽셀을 그려내기까지 시간이 꽤 걸릴 수 있다. 스트리밍이란 파일 전체를 한번에 받는 것이 아니라 부분부분씩 받아 적용하는 것을 뜻한다.

사용자가 보는 화면에 데이터를 빨리 보여줄 필요가 있다. 번들 크기가 클수록 로딩 타임, 프로세스 타임, 실행 시간이 늘어난다. 따라서 번들 크기를 줄이는 것이 이런 시간들을 줄이는 방법이 될 수 있다.

### 번들 분리

번들 코드를 나누어 로드 및 실행 시간에 걸리는 시간을 줄일 수 있다. 이는 성능 측정 항목 중 `FCP` (First Contentful Paint)를 향상시킨 것이며 이어서 화면에서 큰 **영역을 차지하는 컴포넌트의 렌더링 속도도 향상**시켰다. → `LCP` (Largest Contentful Paint)를 향상시킨 셈이다.

화면에 무언가 빨리 나타나지만 앱이 제대로 기능하려면 사용자의 인터렉션도 빠르게 가능해져야 한다. 번들 파일들이 로드되고 실행 완료 되어야만 UI가 인터렉트 가능해진다. 화면이 그려지고 사용자가 모든 요소들에 인터렉트 가능해지기까지의 시간은 `TTI` (Time To Interactive) 로 측정한다.

자바스크립트 엔진은 사용자의 화면에 아무것도 나오지 않는 시점이며 실제로 사용하지 않을지도 모르는 코드들도 구문분석하고 컴파일해야 한다. 브라우저의 성능이 좋아서 이 과정에 드는 비용이 높지 않을 수 있지만, 크기가 큰 번들을 다운로드하도록 하는 것은 여전히 성능에 문제를 일으킬 수 있다. 사용자의 단말 성능이 떨어지거나 네트워크 상태가 불안정한 경우 사용자는 번들이 다운로드되기까지 꽤 많은 시간을 기다려야 할 수 있다.

크기가 큰 번들을 `a.js`, `b.js` 두 개의 작은 번들로 나누었다. 이로써 최초 로딩 시점에 더 작은 데이터만 받아도 되도록 하여 성능을 향상시켰다.
