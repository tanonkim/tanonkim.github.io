---
title: "React Fiber 아키텍쳐 (1)"
date: 2025-08-23 17:45:00 +0900
categories: [react, react-study]
tags: [react, fiber]
---

React는 16 버전부터 파이버라고 불리는 새로운 코어 아키텍쳐를 채택했다. 기존의 스택 기반 알고리즘을 완전히 새롭게 작성한 것이고, 19버전까지 지속적으로 업데이트 및 리팩터링 되고 있다.

Version : React 18.2

## 0. 복습

### **JSX**

```javascript
const root = ReactDOM.createRoot(document.getElementById('root'));
root.render(<h1>Hello, world!</h1>);  
```

JSX를 사용하고 있으므로 위 코드는 JavaScript엔진이 이해할 수 있는 순수 JavaScript 코드로 변환될 것이다.

### **React Element**

```javascript
const element = <h1>Hello, world</h1>;
```

React Element는 일반 객체이며, 쉽게 생성할 수 있다. ReactDOM은 React 엘리먼트와 일치하도록 DOM을 업데이트한다.

### 결론

JSX로 선언된 표현식은 React 엘리먼트 객체로 치환된다.

React 엘리먼트는 일반 객체이다. (클래스 인스턴스가 아니다)

→ JSX 표현식이 치환된 객체이므로 태그이름, attribute 및 값, 자식 등의 정보가 있을 것이다.

---

### DOM에 엘리먼트 렌더링하기

예) React 엘리먼트 렌더링

```javascript
const root = ReactDOM.createRoot(
	document.getElementById('root');
)
const element = <h1>Hello World</h1>
root.render(element)
```

- 네이비트 DOM에 React 엘리먼트를 렌더링하기 위해선 루트를 생성해야 한다.
  - React 엘리먼트는 root를 통해 렌더링해야 한다. (Native DOM에 렌더링할 수 없다.)

### 렌더링된 엘리먼트 업데이트 하기

```javascript
const root = ReactDOM.createRoot(  
  document.getElementById('root')
);

function tick() {  
  const element = (
    <div>
      <h1>Hello, world!</h1>
      <h2>It is {new Date().toLocaleTimeString()}.</h2>
    </div>
  );
  root.render(element);
}

setInterval(tick, 1000);  
```

React 엘리먼트를 업데이트를 하려면 JSX 표현식을 통해 새로운 React 엘리먼트를 생성하고, 루트를 통해 렌더링해야 한다.

### 변경된 부분만 업데이트 하기

React DOM은 업데이트 시 변경된 부분만 native DOM에 반영하도록 되어있다. React 내부 알고리즘을 통해 변경된 부분만 반영하면서 DOM 트리 변경을 줄여 퍼포먼스 향상을 시키는 것이다.

**리스트와 키**

React는 형제의 태그 이름이 동일하다면 key attribute를 선언하는것을 권장한다. React 엘리먼트는 태그 이름이 동일한 형제 중 key가 동일할 때 같은 엘리먼트라고 판단한다. (같은 엘리먼트는 렌더링하지 않음)

---

### 재조정_Reconciliation

n개의 엘리먼트가 있는 트리를 다른 트리로 변환하는 알고리즘의 복잡도는 최소 O(n^3)로 알려져있다. (DOM + JS + CSSOM  과정부터 RenderTree → Style → Layout →  Paint → Composite → Display)

최소 O(n^3)은 성능상 안좋으므로 React는 아래의 두 조건을 전제로 복잡도가 O(n)인 휴리스틱 알고리즘을 재조정에 사용한다.

1. 서로 다른 타입의 엘리먼트는 서로 다른 트리를 만든다. → 이전과 다른 타입의 React 엘리먼트로 교체되었다면 더 비교하지 않고 트리를 전체 교체한다.
2. key prop을 통해 여러 렌더링 사이에서 어떤 자식 엘리먼트가 변경되지 않아야 할지 표시가 가능하다. → key가 동일한 React 엘리먼트는 이전과 동일한 엘리먼트로 취급한다.

### Ref/DOM

Ref는 직접 DOM을 조작하는 명령형 방식이다. (React가 제어가능한 부분이 아님)

Ref의 바람직한 사례는 다음과 같다

- 포커스, 텍스트 선택 영역, 미디어의 재생을 관리할 때
- 애니메이션을 직접 실행시킬 때
- Third Party DOM 라이브러리를 React와 함께 사용할 때

세 가지 모두 React에서 컨트롤이 가능한 영역인 태그이름, attribute, 자식이 아니라 native DOM API 등을 직접 컨트롤해야 하는 경우이다. 즉 React가 제어가능한 부분은 선언형 API를 통해 조작하라는 의미이다.

React는 JSX 표현식이 치환된 React 엘리먼트의 props, states를 통해 React DOM을 갱신하므로, Ref를 통해 native DOM을 직접 조작하면 변경 사항을 직접 React가 유지할 수 없을 것이라고 추정할 수 있다.

요약 :  Ref를 통한 객체 접근은 React의 선언형 업데이트 흐름 밖에서 일어나는 일이다.

**Ref와 관련하여 조금 더 자세히**

React는 선언형을 지향한다. 결과가 어떻게 보일지 기술한다. (어떻게 동작할지 직접 지시보다는)

```javascript
// 선언형 (React가 이 상태에 맞게 UI를 자동으로 갱신)
const [isOpen, setIsOpen] = useState(false);
return <Modal open={isOpen} />
```

```javascript
// 명령형 (직접 DOM을 조작)
const modalRef = useRef();
modalRef.current.open();
```

Ref의 역할 : React 외부의 DOM에 직접 접근할 수 있게 한다.

### React가 ref의 직접 조작을 경계하는 이유

> React가 제어 가능한 부분은 선언형 API로 조작하라

React의 UI 갱신

1. state or props가 변경
2. 이를 기반으로 Virtual DOM을 생성
3. Virtual DOM과 diffing해서 실제 DOM 업데이트

그런데 Ref 직접 조작 시 이 과정과 무관한 변경이 생긴다.

```javascript
const ref = useRef();
useEffect(() => {
  ref.current.style.background = 'red'; // 직접 DOM 변경
}, []);
```

React는 내가 background를 red로 만든게 아니니까 유지할 필요 없어로 생각하고 원래의 값으로 변경시킬 수 있음 ⇒ Virtual DOM과 Native DOM이 불일치하게 됨

### Virtual DOM과 Internals

재조정 : Virtual DOM은 UI의 가상적인 표현을 메모리에 저장하고 React DOM과 같은 라이브러리에 의해 실제 DOM과 동기화하는 프로그래밍 개념이다.

Virtual DOM은 기술보다는 패턴에 가깝다. 보통은 사용자의 인터페이스를 나타내는 객체이기 때문에 React Element와 연관이 된다. 그러나 React 컴포넌트 트리에 대한 추가 정보를 포함하기 위해 `Fiber` 라는 내부 객체를 사용한다.

## 1. React가 하는 일

예)

```javascript
const FooComponent = (): JSX.Element => {  
    return <div>
			        <p>Title</p>
			        <button>Click me!</button>
				    </div>
}

ReactDOM.createRoot(document.querySelector('#container')).render(<FooComponent />); 
```

결과는 아래와 같다

