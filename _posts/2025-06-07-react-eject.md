---
title: "React Eject"
date: 2025-06-07 17:59:00 +0900
categories: [react, react-study]
tags: [react, cra]
---

## CRA를 사용하는 이유

페이스북에서 개발한 CRA(Create-React-App) 보일러 플레이트를 사용하게 되면 간단하게 리액트 프로젝트를 시작할 수 있다.  CRA는 리액트 프로젝트를 처음 실행할 때 필요한 **여러가지 라이브러리나 웹팩의 설정 없이** 간편하게 프로젝트를 시작할 수 있다.

**참고** : 현재는 Deprecated되었다. React팀은 Vite, Next.js Remix와 같은 현대적인 툴로 전환하라는 메세지를 전했다. (연습용 테스트로 진행되었다.)

### CRA가 가지는 장점

1. 단 하나의 one build dependency를 가지므로 React 프로젝트를 구성할 때 필요한 Webpack, Babel, ESLint 등 간의 연결에 신경쓰지 않아도 된다.
2. Webpack, Babel, ESLink는 처음 프로젝트를 구성할 때 반드시 필요한 패키지이다. 이를 처음 설정할 때 시간이 상당히 소요될 수 있는데, CRA 프로젝트는 필요한 필수적인 설정을 대신 해준다.
3. CRA는 Autoprefixer를 지원한다. → 일반적인 CSS 코드 생성을 하게 되면 자동으로 -webkit-, -ms-등을 자동으로 적용해준다.

### Eject를 사용하는 이유

Eject는 one build dependency를 가진 프로젝트를 custom하기 위한 방법이다. CRA 프로젝트에서 다음의 명령어를 수행하기만 하면 된다.

```
yarn eject
```

![](/assets/img/2025-06-07/react-eject-1.png)

숨겨져 있던 모든 설정(All, configurations, webpack 설정, babel 설정)과 패키지들이 가지는 의존성을 볼 수 있게 된다.

![](/assets/img/2025-06-07/react-eject-2.png)

eject 이전 사진과 이후 사진을 보면 여러가지 숨겨져 있던 설정들과 패키지들이 보이는 것을 확인할 수 있다.

### Eject를 쓰면 안되는 이유

eject는 프로젝트를 자유롭게 customizing한다는 점에서 매력적으로 보이나, CRA를 쓰면서 갖게 되는 이점을 포기하게 된다.

1. CRA의 모든 configuration을 직접 유지보수해야 한다. Webpack, Babel 등 이외에도 많은 설정을 해야 한다.
2. One Build Dependency의 장점을 잃게 된다. 작업 도중 하나의 패키지가 필요해서 설치하거나, 삭제할 때 항상 다른 패키지들과의 의존성을 신경써야 한다.

그럼에도 불구하고 Eject를 하고 싶다면, 몇가지 고려해봐야 할 것이 있다.

1. 얼마나 많은 설정을 기존 CRA 설정에 추가하거나 뺄 것인가?
2. 추가하고 싶은 설정의 중요성이 Eject를 하기 이전에 가지는 장점보다 큰가?

### 참고(Webpack과 Babel의 역할)

**Babel**

React에서 webpack과 babel이 필요한 이유는 JSX과 ES6+ 코드가 브라우저에서 바로 이해되지 않거나, 애플리케이션을 효율적으로 관리/번들링하려면 빌드 프로세스가 필요하기 때문이다.

- Babel : JS 변환

```typescript
const element = <h1>Hello, world!</h1>;
```

브라우저는 JSX를 이해하지 못한다. Babel이 JSX를 React.creacteElement() 호출로 변환한다.

예) babel이 변환

```typescript
const element = React.createElement("h1", null, "Hello, world!");
```

뿐만 아니라 ES6+도 Babel이 ES5로 변환해줘서 구형 브라우저에서 동작할 수. ㅣㅆ게 만들어준다.

**Webpack**

React App은 여러개의 JS파일, CSS, 이미지로 구성된다.

웹 브라우저는 모듈 시스템을 완벽하게 지원하지 않기 때문에, 모듈을 코드 스플리팅된 여러 파일/또는 하나의 파일로 번들링해줘야 한다. Webpack은 모듈을 하나의 파일로 묶고, 의존성을 해결하고, 코드 스플리팅까지 처리한다.

Hot Module Replacement(HMR)도 Webpack이 제공해서 개발 중 변경된 코드를 실시간으로 반영한다. → 개발 서버 제공

**CRA(Vite) 시대 이후에는**

CRA에서는 내부적으로 Babel과 Webpack 설정이 숨겨져 있고, Vite로 넘어오면서 Rollup 기반 번들러가 사용되지만 Babel처럼 JSX변환 등이 여전히 필요하다.
