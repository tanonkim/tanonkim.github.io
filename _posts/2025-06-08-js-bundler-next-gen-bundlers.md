---
title: "차세대 번들러 비교 및 분석"
date: 2025-06-08 20:52:00 +0900
categories: [javascript, bundler]
tags: [javascript, bundler]
---

## 번들러란?

번들러는 여러개의 파일들을 하나의 파일로 묶어주는 도구이다.

### 번들러가 필요해진 이유

태초에 브라우저는 ES Modules를 지원하지 않는다. 이말은 ES6 import 문법을 사용할 수 없고, `<script>`간 모듈을 내보낼 수 없다는 의미이다.

초기에는 몇몇 스크립트 파일만 있으면 서비스가 가능했지만, 웹서비스가 기하급수적으로 복잡해지면서 방대해진 코드들은 관리하기가 어려워졌다. 개발자들은 다양한 전략으로 코드를 여러 파일로 분리했고, `window`, `jQuery`와 같은 **전역객체**를 활용하여 스크립트 파일 간 모듈을 공유했다.

사람들은 웹브라우저에서 더 많은 활동을 하고, 더 좋은 서비스 경험이 요구되었다. 더 작은 용량의 더 최적화된 리소스 제공, 흩어져 있는 여러 파일을 하나로 통합 및 문자 압축, 다양한 사용자 환경에서도 돌아가도록 코드를 변환, 안 쓰이는 소스코드 부분을 분석해서 제거 등, 웹사이트에서 전처리 요구사항이 점점 많아졌고, 이를 자동화시켜줄 친구가 필요해졌다.

여기서 grunt, glub과 같은 도구가 등장했고, `webpack`의 시대가 도래하게 되었다.

24년도의 stateofjs의 자료를 찾아보면, 번들러의 생태계가 빠른 속도로 변하고 있다.

![](/assets/img/2025-06-08/js-bundler-next-gen-bundlers-1.png)

---

## Webpack

2012년 Tobias Koppers에 의해 webpack v1.0이 효율적으로 js파일을 통합해주는 도구로 공개되었다.

2016년에 webpack v2.0이 릴리즈되고 ES6을 지원하여 커뮤니티의 큰 관심을 얻게 되었다.

2017년 webpack v3.0이 릴리즈되고 성능개선, 기능 추가 되면서 사람들이 열광하였다.

2018년 webpack v4.0이 릴리즈되었는데 동시에 React, Angular의 등장으로 SPA가 급부상하면서 번들러의 역할이 막중해지자 대웹팩시대가 도래하였다.

### **webpack의 정체성**

webpack의 정체성은 번들어의 목적인 **`통합`**에 있다. 번들을 통합해서 관리할 수는 없을까에 대한 고민이 webpack을 상승하게 만든 요인이다.

### **webpack의 장점**

하나의 설정파일에서 원하는 번들이 생성될 수 있도록 컨트롤할 수 있다. entry와 output을 명시하고, 어떤 plugin과 loader로 파일들을 다룰 건지 명시만 하면 된다.

예) webpack.config.js

```typescript
const path = require('path');
const HtmlWebpackPlugin = require("html-webpack-plugin");

module.exports = {
  entry: {
    home: './pages/home.js',
  },
  output: {
    path: path.resolve(__dirname, 'dist'),
    filename: '[name].bundle.js',
  },
  module: {
    rules: [
      {
        test: /\.js$/,
        exclude: /node_modules/,
        use: ["babel-loader"],
      },
    ],
  },
  plugins: [new HtmlWebpackPlugin()],
};
```

### **풍부한 plugins와 loaders**

webpack은 굉장히 강력한 개발 커뮤니티가 존재한다. 쉽게 plugin과 loader를 부착할 수 있기 때문인데, loader를 통해서 파일들을 변환/번들링/빌드를 진행하고 plugin을 통해서 output 파일을 튜닝해준다.

### **강력한 개발서버**

webpack은 Hot Module Replacement(HMR)을 처음으로 제안했고 2012년부터 사용해왔다. HMR은 소스코드의 변화를 감지하여 브라우저를 직접 새로고침할 필요 없이 변화를 바로 반영한다. 개발자는 덕분에 신속하게 개발을 할 수 있게 되었다.

webpack v2, v3, v4를 거쳐 다양한 css같은 파일의 변화도 감지할 수 있게 되었고 안정화되었다.

### **`Code Splitting`**

> *"Code splitting is one of the most compelling features of webpack."*

webpack의 가장 큰 기능 중 하나는 code splitting이다. 번들 로드 최적화하는 작업으로 파일들을 여러 번들 파일로 분리하여 병렬로 스크립트를 로드하여 페이지 로딩 속도를 개선할 수 있다. 추가로 초기에 구동할 필요가 없는 코드를 분리하여 lazy loading을 통해 페이지 초기 로딩 속도를 개선할 수 있다.

### Webpack VS grunt

grunt는 task runner이다. minifying, 파일 통합, lint 적용 등 등록된 task를 순서대로 실행시켜 준다. 설정이 파편화되어있고, 개발자가 직접 컨트롤해야 하는 영역이 많다.

반면 webpack은 module bundler로 모듈을 통합하는 것에 초점이 맞춰져있다.

### Webpack VS glup

glup은 grunt와 흡사한 task runner이다. 다만 stream-based로 파일을 접근하기에 grunt에 비해 성능이 더 우수하다고 평가받는다.

## Rollup