```javascript
<div id="container">  
    <div>
        <p>Title</p>
        <button>Click me!</button>
    </div>
</div>  
```

JSX 문법을 사용하여 엘리먼트 마크업과 완전히 동일한 내용이 루트 컨테이너에 삽입되었다.

예) state를 사용한 변경

```javascript
function ToggleName(): JSX.Element {  
    const [flag, setFlag] = React.useState(false);
    return (
        <div>
            <p>Name : {flag ? "Alice" : "Samuel"}</p>
            <button onClick={() => setFlag(!flag)}>Click to switch</button>
        </div>
    );
}
```

결과는 아래와 같다

```javascript
<div id="container">
    <div>
        <p>Name : Samuel</p>
        <button>Click to switch</button>
    </div>
</div>
```

그리고 버튼을 클릭하면 Samuel이 Alice로 변경된다.

즉 React는 JSX로 작성된 React 엘리먼트를 HTML엘리먼트로 변환하여 DOM에 렌더링하여 변경이 발생했을 때 정확히 변경된 부분만 파악하여 적용하는 방식을 취하고 있다.

⇒ Attritube, value 등의 변경은 핀 포인트로 적용하고, 엘리먼트 타입의 변경은 하위 트리 전체를 변경하는 것이다.

## 2. 사전 지식

### UI 업데이트 주기

JavaScript엔진은 단일 쓰레드이다. 하나의 쓰레드에서 사용자 상호작용, DOM이벤트, Ajax 등 모든 로직을 순차적으로 처리한다. 그래서 코드를 실행하는 중에는 브라우저는 UI 업데이트가 불가능하다.

현재 모니터의 주사율이 초당 60회 화면을 갱신을 한다. 다음 화면 갱신까지 약 16ms가 걸린다는 의미이다. 작성된 코드가 16ms 이상의 시간을 소비하면서 실행된다면 UI 업데이트는 모니터의 주사율을 따라갈 수 없다. → 화면이 충분히 빠르게 갱신되지 못하고 끊기는 현상이 발생한다.

이를 극복하기 위해서 개발자가 직접 프론트엔드에서 코드의 동시성을 적절히 조절하는게 필요했으나, React Fiber 구조에서는 UI 갱신 작업을 작은 단위로 나누어 내부적으로 스케쥴링한다.

→ 대규모 UI 갱신에서 **16ms를 초과하지 않도록** 작업되었다.

- 기존 스택 재조정 알고리즘
- 새로운 재조정 알고리즘

  ![](/assets/img/2025-08-23/react-fiber-architecture-1-1.gif)

### 동시성 스케쥴링

React 개발팀은 기존의 알고리즘으로 시간이 갈수록 커지고 복잡해지는 앱 컴포넌트를 응답성을 읽지 않으면서 빠르게 업데이트하는게 어렵다고 판단했다. Fiber가 기존의 스택 재조정자와 근본적으로 다른 점이 `동시성`이다.

DOM 업데이트, 렌더링 로직을 작업 단위로 구분하고, 이를 비동기로 실행하여 최대 실행시간이 16ms이 넘지 않도록 제어한다.

Fiber는 작업을 chunk로 분리하여 실행시간을 관리하는 것이 아닌, **작업의 유형**에 따라 우선순위를 부여하고 기존의 작업보다 더 높은 우선순위가 높은 작업이 유입될 경우 일시 중단하고 유입된 작업을 다시 처리 후 돌아온다. ⇒ 기존 스택 알고리즘의 문제 극복 : 중단하거나 중요한 작업에 대해 먼저 수행할 수 없다는 단점 극복

## 3. React의 동작 단계

1. Render 단계 : JSX 선언 또는 React.createElement()를 통해 일반 객체인 React 엘리먼트를 생성한다.
2. Reconcile 단계 : 재조정 단계 _ 기존의 렌더링된 실제 DOM트리와 새로 렌더링할 React 엘리먼트와 비교하여 변경점을 적용한다.
3. Commit 단계 : 새로운 DOM 엘리먼트를 브라우저 뷰에 `커밋`한다.
4. Update 단계 : props, state 변경 시 해당 컴포넌트와 하위 컴포넌트에 대해 위 과정을 반복한다.

### Fiber Node의 구성요소

```typescript
function FiberNode(
  tag: WorkTag,
  pendingProps: mixed,
  key: null | string,
  mode: TypeOfMode,) {
  // Instance
  this.tag = tag;
  this.key = key;
  this.elementType = null;
  this.type = null;
  this.stateNode = null;

  // Fiber
  this.return = null;
  this.child = null;
  this.sibling = null;
  this.index = 0;

  this.ref = null;
  this.refCleanup = null;

  this.pendingProps = pendingProps;
  this.memoizedProps = null;
  this.updateQueue = null;
  this.memoizedState = null;
  this.dependencies = null;

  this.mode = mode;

  // Effects
  this.flags = NoFlags;
  this.subtreeFlags = NoFlags;
  this.deletions = null;

  this.lanes = NoLanes;
  this.childLanes = NoLanes;

  this.alternate = null;
  ...
}
```

- Instance
  - tag
    - react fiber는 1:1 관계를 가진다. DOM/component든 무조건 1:1 관계를 지니며, this.tag에 저장된다. 해당 타입에서 funcitonal component, class component 등 우리가 아는 여러 유형들의 노드들에 대해 번호로 관리한다
  - type
  - elementType
    - 어떤 요소인지 타입 구분
  - key
    - 컴포넌트를 식별하는데 사용되는 고유값. 한 배열 안에 같은 tag를 가진 여러 요소들이 있다면 재조정자는 순서와 일부 요소 변경에 대해 같은 태그를 가지고 있기에 비교하기가 어려운데, key를 통해 컴포넌트를 구별하고 불필요한 렌더링 방지한다.
  - stateNode
    - 해당 객체에 대한 참조를 가지고 있으며, 이 속성을 사용하여 파이버와 관련된 상태들에 접근이 가능하다.
- Fiber
  - child, sibling, return
    - Fiber를 기반으로 트리를 만드는 과정에서 필요한 관계가 이 세 가지 속성에 의해 만들어진다. children이 아닌 child 1개의 자식을 가지고 있고, 자식의 첫 번째 요소를 가리킨다. ⇒ 부모에 여러 자식이 있다면 첫 요소만 자식 요소이고, 나머지는 자식 요소들의 형제 요소이다.
    - return의 경우 부모 항목에 대한 참조이다.
  - index
  - ref, refCleanup
    - 특정 컴포넌트나, DOM 요소에 접근할 수 있도록 DOM/인스턴스에 대한 참조
    - 컴포넌트가 언마운트 될 때 메모리 누수를 방지하기 위해 refCleanup을 통해 불필요한 참조를 제거
  - mode
    - FiberNode가 렌더링되는 방식을 정의
    - Concurrent Mode, StrictMode와 같은 동작을 결정
  - pendingProps
    - 아직 작업에 대해 처리하지 못한 props를 저장
  - momoizedState
    - pendingProps가 렌더링 완료되면 memoizedProps로 옮겨가며, 현재 가지고 있는 props를 표현한다.
  - updateQueue
    - 상태 업데이트, 콜백함수, DOM 업데이트 등 필요한 작업을 담아두는 큐
  - alternate
    - 리액트에 존재하는 두 가지 트리 중에서 자신을 포함하지 않은 반대편 트리
