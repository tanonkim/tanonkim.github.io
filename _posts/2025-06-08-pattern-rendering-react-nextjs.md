---
title: "Rendering Patterns 2 : React.js, Next.js"
date: 2025-06-08 17:10:00 +0900
categories: [design-pattern, rendering]
tags: [design-pattern, nextjs]
---

## 1. ReactJS

JavaScript를 사용하여 UI를 직접 구성하는 것에 대한 수요가 증가했다. 이를 위해 UI를 개발하기 위해 페이스북이 개발한 오픈소스 라이브러리인 React가 사용되기 시작했다. Vue, Angular, Svelte, Lit와 같은 도구들이 있지만, 여기서 렌더링 패턴을 언급하면서 살펴볼만한 가치가 있다.

개발자들은 웹 인터페이스를 위한 설계에 관련된 이야기를 많이 한다. 그리고 인터페이스의 주요 구성 요소는 버튼, 리스트, 네비게이션 등과 같은 것이다. React는 이런 요소들을 조합해 인터페이스를 표현하기 위한 최적화되고 단순한 방법을 제시한다. 또 복잡하거나, 구현하기 까다로운 인터페이스를 세 가지 주요 컨셉(Component, Props, State)을 통해 쉽게 개발할 수 있도록 돕는다.

React는 `컴포넌트`에 초점이 맞춰져 있기 때문에 웹의 디자인 시스템의 요소들을 구성하는데 적합하다. 그래서 본질적으로 React 기반으로 설계하게 되면 **모듈식**으로 구성하는 것에 대해 시너지가 발생하며, 완성된 페이지나 뷰를 보기 전에 개별 컴포넌트를 디자인하는게 가능해지므로, 각 요소의 범위와 목적에 대해 충분히 이해할 수 있다. 이 과정을 `컴포넌트`라고 한다.

### 용어

React : 페이스북이 2013년에 개발한 라이브러리

ReactDOM : DOM과 서버 렌더링을 위한 패키지

JSX : JavaScript를 확장한 구문

Redux : 중앙화된 상태 컨테이너

Hooks : React에서 클래스 없이 상태 등을 사용할 수 있는 새로운 방법

ReactNative : JavaScript를 이용하여 크로스 플랫폼 네이티브 앱을 개발할 수 있는 라이브러리

Webpack : JavaScript 모듈 번들러

CRA - Create React App : React 프로젝트를 만들 수 있는 CLI 도구

Next.js : SSR, 코드 스플리팅, 성능 최적화, 기타 동급 최고의 기능을 포함한 React 프레임워크

### JSX

JSX는 JavaScript를 확장한 구문으로 XML과 유사한 형태의 HTML템플릿을 포함하고 있다.

```javascript
// How JSX Works
<MyButton color="blue" shadowSize={2}>
	Click ME
</MyButton>

==> 

React.createElement(
	MyButton,
	{color : 'blue', shadowSize : 2},
	'Click Me'
)
```

### Component, Props, State

1. **Component**

컴포넌트는 React앱의 기본 빌딩 블록이다. 컴포넌트는 선택적으로 props를 받을 수 있고, 화면에 보여질 React 엘레먼트를 반환한다.

React 앱은 컴포넌트 안의 컴포넌트 안의 컴포넌트이다. 그래서 개발자는 페이지를 개발하는게 아닌, 컴포넌트를 개발해야 한다.

컴포는트는 UI를 독립적이며 재사용가능하게 해 준다.

- 함수형 컴포넌트

```javascript
function Badge(props) {
  return <h1>Hello, my name is {props.name}</h1>
}
```

프로퍼티가 담긴 props 객체를 인자로 받고, React 엘레먼트를 반환하므로, 유효한 함수형 컴포넌트이다.

- 클래스형 컴포넌트

```javascript
class Badge extends React.Component {
  render() {
    return <h1>Hello, my name is {this.props.name}</h1>
  }
}
```

ES6의 클래스 문법으로 정의할 수 있다.

1. **Props**

Props는 컴포넌트의 프로퍼티이며, React 컴포넌트의 내부 데이터를 가리킨다. Props는 컴포넌트의 호출에서 만들어지고, 다른 컴포넌트에게 전달된다.

- prop으로 넘어갈 값들을 미리 정해야 한다
- 컴포넌트에 전달된 prop의 값은 readOnly이다.

함수형 컴포넌트는 prop인자를 직접 사용, 클래스는 this.props의 값을 참조하는 것으로 사용하면 된다.

1. **State**

state는 컴포넌트가 마운트되고 언마운트 되기 전까지 변경될 수 있는 값들을 가지고 있는 객체이다. state는 계속 변화하기 때문에 개발자가 원하는 시점에 원하는 화면을 그려내기 위해 변화하는 데이터를 관리할 방법이 필요한데, 이를 `상태관리` 라고 한다.

React에서 State는 전역적으로 추적 가능하며, 데이터는 필요에 따라 컴포넌트 간에 공유할 수 있다.

### React에서 상태를 추가하는 방법

