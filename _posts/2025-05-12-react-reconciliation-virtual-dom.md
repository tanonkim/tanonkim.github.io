---
title: "React - Reconciliation / Virtual DOM"
date: 2025-05-12 21:13:00 +0900
categories: [react, react-study]
tags: [react, virtual-dom]
---

{% raw %}
## Reconciliation 요약

리액트는 컴포넌트에서 prop이나 state가 변경이 될 때, 직전의 렌더링 요소(element)와 비교하여 실제 DOM을 업데이트를 할지 말지 결정해야 한다. 이 때 element가 일치하지 않으면 리액트는 새로운 요소로 DOM을 업데이트 해야 하는데, 이런 프로세스를 `Reconciliation`이라고 한다.

![](/assets/img/2025-05-12/react-reconciliation-virtual-dom-1.png)

## Virtual DOM

### DOM이란

Document Object Model로 HTML/XML과 같은 웹문서의 구조화된 표현이다. 프로그래밍 언어가 웹페이지 내의 객체에 접근하고 조작가능하게 하는 인터페이스로 동작하며, 주로 자바스크립트에서 웹페이지의 요소를 선택/수정/추가/제거할 때 사용된다.

### DOM의 속도

DOM은 트리구조로 표현된다. 따라서 변경과 업데이트는 비교적 빠른 편이다. 하지만 변경된 후 업데이트 된 element를 다시 렌더링하여 UI를 업데이트 해야 하는데, HTML → (DOM created with JS) → RenterTree → Style → Layout → Paint → Composite → Display 과정을 매번 거치기에 UI구성 요소가 많아지면 리렌더링 속도가 느려진다.

이 과정의 시간 복잡도 : O(N3)

### Virtual DOM

이에 대한 해결책으로 업데이트가 필요한 **최소한의 DOM만 조작**하는 것이다.

![](/assets/img/2025-05-12/react-reconciliation-virtual-dom-2.png)

리액트는 이를 위해 실제 DOM 대신 사본과 같은 Virtual DOM을 도입하였고, 가상의 DOM을 업데이트하는 방식을 사용해 실제 DOM의 업데이트 횟수를 줄인다.

**과정**

1. 업데이트 한 전체 UI를 Virtual DOM에 리렌더링
2. 실제 DOM과 Virtual DOM을 비교 (Diffing Algorithm을 사용)
3. 바뀐 부분만 실제 DOM을 적용
   1. 최소한의 렌더링이 가능해진다.
   2. ReactDOM.render()가 React element를 DOM에 렌더링할 때 필요한 부분만 렌더링한다. (React 17)
      1. React 18 : ReactDOM.hydrate() → SSR 후 클라이언트 하이드레이션
      2. React 18 : React.createRoot()

   **1. 클라이언트 사이드 렌더링 (CSR)**

   ```typescript
   import { createRoot } from 'react-dom/client';

   const root = createRoot(document.getElementById('root'));
   root.render(<App />);
   ```

   **2. 서버 사이드 렌더링 (SSR) 후 하이드레이션**

   ```typescript
   import { hydrateRoot } from 'react-dom/client';

   hydrateRoot(document.getElementById('root'), <App />);
   ```

리액트는 이러한 내부 동작을 추상화하고 바뀐 부분을 업데이트하기 위해 state값만 변경하면 된다. → 따라서 리액트는 **선언적**이라고 할 수 있다. (React에게 원하는 UI의 상태를 알려주면 DOM이 그 상태와 일치하도록 하기 때문에, 내부 알고리즘의 동작 등은 개발자가 알 필요가 적어진다.)

## Reconciliation

### Diffing Algorithm (비교 알고리즘)

두 개의 트리를 비교할 때 React는 두 엘리먼트의 root부터 비교한다.

예1) element의 타입이 다른 경우 : 이전의 트리를 버리고 완전히 새로운 트리를 구축

```typescript
// 이전 트리
<div>
  <Counter />
</div>

// 새로운 트리
<span>
  <Counter />
</span>

// 이전 Counter는 Drop, 새로운 Counter 사용
```

예2) DOM 엘리먼트의 타입이 같은 경우 : 두 속성의 엘리먼트의 속성을 확인 후 동일한 내용은 유지, 변경된 속성만 갱신

```typescript
// Changed ClassName
<div className="before" title="stuff" />
<div className="after" title="stuff" /> // className만 수정

// Changed Styles
<div style={{color: 'red', fontWeight: 'bold'}} />
<div style={{color: 'green', fontWeight: 'bold'}} /> // color속성만 수정
```

DOM의 노드 처리가 끝나면 React는 해당 노드의 자식들을 재귀적으로 처리한다.

### 재귀적 처리

DOM 노드의 자식들을 재귀적으로 처리할 때, React는 기본적으로 동시에 두 리스트를 순회하고 차이점이 있으면 변경된 부분을 갱신한다.

예) `<li>third</li>` 추가

```typescript
// 이전 트리
<ul>
  <li>first</li>
  <li>second</li>
</ul>

// 새로운 트리
<ul>
  <li>first</li>
  <li>second</li>
  <li>third</li>
</ul>
```

위에서 비교하다가 새로운 트리에서 third가 생겼으므로 해당 태그만 트리에 추가한다.

**Key**

```typescript
// 이전 트리
<ul>
  <li>first</li>
  <li>second</li>
</ul>
// 새로운 트리
<ul>
	<li>third</li> // 추가
  <li>first</li>
  <li>second</li>
</ul>
```

first와 second가 같음에도 불구하고 위에서부터 트리를 전체 교체한다. 즉 불필요한 렌더잉이 존재하기에 성능저하를 유발한다. → 이 문제를 해결하기 위해 key 속성을 제공

자식 노드들에 key가 할당되었다면, React는 key를 통해 기존 트리와 변경 이후 트리의 자식들이 일치하는지 확인한다.

```typescript
// 이전 트리
<ul>
  <li key="100">first</li>
  <li key="200">second</li>
</ul>
// 새로운 트리
<ul>
	<li key="300">third</li>
  <li key="100">first</li>
  <li key="200">second</li>
</ul>
```

위의 코드는 300이라는 key를 가진 third만 새로운 트라에 추가되고 나머지는 이동만 하게 된다.
{% endraw %}