- Flags
  - flags
    - 노드에서 수행하는 작업을 나타내는 비트 플래그
    - placement(새로운 DOM 노드 삽입), update(기존 DOM 노드 업데이트), deletion(DOM 노드 삭제) 등의 플래그를 통해 리렌더링 여부 결정
  - subtreeFlags
    - 하위 트리 내에서 발생해야 하는 작업을 나타냄
  - deletions
    - 삭제해야 하는 자식 노드들의 목록 관리
    - 삭제해야 하는 노드들을 미리 배열에 추가하고 일괄 처리
  - lanes
    - Concurrent Mode에서 작업의 우선순위를 추적
  - childLanes
    - 하위 트리에서 실행할 작업들의 우선순위를 추적

### 0. 전체 요약

아래는 React 컴포넌트가 어떻게 DOM에 렌더링되는지 내부 코드를 보고, 파이버 재조정자의 구체적인 동작이다

### 가상 DOM (Fiber)를 이용한 리액트 렌더링 \<전체 흐름 요약\>

리액트의 virtual DOM 개념은 렌더링 되어있는 상태의 DOM 요소에 가지고 있는 rootFiber의 정보와 추가적으로 하나의 rootFiber를 구성한다. 여기서 root는 fiberRoot과 rootFiber 두 가지의 root가 있다.

- fiberRoot

  createRoot를 통해 만드는 가장 기본적인 html 템플릿의 루트로, 안쪽 요소의 렌더링으로 계속해서 이루어지는 SPA 특성 상 고정적인 head, body를 가진 root를 두고 **안쪽 요소만 갈아끼우기 위해 rootFiber를 따로 두어** 실질적인 렌더링이 반복적으로 이루어지는 root를 구분했다.

root에 대한 fiberRoot를 구성하는 과정에서 rootFiber를 하나 더 구성하는 것이 virtual DOM의 핵심이다. 기존 렌더링되어있는 상태의 DOM 정보를 가진 fiber와 새롭게 변동 사항이 있을 경우 이를 반영한 버전의 fiber를 따로 가지고 있다. → 상태가 변경되었는 경우, 리액트는 재조정 과정을 통해 기존의 렌더링된 DOM 상태를 가진 fiber와 변경되어야 할 fiber를 비교하여 업데이트가 이루어져야 하는 부분을 찾고 반영한다.

![](/assets/img/2025-08-23/react-fiber-architecture-1-2.png)

[`Reconcile 단계`]

위 사진처럼 rootFiber에서부터 재조정과정을 시작한다. fiber는 재조정 과정에서 **performUnitOfWork** 함수를 실행하는데, beginwork와 completework 함수가 포함되어있다. 따라서 안쪽 fiber로 계속 진입하면서 beginwork가 실행되고, 끝까지 갔다가 돌아오는 과정에서 return을 받으면 completework를 실행하며 해당 fiber에 대한 작업을 종료한다.

- 상태 변경이 이루어진 fiber의 위치

props/state가 변경 될 때 리액트가 재조정 과정을 통해 현재 렌더링 된 DOM 상태를 가진 fiber와 상태 변경이 이루어진 fiber를 비교하여 변경이 이루어져 업데이트가 이루어져야 하는 부분을 찾고 반영한다.

→ 상태 변경이 이루어진 fiber의 할당 위치는 현재 렌더링되어있는 DOM트리에 있다. 즉 fiberRootNode의 current를 가리키고 있는 **rootFiber의 alternate 프로퍼티**이다.

[`Update 단계`]

rootFiber는 current가 가지키고 있는 rootFiber와는 별개로 현재 렌더링된 rootFiber의 alternate props로 다른 rootFiber를 가지고 있다. → 이 두 fiber는 alternate를 통해 서로에 대해 접근하며 변동사항을 반영하고 update된 DOM tree를 만들어낸다. 이렇게 만들어진 DOM tree는 workInProgress트리라고 한다.

*workInProgress 트리는 current가 가지고 있지 않은 alternate의 rootFiber에서 만들어진다.

![](/assets/img/2025-08-23/react-fiber-architecture-1-3.png)

dfs 방식을 활용하여 새롭게 변동사항이 반영된 fiber와 기존의 fiber를 비교한 뒤, 바뀐 부분을 찾아 반영하고, 마지막에 DOM 요소를 리턴하게 되면 새롭게 바꾸어야 할 부분을 반영한 DOM 엘레먼트를 가지게 된다.

요약 : 변경사항이 모두 반영된 rootFiber를 DOM Element가 모두 만들어지고 나면, fiber Root 즉 createRoot로 만든 root 컨테이너 파이버의 current가 alternate의 rootFiber를 가리키게 한다. 해당 파이버는 새롭게 반영, DOM element 생성이 완료된 fiber이다. current의 포인터만을 alternate를 통해 반대편의 만들어진 rootFiber를 가리키도록 하면서 대기 없이 업데이트가 반영된 DOM 트리를 갈아끼운다.

### **1. Render 단계**

### 엔트리 포인트 생성(Render 단계 이전)

1. 간단한 형태의 React 컴포넌트

```typescript
import React from "react";

export function SimpleComponent(): JSX.Element {
  const [name, setName] = React.useState<string>("Alice");
  return (
    <div>
      <h1>Hello react!</h1>
      <section>
        <p>{`Name : ${name}`}</p>
        <button
          onClick={(e) => setName(name === "Samuel" ? "Alice" : "Samuel")}
        >
          Click me
        </button>
      </section>
    </div>
  );
}
```

```typescript
window.onload = () => {
  const root = ReactDOM.createRoot(document.querySelector("#container")!);

  root.render(<SimpleComponent />);
};
```

**1.1 루트 생성**

React는 SPA(Single-Page-Application)이기 때문에 여러 페이지를 전환하는 것이 아닌, 하나의 페이지에서 모든 서비스 로직이 이루어지는 형태이며, 페이지 또한 새롭게 페이지 전환을 하는 방식이 아니다 보니까 하나의 `Index.html`만 두고 안쪽 요소만 바꾸어 가면서 렌더링을 하는 방식을 사용한다.

따라서 createRoot()는 몇 가지 플래그를 설정한 후 `createFiberRoot()`를 호출하여 고유 FiverRootNode를 생성한다.