컴포넌트를 설계할 때 상태를 추가하는 것은 가급적 마지막에 해야 한다. Props와 이벤트만 가지고 개발하는 것이 컴포넌트를 유지하기 쉽고, 테스트하기 쉽고, 이해하기 쉽게 만든다.

Redux : 반응형 프레임워크를 위한 유명한 상태관리 시스템이며, 중앙화된 상태관리 기능을 구현할 수 있다.

### 기타 React 컨셉

1. 생명주기

   모든 컴포넌트는 마운트되고, 렌더링되며 언마운트되는 단계를 거친다. 이 사이에 발생하는 일련의 이벤트들을 컴포넌트의 생명주기라고 한다. 이런 이벤트 자체는 부분적으로 컴포넌트의 내부의 상태와 관련이 있지만, 생명주기는 조금 다르다. React는 필요에 따라 컴포넌트를 로드하거나 언로드하는 내부 코드가 있고,  컴포넌트는 각 단계에서 이 내부 코드를 사용한다.

   render() : 클래스 컴포넌트에서만 필요하며, React 내에서 가장 많이 구현된 메서드이다. 마운팅되고, 렌더링되는 과정 중에 컴포넌트의 UI 렌더링을 처리한다.

   - 컴포넌트 생성/제거

   componentDidMount() : 컴포넌트의 출력 결과가 DOM에 렌더링되었을 때 실행

   componentWillUnMount() : 컴포넌트가 언마운트되고 제거될 때 즉시 호출

   - Props, State가 업데이트 될 때

   shouldComponentUpdate() : 새로운 prop이나, state를 받아 렌더링 되기 직전에 호출

   componentDidUpdate() : 업데이트가 일어난 직후 호출, 최초 렌더링 직후에는 호출되지 않는다.

2. 고차함수 (HOC)

   React에서 컴포넌트 재사용을 위한 고급 기술이다. HOC는 기본적으로 컴포넌트를 받아 컴포넌트를 반환하는 함수를 의미한다. (함수 인자로 받거나 함수를 반환하는 함수)

   컴포넌트가 props를 UI로 변환할 때 고차 함수 컴포넌트는 특정 컴포넌트를 다른 컴포넌트로 변환한다.

3. 컨텍스트

   보통 React에서 데이터는 props로 컴포넌트 간에 전달되는데, 컨텍스트를 사용하면 중간 컴포넌트에게 불필요한 데이터를 내려줄 필요가 없게 된다.

### React Hooks

React의 상태나 생명주기 기능 중간에 개입할 수 있는 함수이다. 훅을 사용하면 클래스를 작성하지 않고도 state나 react의 기타 기능을 활용할 수 있게 해준다.

---

## 2. Next.js

Next.js는 vercel에서 개발된 하이브리드 react 앱을 위한 프레임워크이다. 복잡한 설정 파일 없이도 확장이 가능하고, 성능또한 좋은 react코드를 개발할 수 있게 해 준다. 따라서 개발자는 비즈니스 로직을 개발하는데 집중할 수 있다.

### 기본 기능

**pre-rendering**

next.js는 클라이언트 측 페이지와는 별개로 HTML 코드들을 미리 생성해둔다. 이 과정을 pre-rendering이라고 한다. next.js는 서버사이드 렌더링 혹은 정적 페이지 생성을 통한 pre-rendering을 지원한다. 하나의 앱 내에서 페이지 별로 서버사이드 렌더링 혹은 정적 페이지 생성을 선택적으로도 사용할 수 있다. (클라이언트 사이드 렌더링도 적용할 수 있다.)

**데이터 불러오기 (page router)**

next.js는 서버사이드 렌더링, 정적 페이지 생성 양쪽에서 모두 데이터를 불러올 수 있다.

1. getStaticProps : 데이터 렌더를 위해 정적 사이트 생성과 함께 사용한다.
2. getStaticPaths : 동적 라우팅 처리를 위해 정적 사이트 생성과 함께 사용한다.
3. getServerSideProps : 서버사이드 렌더링

**정적 컨텐츠 서빙**

public 폴더에 이미지와 같은 정적 컨텐츠를 넣음녀 된다. 절대 경로를 사용하면 된다.

**이미지 최적화**

Next.js는 이미지 자동 최적화 기능이 구현되어있어, 최신 포멧의 이미지를 제공할 수 있다.

```javascript
import Image from 'next/image'
<Image src="/logo.png" alt="Logo" width={500} height={500} />
```

**라우팅**

동적 라우팅 : /products/[id].ts → 동적 페이지로 링크 거는 것도 가능하다.

**코드 스플리팅**

코드 스플리팅 기능을 활용하면 클라이언트 측 성능 확보를 위해 꼭 필요한 javascript만 보낼 수 있다.

1. 라우팅 경로 기반

   Next.js에 기본으로 적용되어있으며, 사용자가 라우팅 할 때 최초로 필요로 하는 코드들만 전송한다. → 나머지 코드들은 앱 내에서 페이지를 이동할 때 추가적으로 전송한다. 이는파싱하고 컴파일해야 하는 코드량을 줄여 페이지 로드 타임을 감소시킬 수 있다.

2. 컴포넌트 별

   dynamic import()를 통해 컴포넌트 코드 스필리팅을 지원한다.