webpack의 시대가 도래되면서 2017년부터 개발이 시작된 module bundler이다. webpack의 열풍에 가려져 있었지만, 점차 많은 차세대 번들러가 Rollup을 벤치마킹하게 되었다.

### **Rollup의 정체성**

Rollup의 정체성은 `확장`에 있다.

> *"compiles small pieces of code into something larger and more complex, such as a library or application"*

작은 코드 조각들을 거대하고 복잡한 어플리케이션 혹은 라이브러리로 만들어준다고 소개한다. 같은 소스코드를 다양한 환경에 맞춰 다르게 빌드하는 것을 생각하면 될 것 같다.

그러면서 아래의 여론이 형성되었다.

> 어플리케이션을 만들 때는 webpack, 라이브러리를 만들 땐 rollup으로!

rollup의 사용방식과 구성은 webpack과 비슷하다. input과 entry를 설정하고 번들링에 적용할 기능을 plugin 형태로 끼워넣으면 된다

```javascript
import commonjs from '@rollup/plugin-commonjs';
import resolve from '@rollup/plugin-node-resolve';
import { terser } from 'rollup-plugin-terser';

const production = !process.env.ROLLUP_WATCH;

export default {
  input: 'src/main.js',
  output: {
    file: 'public/bundle.js',
    format: 'iife',
    sourcemap: true,
  },
  plugins: [
    resolve(),
    commonjs(),
    production && terser(),
  ],
};
```

### Rollup VS webpack

webpack은 내부적으로 CommonJS를 사용하고, Rollup은 typescript(ES6)을 사용한다. 이로 인해 아래의 특성들이 존재한다.

- ES6 모듈 형태로 빌드가능한 Rollup

  webpack은 v5이전까진 CommonJS형태로만 번들링이 가능했다. (이후에는 ES6으로 번들링이 가능하다.)

  라이브러리는 ES6 번들 생성에 대한 수요가 강하다. ES6 환경에서는 ES6 번들이 사용되고, CommonJS환경에서는 CommonJS 번들이 사용되도록 해야 라이브러리 사용자는 더욱 최적화된 애플리케이션을 빌드할 수 있다.

- 조금 더 빠른 Rollup

  webpack은 모듈들을 함수로 감싸서 평가하는 방식을 사용하지만, Rollup은 모듈들이 호이스팅하여 한번에 평가하여 성능상 이점이 있다.

- 조금 더 가벼운 번들을 생성하는 Rollup

  `tree shaking` 은 기본적으로 ES6에서 제대로 동작한다.

  단순히 레퍼런스되지 않는 코드를 제거하는 것이 아닌, 사용되는 모듈만 AST 트리에 포함시키는 방식으로 불필요한 코드들을 제거하기 때문이다.

  Rollup은 공식 플러그인을 통해서 CommonJS 코드를 ES6으로 변환할 수 있다.

- CommonJS 코드를 ES6에서 사용가능한 Rollup

  ES6에서 CommonJS식의 require는 지원하지 않는다. webpack에서 공식적으로 ES6 혹은 CommonJS의 형태로 코드 베이스를 사용하기를 권장한다.

---

## ESBuild

Webpack, Rollup 모두 내부적으로 JavaScript를 기반으로 번들링을 한다. 따라서 JavaScript 언어가 가질 수 밖에 없는 성능상의 한계가 있다.

ESBuild는 Go로 작성되었고, JS기반의 번들러보다 10-100배까지 빠른 퍼포먼스를 보여준다. 빠른 이유에 대해서 아래와 같이 소개한다.

![](/assets/img/2025-06-08/js-bundler-next-gen-bundlers-2.png)

- 네이티브 코드 방식 사용
- 병렬처리 최적화
- 메모리 사용 최적화
- 자체 JavaScript 파서 사용

번들로로써는 필수적인 기능이 갖춰졌지만, 설정이 webpack, Rollup처럼 유연하지 못하고 안전성 관련 이슈가 있다.

구현된 필수적인 기능들

- JS, CSS, TS, JSX built-in
- Bundles ESM, CommonJS Modules
- Tree shaking, minification, and source maps
- Local server, watch mode, plugins

이슈사항

- Code Splitting and CSS
- ES5 이하의 문법을 100% 지원하지 않음

---

## Vite(비트)

ESBuild의 단점을 보완시킨 라이브러리이다.

Vue.js의 창시자인 Evan You가 만든 Frontend Build Toll이다. 2021년에 등장했지만 벌써 V6가 나온만큼 가장 활발하게 업데이트되고 있는 라이브러리이다.

Vite는 아래의 특징이 있다.

1. native ES Module 기반의 강력한 개발 서버
2. ESBuild로 파일을 통합하고 Rollup을 통해 번들링

⇒ ESBuild로 성능을 극적으로 끌어오고, Rollup을 통해 번들링의 유연성을 챙겼다.

### 특징

- 개발 서버 구동 시간이 거의 0에 가깝다.
- 모든 CommonJS 및 UMD 파일을 ESM으로 불러올 수 있도록 변환한다.
- 별도의 설정 없이 다양한 리소스를 import하는게 가능하다.
- CSS 빌드 최적화
  - Direct Import 구문에 대해 Preload하도록 함으로써, 네트워크 비용을 줄였다.

### 주의점

1. Vite는 기본적으로 ES6을 타겟으로 번들을 생성한다.
   1. → ES5 이하로 타겟을 잡으려면 polyfill을 다뤄야 한다. (공식적으로 [`@vitejs/plugin-legacy`](https://github.com/vitejs/vite/tree/main/packages/plugin-legacy) 플러그인을 제공한다.)