**[코드) createFiberRoot 함수](https://github.com/facebook/react/blob/v18.2.0/packages/react-reconciler/src/ReactFiberRoot.new.js#L134)**

```javascript
export function createFiberRoot(
  containerInfo: any,
  tag: RootTag,
  hydrate: boolean,
  initialChildren: ReactNodeList,
  hydrationCallbacks: null | SuspenseHydrationCallbacks,
  isStrictMode: boolean,
  concurrentUpdatesByDefaultOverride: null | boolean,
  identifierPrefix: string,
  onRecoverableError: null | ((error: mixed) => void),
  transitionCallbacks: null | TransitionTracingCallbacks,
): FiberRoot {
  const root: FiberRoot = (new FiberRootNode(
    containerInfo,
    tag,
    hydrate,
    identifierPrefix,
    onRecoverableError,
  ): any);
  if (enableSuspenseCallback) {
    root.hydrationCallbacks = hydrationCallbacks;
  }

  if (enableTransitionTracing) {
    root.transitionCallbacks = transitionCallbacks;
  }

  // Cyclic construction. This cheats the type system right now because
  // stateNode is any.
  const uninitializedFiber = createHostRootFiber(
    tag,
    isStrictMode,
    concurrentUpdatesByDefaultOverride,
  );
  root.current = uninitializedFiber;
  uninitializedFiber.stateNode = root;
```

FiberRootNode와는 별개로 HostRoot 타입의 일반 FiverNode를 하나 더 생성해 root.current에 할당하고, root를 uninitializedFiber.stateNode에 할당하여 순환참조가 가능하도록 한다. 그리고 root.current를 DOM 엘리먼트에 React 루트로 저장한다.

![](/assets/img/2025-08-23/react-fiber-architecture-1-4.png)

---

```typescript
// JSX
root.render(<SimpleComponent />);
```

React public API의 ReactRootDOM.render()는 JSX 등으로 작성된 코드가 실제 DOM으로 커밋되기까지 모든 과정을 의미한다.

```typescript
// Transpiled JS
root.render(React.createElement(SimpleComponent, null));
```

React에서 render단계는 JSX 또는 React.createElement로 작성된 코드를 React 엘리먼트로 변경하는 작업만을 의미한다. React 엘리먼트는 클래스가 아닌 **일반 객체**이다. 사용자가 작성한 컴포넌트 또는 엘리먼트 타입과 attribute, 자식에 관한 정보를 담고 있다.

**1.3 React Element**

React 엘리먼트를 실질적으로 생성하는 ReactElement는 매우 간단한 형태의 일반 객체이다.

예) Transpile된 형태

```typescript
// Transpiled <SimpleComponent />
export function SimpleComp() {
  const [name, setName] = React.useState("Alice");
  return React.createElement(
    "div",
    null,
    React.createElement("h1", null, "Hello react!"),
    React.createElement(
      "section",
      null,
      React.createElement("p", null, `Name : ${name}`),
      React.createElement(
        "button",
        { onClick: (e) => setName(name === "Samuel" ? "Alice" : "Samuel") },
        "Click me"
      )
    )
  );
}
```

```typescript
// Transpiled <SimpleComponent />
export function SimpleComp() {
  const [name, setName] = React.useState("Alice");
  return React.createElement(
    "div",
    null,
    React.createElement("h1", null, "Hello react!"),
    React.createElement(
      "section",
      null,
      React.createElement("p", null, `Name : ${name}`),
      React.createElement(
        "button",
        { onClick: (e) => setName(name === "Samuel" ? "Alice" : "Samuel") },
        "Click me"
      )
    )
  );
}
```

- `<SimpleComponent />` **함수**가 반환하는 React 엘리먼트의 값

  ```typescript
    const el = SimpleComponent();
    console.log(el);
  ```

![](/assets/img/2025-08-23/react-fiber-architecture-1-5.png)

React.createElement()을 반복 호출하여 props.children에 하위 React 엘리먼트가 있다.

- `<SimpleComponent />` 가 반환하는 React 엘리먼트의 값

  ```javascript
    console.log(<SimpleComponent />);
  ```

![](/assets/img/2025-08-23/react-fiber-architecture-1-6.png)

`<SimpleComponent />` 가 반환하는 React 엘리먼트을 보면 props에 children 필드가 없다. 위에서 보았듯 React.createElement() 내에서는 하위 컴포넌트의 render함수를 호출하지 않기 때문이다.

Render 단계에 createElement()가 호출된다고 했지만, 실제로는 render함수가 Render단계에서 독릭접으로 끝까지 실행되는 것이 아니다.

`SimpleComponent()`는 모든 native 엘리먼트 태그가 인라인으로 선언되어 반환 시 모든 하위 태그가 React 엘리먼트로 변환되었지만, `<SimpleComponent />`는 SimpleComponent 함수 컴포넌트를 렌더링할 것임을 나타내는 React 엘리먼트를 반환한다.

`SimpleComponent()`는 Reconcile 단계에 실행되며, 내부 동작을 알아야 `SimpleComponent()`의 실행 타이밍을 알 수 있다.

### **2. Reconcile 단계**

`root.render(<SimpleComponent />)`를 실행하면 SimpleComponent React 엘리먼트만 생성 후 Reconcile 단계에 진입한다. ⇒ **FiberRoot는 uninitialized된 상태이다.**

재조정자는 FiberNode를 하나의 작업 단위(unitOfWork)로 취급한다. FiberNode는 자체로 렌더링에 필요한 정보를 담고 있는 객체이자 재조정 작업 단위인 것이다.

```typescript
function performUnitOfWork(unitOfWork: Fiber): void {
  // The current, flushed, state of this fiber is the alternate. Ideally
  // nothing should rely on this, but relying on it here means that we don't
  // need an additional field on the work in progress.
  const current = unitOfWork.alternate;
  setCurrentDebugFiberInDEV(unitOfWork);

  let next;
  if (enableProfilerTimer && (unitOfWork.mode & ProfileMode) !== NoMode) {
    startProfilerTimer(unitOfWork);
    next = beginWork(current, unitOfWork, subtreeRenderLanes); 
	   // current : 현재 렌더링 된 DOM의 반대 DOM 트리의 파이버 (곧 현재가 될거라서 current)
    // unitOfWork : 변경사항이 적용된 파이버 
    stopProfilerTimerIfRunningAndRecordDelta(unitOfWork, true);
  } else {
    next = beginWork(current, unitOfWork, subtreeRenderLanes);
  }

  resetCurrentDebugFiberInDEV();
  unitOfWork.memoizedProps = unitOfWork.pendingProps;
  if (next === null) {
    // If this doesn't spawn new work, complete the current work.
    completeUnitOfWork(unitOfWork);
  } else {
    workInProgress = next;
  }

  ReactCurrentOwner.current = null;
}
```

Reconcile 단계는 beginWork 단계와 completeUnitOfWork 단계로 나뉜다.

- **아래의 함수가 React와 파이버 재조정자의 핵심이다.**

```javascript
  const current = unitOfWork.alternate;
```

alternate를 current로 지정함은 무슨 의미일까? Fiber Node의 alternate는 순환참조하고 있는 fiber에서 반대편의 DOM을 가리킨다 (위의 사진 참조). 따라서 alternate를 할당한 이유는 현재 DOM이 아닌 반대편의 변경사항을 반영한 DOM을 만든 다음에 current로 교체하여 새로운 current로 만들기 위해 **현재 작업할 Fiber 노드의 이전 작업 상태를 가져오는 것이다.**

**1.4.1 첫 unitOfWork**

```typescript
function beginWork(
  current: Fiber | null,
  workInProgress: Fiber,
  renderLanes: Lanes,
): Fiber | null {
...
    switch (workInProgress.tag) {
    ...
    case HostRoot:
      return updateHostRoot(current, workInProgress, renderLanes);
...
}
```

React 내부에서 Root FiberNode를 HostRoot 컴포넌트라고 부른다. 해당 코드르 보면 beginWork와 completeUnitOfWork함수가 반복적으로 실행되는 구조이다.

```typescript
    next = beginWork(current, unitOfWork, subtreeRenderLanes);
    // current : 현재 렌더링 된 DOM의 반대 DOM 트리의 파이버 (곧 현재가 될거라서 current)
    // unitOfWork : 변경사항이 적용된 파이버
```

beginWork는 fiber와 workInProgress라는 다른 fiber를 받는데, current와 unitOfWork를 넣어주어 두 Fiber를 비교하고 변경사항을 반영한 DOM Element를 받는다.

- `beginWork`는 Fiber 트리의 **하위로 내려가면서** 자식 노드부터 작업을 처리
- 자식 노드의 작업이 끝나면 그 Fiber에서 다시 `completeUnitOfWork`가 실행되어, 작업이 상위로 완료되며 반환
- 이 과정에서 **재귀적 호출**이나 **순환적 처리**를 통해, 트리의 모든 노드에 대한 작업이 차례차례 처리

**`*current 파이버트리, workInProgress 파이버 트리`**

react는 current라는 파이버트리와 workInProgress라는 파이버 트리 두 가지 트리를 두고 계속 스위칭해가며 한쪽에는 새롭게 변동사항이 반영된 workInProgress 트리를 만들고, 그 동안은 current 트리를 보여주다가 workInProgress의 DOM이 모두 완성이 되면 이를 current로 바꾸기 때문에 workInProgress Tree와 current Tree에 대해 인지하는게 중요하다

**코드) HostRoot 컴포넌트 내부를 업데이트하는 함수**

```typescript
function updateHostRoot(current, workInProgress, renderLanes) {
  pushHostRootContext(workInProgress);

  if (current === null) {
    throw new Error('Should have a current fiber. This is a bug in React.');
  }

  const nextProps = workInProgress.pendingProps;
  const prevState = workInProgress.memoizedState;
  const prevChildren = prevState.element;
  cloneUpdateQueue(current, workInProgress); // 배치처리
  processUpdateQueue(workInProgress, nextProps, null, renderLanes); // 배치처리
...
}
```

최초 렌더링이므로 prevState, prevChildren은 비어있다. props도 없으므로 nextProps도 비어있다. HostRoot가 업데이트 해야 할 것은 유일한 자식인 `<SimpleComponent />` 이다.

update 과정에서 변동사항을 가진 workInProgress 트리에서 필요한 pendingProps와 memoizedState를 가져오고 업데이트를 진행한다. 여기서 `updateQueue`가 발생하는 이유는 업데이트가 필요한 여러 상태가 존재하는 경우 배치 처리하는 것이 효율적이기 때문이다.

코드) processUpdateQueue()

```typescript
export function processUpdateQueue<State>(
  workInProgress: Fiber,
  props: any,
  instance: any,
  renderLanes: Lanes,
): void {
  // This is always non-null on a ClassComponent or HostRoot
  const queue: UpdateQueue<State> = (workInProgress.updateQueue: any);

  hasForceUpdate = false;
```

workInProgress.updateQueue.shared.pending.payload에 위와같이 SimpleComp 컴포넌트가 element 필드로 있는 것을 알 수 있다.

```typescript
function processUpdateQueue(){
...
    // [[Link]](https://github.com/facebook/react/blob/v18.2.0/packages/react-reconciler/src/ReactFiberClassUpdateQueue.new.js#L588)
    newState = getStateFromUpdate(
              workInProgress,
              queue,
              update,
              newState,
              props,
              instance,
                        );
    ...
    // [Link]
    workInProgress.memoizedState = newState;
...
}
```

그리고 getStateFromUpdate()에서 newState를 반환하면 안에 element 필드에 SimpleComponent가 있으므로 workInProgress.memoizedState에 할당한다.

재조정 과정은 `reconcileChildren` -> `reconcileChildFibers` -> `reconcileSingleElement`이라는 일련의 과정을 통해 모든 Fiber에 대해 비교하고 재조정하는 과정을 거친다.

루트는 이미 hydrated 상태이므로 reconcileChildren()에 진입한다.

**코드**

```typescript
// [[Link]](https://github.com/facebook/react/blob/v18.2.0/packages/react-reconciler/src/ReactFiberBeginWork.new.js#L288)
export function reconcileChildren(
  current: Fiber | null,
  workInProgress: Fiber,
  nextChildren: any,
  renderLanes: Lanes,
) {
  if (current === null) {
    // If this is a fresh new component that hasn't been rendered yet, we
    // won't update its child set by applying minimal side-effects. Instead,
    // we will add them all to the child before it gets rendered. That means
    // we can optimize this reconciliation pass by not tracking side-effects.
    workInProgress.child = mountChildFibers(
      workInProgress,
      null,
      nextChildren,
      renderLanes,
    );
  } else {
    // If the current child is the same as the work in progress, it means that
    // we haven't yet started any work on these children. Therefore, we use
    // the clone algorithm to create a copy of all the current children.

    // If we had any progressed work already, that is invalid at this point so
    // let's throw it out.
    workInProgress.child = reconcileChildFibers(
      workInProgress,
      current.child,
      nextChildren,
      renderLanes,
    );
  }
}
```

HostRoot는 Current가 존재하므로 reconcileChildFibers()에 진입한다.

**코드**

```javascript
// [[Link]](https://github.com/facebook/react/blob/v18.2.0/packages/react-reconciler/src/ReactChildFiber.new.js#L1245-L1279)
// This API will tag the children with the side-effect of the reconciliation
// itself. They will be added to the side-effect list as we pass through the
// children and the parent.
function reconcileChildFibers(  
  returnFiber: Fiber,
  currentFirstChild: Fiber | null,
  newChild: any,
  lanes: Lanes,
): Fiber | null {
....
    // [[Link]](https://github.com/facebook/react/blob/v18.2.0/packages/react-reconciler/src/ReactChildFiber.new.js#L1269)
  // Handle object types
  if (typeof newChild === 'object' && newChild !== null) {
    switch (newChild.$$typeof) {
      case REACT_ELEMENT_TYPE:
        return placeSingleChild(
          reconcileSingleElement(
            returnFiber,
            currentFirstChild,
            newChild,
            lanes,
          ),
        );
...
}
```

newChild가 React 엘리먼트 이므로 reconcileSingleElement에 진입한다.

**코드**

```javascript
// [Link](https://github.com/facebook/react/blob/9e3b772b8cabbd8cadc7522ebe3dde3279e79d9e/packages/react-reconciler/src/ReactChildFiber.new.js#L1129)
function reconcileSingleElement(  
    returnFiber: Fiber,
    currentFirstChild: Fiber | null,
    element: ReactElement,
    lanes: Lanes,
  ): Fiber {
    const key = element.key;
    let child = currentFirstChild;
    while (child !== null) {
            ...
    }

    if (element.type === REACT_FRAGMENT_TYPE) {
      const created = createFiberFromFragment(
        element.props.children,
        returnFiber.mode,
        lanes,
        element.key,
      );
      created.return = returnFiber;
      return created;
    } else {
		  // [[Link]](https://github.com/facebook/react/blob/9e3b772b8cabbd8cadc7522ebe3dde3279e79d9e/packages/react-reconciler/src/ReactChildFiber.new.js#L1199)
      const created = createFiberFromElement(element, returnFiber.mode, lanes);
      created.ref = coerceRef(returnFiber, currentFirstChild, element);
      created.return = returnFiber;
      return created;
    }
  }
```

1. reconcileChildren의 경우 기존 노드와 비교하여 변경사항을 감지하고, 새롭게 Fiber트리를 만들거나 갱신하는 작업을 수행한다.
2. 이 과정에서 내부에서 호출하는 reconcileChildFibers를 통해 자식 Fiber를 비교하여 어떤 노드를 갱신, 추가 등의 작업을 결정하고
3. reconcileSingleElement를 통해 갱신이 되었는 경우 새로운 fiber를 생성하거나, 기존의 fiber를 재활용한다.

해당 코드는 기존 child가 있는 경우 변경점을 갱신하기 전 정리 작업을 한다. createFiberFromElement()을 호출하여 SimpleComponent 컴포넌트의 FiberNode를 생성한다.

Reconcile 작업은 좁은 의미로 FiberNode를 생성하고, SimpleComponent()를 호출하지 않았다. 왜냐하면 FiberNode를 보면 alternate, child, memoizedState 모두 null이다.

---

```typescript
function updateHostRoot(current, workInProgress, renderLanes) {
...
    // [[Link]](https://github.com/facebook/react/blob/v18.2.0/packages/react-reconciler/src/ReactFiberBeginWork.new.js#L1405)
    return workInProgress.child;
}
```

생성된 SimpleComponent FiberNode는 updateHostRoot()의 마지막에 다음 작업 단위로 반환된다.

```typescript
// [[Link]](https://github.com/facebook/react/blob/v18.2.0/packages/react-reconciler/src/ReactFiberWorkLoop.new.js#L1741)
function workLoopSync() {
  // Already timed out, so perform work without checking if we need to yield.
  while (workInProgress !== null) {
    performUnitOfWork(workInProgress);
  }
}

// [Link]
function performUnitOfWork(unitOfWork: Fiber): void {
...
    next = beginWork(current, unitOfWork, subtreeRenderLanes);
...
  if (next === null) {
    // If this doesn't spawn new work, complete the current work.
    completeUnitOfWork(unitOfWork);
  } else {
    workInProgress = next;
  }

  ReactCurrentOwner.current = null;
}
```

next === workInProgress.child(SimpleComponent)이기 때문에 completeUnitOfWork()로 넘어가지 않고 child(SimpleComponent)를 대상으로 beginWork()로 진입한다. 재조정은 DFS(깊이우선 탐색)이다.

```typescript
function beginWork(
  current: Fiber | null,
  workInProgress: Fiber,
  renderLanes: Lanes,
): Fiber | null {
...
    // [[Link]](https://github.com/facebook/react/blob/v18.2.0/packages/react-reconciler/src/ReactFiberBeginWork.new.js#L3780-L3788)
    switch (workInProgress.tag) {
        case IndeterminateComponent: {
          return mountIndeterminateComponent(
            current,
            workInProgress,
            workInProgress.type,
            renderLanes,
          );
	      }
...
}
```

SimpleComponent()의 tag는 IndeterminateComponent(컴포넌트 타입이 결정되지 않은 상태)로 mountIndeterminateComponent()로 진입한다. 내부에서 renderWithHooks()로 진입하고, 여기서 SimpleComponent()를 호출해 SimpleComponent 컴포넌트를 렌더링한다.

```typescript
// [[Link]](https://github.com/facebook/react/blob/v18.2.0/packages/react-reconciler/src/ReactFiberHooks.new.js#L374)
export function renderWithHooks<Props, SecondArg>(
  current: Fiber | null,
  workInProgress: Fiber,
  Component: (p: Props, arg: SecondArg) => any,
  props: Props,
  secondArg: SecondArg,
  nextRenderLanes: Lanes,
): any {
...
    // [[Link]](https://github.com/facebook/react/blob/v18.2.0/packages/react-reconciler/src/ReactFiberHooks.new.js#L434)
  let children = Component(props, secondArg);
```

이 chlidren이 위에서 보았던 SimpleComponent()함수가 반환하는 React 엘리먼트이다.

```typescript
function mountIndeterminateComponent(
  _current,
  workInProgress,
  Component,
  renderLanes,
) {
...
        // [[Link]](https://github.com/facebook/react/blob/v18.2.0/packages/react-reconciler/src/ReactFiberBeginWork.new.js#L1818)
    reconcileChildren(null, workInProgress, value, renderLanes);
...
}
```

다시 mountIndeterminateComponent()로 돌아오면 reconcileChildren()을 호출한다.

여기서 SimpleComponent의 루트 child인 `<div />`의 FiberNode가 생성된다. 그리고 다시 `<div />` React 엘리먼트를 대상으로 beginWork()를 실행하여 child인 `<h1 />`의 FiberNode를 생성할 것을 예상할 수 있다.

`<h1 />`의 child는 없으므로 next === null이 성립하여 completeWork에 진입한다. completeWork는 completeUnitOfWork()가 completeWork()를 감싸고 있다.

```typescript
// [[Link]](https://github.com/facebook/react/blob/v18.2.0/packages/react-reconciler/src/ReactFiberCompleteWork.new.js#L849)
function completeWork(
  current: Fiber | null,
  workInProgress: Fiber,
  renderLanes: Lanes,
): Fiber | null {
    ...
  switch (workInProgress.tag) {
        ...
      // [[Link]](https://github.com/facebook/react/blob/v18.2.0/packages/react-reconciler/src/ReactFiberCompleteWork.new.js#L959)
    case HostComponent: {
            ...
            // 렌더링된 FiberNode가 있거나 `stateNode`에 DOM 인스턴스가 생성되었다면
      if (current !== null && workInProgress.stateNode != null) {
        // Update 로직
        updateHostComponent(
          current,
          workInProgress,
          type,
          newProps,
          rootContainerInstance,
        );
                ...
      } else {
	        // 없다면 DOM 인스턴스 생성
                    ...
          const instance = createInstance(
            type,
            newProps,
            rootContainerInstance,
            currentHostContext,
            workInProgress,
          );

          appendAllChildren(instance, workInProgress, false, false);

          workInProgress.stateNode = instance;
                    ...
      return null;
    }
```

completeWork의 최초 렌더링 시에는 createInstance()를 호출하여 DOM 인스턴스를 생성한다. createInstance() 내부에서 react-dom 렌더러에서 document.createElement(h1)을 호출하여 h1 인스턴스를 생성한다. 그리고, appendAllChildren()에서 FiberNode.stateNode를 생성된 DOM 인스턴스에 부착한다.

요약 : completeWork는 FiberNode로부터 DOM 인스턴스를 완성시키는 과정이다.

```typescript
function completeUnitOfWork(unitOfWork: Fiber): void {
    ...
        // [[Link]](https://github.com/facebook/react/blob/v18.2.0/packages/react-reconciler/src/ReactFiberWorkLoop.new.js#L1945)
    const siblingFiber = completedWork.sibling;
    if (siblingFiber !== null) {
      // If there is more work to do in this returnFiber, do that next.
      workInProgress = siblingFiber;
      return;
    }
    // Otherwise, return to the parent
    completedWork = returnFiber;
    // Update the next thing we're working on in case something throws.
    workInProgress = completedWork;
  } while (completedWork !== null);

  // We've reached the root.
  if (workInProgressRootExitStatus === RootInProgress) {
    workInProgressRootExitStatus = RootCompleted;
  }
}
```

completeUnitOfWork()의 다음 동작을 살펴보면, completedWork.sibling이 있다면 workInProgress로 지정하고 반환한다. beginWork가 DFS이므로 sibling은 아직 beginWork()를 수행하지 않았기 때문이다. sibling이 없다면 returnFiber, 즉 부모를 workInProgress로 지정하고 completeWork()를 다시 호출한다. completeUnitOfWork()를 벗어나지 않은 채 completeWork()를 부모 FiberNode에 대해서 반복적으로 호출하여 DOM 트리를 완성시켜 나간다.

```typescript
<div>
  <h1>Hello react!</h1>
  <section>
    <p>{`Name : ${name}`}</p>
    <button
      onClick={(e) => setName(name === "Samuel" ? "Alice" : "Samuel")}
    >
      Click me
    </button>
  </section>
</div>
```

`<h1>`의 형제는 `<section>`이므로 `<section>`에 대해 beginWork, `<p>` beginWork, `<p>` completeWork, `<button>` beginWork, … 순서로 루트 `<div>`를 completeWork할 때 까지 순차적으로 탐색할 것을 예측할 수 있다.

### 3. Commit 단계

Reconcile 단계가 모두 끝나면 `RootComplete` status로 Commit 단계에 돌입한다. DOM 엘레멘트도 모두 만들어진 상태이고, 해당 요소들은 갱신된 정보를 기준으로 만들어진 요소이다. 이 때 RootComplete Status가 되며 Commit 단계로 돌입하여 브라우저에 변경된 사항들이 보이도록 그린다.

`finishConcurrentRender` -> `CommitRoot` -> `commitRootImpl` -> `commitMutationEffects` -> `commitMutationEffectsOnFiber`의 순서대로 안쪽으로 들어가면서 함수가 호출되고, 본격적으로 변경 내용을 커밋하기 시작한다.

**코드**

```typescript
function finishConcurrentRender(root, exitStatus, lanes) {
  switch (exitStatus) {
    ...
    case RootCompleted: {
      // The work completed. Ready to commit.
      commitRoot(
        root,
        workInProgressRootRecoverableErrors,
        workInProgressTransitions,
      );
      break;
    }
    default: {
      throw new Error('Unknown root exit status.');
    }
  }
}

function commitRoot(
  root: FiberRoot,
  recoverableErrors: null | Array<CapturedValue<mixed>>,
  transitions: Array<Transition> | null,
) {
    ...
        // [[Link]](https://github.com/facebook/react/blob/v18.2.0/packages/react-reconciler/src/ReactFiberWorkLoop.new.js#L1976)
    commitRootImpl(
      root,
      recoverableErrors,
      transitions,
      previousUpdateLanePriority,
    );
    ...
}

function commitRootImpl(
  root: FiberRoot,
  recoverableErrors: null | Array<CapturedValue<mixed>>,
  transitions: Array<Transition> | null,
  renderPriorityLevel: EventPriority,
) {
    ...
    // [[Link]](https://github.com/facebook/react/blob/v18.2.0/packages/react-reconciler/src/ReactFiberWorkLoop.new.js#L2163)
    commitMutationEffects(root, finishedWork, lanes);
}

export function commitMutationEffects(
  root: FiberRoot,
  finishedWork: Fiber,
  committedLanes: Lanes,
) {
    ...
    // [[Link]](https://github.com/facebook/react/blob/v18.2.0/packages/react-reconciler/src/ReactFiberCommitWork.new.js#L2045)
  commitMutationEffectsOnFiber(finishedWork, root, committedLanes);
    ...
}
```

commitMutationEffects()가 본격적으로 변경 내용을 커밋한다. DFS 방식으로 요소들을 생성한다.

```typescript
function commitMutationEffectsOnFiber(
  finishedWork: Fiber,
  root: FiberRoot,
  lanes: Lanes,
) {
    ...
  // The effect flag should be checked *after* we refine the type of fiber,
  // because the fiber tag is more specific. An exception is any flag related
  // to reconcilation, because those can be set on all fiber types.
  switch (finishedWork.tag) {
        ...
        // [[Link]](https://github.com/facebook/react/blob/v18.2.0/packages/react-reconciler/src/ReactFiberCommitWork.new.js#L2254-L2286)
    case HostRoot: {
      recursivelyTraverseMutationEffects(root, finishedWork, lanes);
      commitReconciliationEffects(finishedWork);
    ...
}
```

최초는 HostRoot이므로 recursivelyTraverseMutationEffects()로 바로 진입한다.

```javascript
function recursivelyTraverseMutationEffects(  
  root: FiberRoot,
  parentFiber: Fiber,
  lanes: Lanes,
) {
    ...
  // [[Link]](https://github.com/facebook/react/blob/v18.2.0/packages/react-reconciler/src/ReactFiberCommitWork.new.js#L2071-L2079)
  const prevDebugFiber = getCurrentDebugFiberInDEV();
  if (parentFiber.subtreeFlags & MutationMask) {
    let child = parentFiber.child;
    while (child !== null) {
      setCurrentDebugFiberInDEV(child);
      commitMutationEffectsOnFiber(child, root, lanes);
      child = child.sibling;
    }
  }
  setCurrentDebugFiberInDEV(prevDebugFiber);
}
```

SimpleComponent는 MutationMask flag가 없어 더 이상 child를 탐색하지 않고 commitReconciliationEffects()로 진입한다.

child를 더 깊이 탐색하지 않는 이유는 요소가 새로 생성되어 DOM에 삽입되는 경우에는 하위 요소들에는 따로 변경점(추가, 이동, 삭제 등)을 나타내는 flag를 새기지 않는다. 그 이유로 생성된 요소의 completeWork()에서 자신의 DOM 요소를 생성하면서 동시에 하위 요소들 중 DOM에 삽입해야 하는 Host Component를 찾아 자신에게 이미 연결했기 때문이다. Commit Phase에는 해당 요소만 DOM에 추가한다면 하위 요소들도 자연스레 DOM에 삽입된다.

```javascript

function commitReconciliationEffects(finishedWork: Fiber) {  
    ...
      commitPlacement(finishedWork);
    ...
}

function commitPlacement(finishedWork: Fiber): void {  
    ...
  // Recursively insert all host nodes into the parent.
  const parentFiber = getHostParentFiber(finishedWork);

  // Note: these two variables *must* always be updated together.
  switch (parentFiber.tag) {
        ...
    case HostRoot:
    case HostPortal: {
      const parent: Container = parentFiber.stateNode.containerInfo;
      const before = getHostSibling(finishedWork);
      insertOrAppendPlacementNodeIntoContainer(finishedWork, before, parent);
      break;
    }
    ...
}
```

현재 parent는 HostRoot이므로 insertOrAppendPlacementNodeIntoContainer()로 진입한다.

```typescript
function insertOrAppendPlacementNodeIntoContainer(
  node: Fiber,
  before: ?Instance,
  parent: Container,
): void {
  const {tag} = node;
  const isHost = tag === HostComponent || tag === HostText;
  if (isHost) {
    const stateNode = node.stateNode;
    if (before) {
      insertInContainerBefore(parent, stateNode, before);
    } else {
      appendChildToContainer(parent, stateNode);
    }
  } else if (tag === HostPortal) {
    // If the insertion itself is a portal, then we don't want to traverse
    // down its children. Instead, we'll get insertions from each child in
    // the portal directly.
  } else {
    const child = node.child;
    if (child !== null) {
      insertOrAppendPlacementNodeIntoContainer(child, before, parent);
      let sibling = child.sibling;
      while (sibling !== null) {
        insertOrAppendPlacementNodeIntoContainer(sibling, before, parent);
        sibling = sibling.sibling;
      }
    }
  }
}
```

SimpleComponent의 루트 child인 `<div>`가 아니라 SimpleComponent FiberNode가 전달된 이유 :  `insertOrAppendPlacementNodeIntoContainer()`가 HostComponent를 만날 때까지 재귀적으로 child를 탐색한다. HostComponent(`<div>`)를 발견하면 마지막 child일 경우 appendChildToContainer()로, sibling이 있는 경우 insertInContainerBefore()를 호출하여 parent DOM 아래에 삽입한다. SimpleComp의 child(`<div>`)의 stateNode는 앞서 Reconcile 단계에서 완성된 DOM 트리이므로 SimpleComp는 한번에 커밋된다.

```typescript
function commitRootImpl(
  root: FiberRoot,
  recoverableErrors: null | Array<CapturedValue<mixed>>,
  transitions: Array<Transition> | null,
  renderPriorityLevel: EventPriority,
) {
    ...
    // The work-in-progress tree is now the current tree. This must come after
  // the mutation phase, so that the previous tree is still current during
  // componentWillUnmount, but before the layout phase, so that the finished
  // work is current during componentDidMount/Update.
  root.current = finishedWork;
}
```

모든 커밋이 끝나면 root.current를 finishedWork로 변경한다. current는 마지막으로 커밋이 끝난 HostRoot FiberNode인 것이다.

### 4. Update 단계

예) 버튼 클릭 시 내용의 변경

```typescript
export function SimpleComponent(): JSX.Element {
  const [name, setName] = React.useState<string>("Alice");
  return <div>
    <h1>Hello react!</h1>
    <section>
      <p>{`Name : ${name}`}</p>
      <button
        onClick={(e) => setName(name === "Samuel" ? "Alice" : "Samuel")}
      >
        Click me
      </button>
    </section>
  </div>
}
```

```javascript
// [[Link]](https://github.com/facebook/react/blob/v18.2.0/packages/react-reconciler/src/ReactFiberWorkLoop.new.js#L1663)
function renderRootSync(root: FiberRoot, lanes: Lanes) {  
    ...
    // [[Link]](https://github.com/facebook/react/blob/v18.2.0/packages/react-reconciler/src/ReactFiberWorkLoop.new.js#L1688)
    prepareFreshStack(root, lanes);
    ...
}

// [[Link]](https://github.com/facebook/react/blob/v18.2.0/packages/react-reconciler/src/ReactFiberWorkLoop.new.js#L1451)
function prepareFreshStack(root: FiberRoot, lanes: Lanes): Fiber {  
    ...
    // [Link]
    const rootWorkInProgress = createWorkInProgress(root.current, null);
    ...
}

// [[Link]](https://github.com/facebook/react/blob/v18.2.0/packages/react-reconciler/src/ReactFiber.new.js#L249)
export function createWorkInProgress(current: Fiber, pendingProps: any): Fiber {  
    let workInProgress = current.alternate;
    ...
}
```

`renderRootSync()`를 시작하면 createWorkInProgress()에서 current(HostRoot FiberNode).alternate를 workInProgress로 사용한다. 즉, React는 업데이트할 때마다 FiberNode.alternate:FiberNode를 사용하여 더블 버퍼링 구조를 가진다는 것을 알 수 있다. 최초 렌더링 후 첫 번째 mutation이므로 이전 current.alternate === null로 새로운 FiberNode를 생성할 것을 예상할 수 있다.

이후 Reconcile 단계의 탐색 과정은 첫 렌더링과 동일하다. 먼저 SimpleComp의 state가 변경되었으므로 updateFunctionComponent()에서 SimpleComp() 렌더링 시 memoizedState의 값이 "Samuel"로 변경된다.

또한 children React 엘리먼트의 children인 `<p>`도 "Samuel"로 변경되었다.

```typescript
// [[Link]](https://github.com/facebook/react/blob/v18.2.0/packages/react-reconciler/src/ReactChildFiber.new.js#L736)
function reconcileChildrenArray(
    returnFiber: Fiber,
    currentFirstChild: Fiber | null,
    newChildren: Array<*>,
    lanes: Lanes,
  ): Fiber | null {
    ...
    // [[Link]](https://github.com/facebook/react/blob/v18.2.0/packages/react-reconciler/src/ReactChildFiber.new.js#L784-L789)
    const newFiber = updateSlot(
        returnFiber,
        oldFiber,
        newChildren[newIdx],
        lanes,
      );
    ...
}
```

```typescript
> newFiber.memoizedProps.children[0].props.children
< 'Name : Alice'
> newFiber.pendingProps.children[0].props.children
< 'Name : Samuel'
```

변경된 props는 `<section>`의 Reconcile 단계에서 `<p>`의 FiberNode의 pendingProps에 할당된다.

첫 렌더링 시 completeWork()에서 DOM 인스턴스를 생성했으므로 state 변경 시에도 completeWork()에서 DOM 인스턴스를 변경할 것이라 생각할 수 있다.

```typescript
function completeWork(
  current: Fiber | null,
  workInProgress: Fiber,
  renderLanes: Lanes,
): Fiber | null {
    ...
    switch (workInProgress.tag) {
        ...
        // [[Link]](https://github.com/facebook/react/blob/v18.2.0/packages/react-reconciler/src/ReactFiberCompleteWork.new.js#L959)
        case HostComponent: {
      popHostContext(workInProgress);
      const rootContainerInstance = getRootHostContainer();
      const type = workInProgress.type;
      if (current !== null && workInProgress.stateNode != null) {
        updateHostComponent(
          current,
          workInProgress,
          type,
          newProps,
          rootContainerInstance,
        );
        ...
    }
}
```

```typescript
// [[Link]](https://github.com/facebook/react/blob/v18.2.0/packages/react-reconciler/src/ReactFiberCompleteWork.new.js#L277-L284)
const updatePayload = prepareUpdate(
      instance,
      type,
      oldProps,
      newProps,
      rootContainerInstance,
      currentHostContext,
    );
// updatePayload === ['children', 'Name : Samuel']
workInProgress.updateQueue = updatePayload;
```

실제로는 위와 같이 oldProps와 newProps를 비교하여 children 갱신 사항만 updateQueue에 저장한다. 그리고 updateQueue가 있으므로 stateNode를 insert하는 대신 commitUpdate()로 진입한다.

```typescript
function updateDOMProperties(
  domElement: Element,
  updatePayload: Array<any>,
  wasCustomComponentTag: boolean,
  isCustomComponentTag: boolean,
): void {
    ...
    // [[Link]](https://github.com/facebook/react/blob/v18.2.0/packages/react-dom/src/client/ReactDOMComponent.js#L362)
    } else if (propKey === CHILDREN) {
      setTextContent(domElement, propValue);
    ...
}
```

최종적으로 updateDOMProperties()에서 setTextContent()를 호출하여 DOM의 text 값을 변경한다.

모든 커밋이 완료되면 첫 렌더링과 마찬가지로 root.current를 작업이 끝난 HostRoot FiberNode로 변경하여 다음 업데이트를 준비한다.
