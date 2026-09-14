---
title: "React 공식문서 - Section1 학습하기"
date: 2025-08-05 13:40:00 +0900
categories: [react, react-docs]
tags: [react]
---

### 컴포넌트 만들고 중첩하는 방법

React앱은 컴포넌트로 구성됨.

컴포넌트 : 고유한 로직과 모양을 가진 사용자 인터페이스의 일부

```typescript
function MyButton() {
  return (
    <button>
      I'm a button
    </button>
  );
}

export default function MyApp() {
  return (
    <div>
      <h1>Welcome to my app</h1>
      <MyButton />
    </div>
  );
}
```

React 컴포넌트는 항상 대문자로 시작

### 마크업과 스타일 추가하는 방법

JSX : 마크업 문법 (JS + HTML)

```typescript
function AboutPage() {
  return (
    <>
      <h1>About</h1>
      <p>Hello there.<br />How do you do?</p>
    </>
  );
}
```

스타일

```typescript
<img className="avatar" />
```

```typescript
/* In your CSS */
.avatar {
  border-radius: 50%;
}
```

### 데이터를 표시하는 방법

```typescript
return (
  <h1>
    {user.name}
  </h1>
);
```

```typescript
return (
  <img
    className="avatar"
    src={user.imageUrl}
  />
);
```

### 조건과 리스트를 렌더링하는 방법

if문을 사용하여 조건부로 JSX를 포함가능

```typescript
let content;
if (isLoggedIn) {
  content = <AdminPanel />;
} else {
  content = <LoginForm />;
}
return (
  <div>
    {content}
  </div>
);
```

더 간단한 코드

```typescript
<div>
  {isLoggedIn ? (
    <AdminPanel />
  ) : (
    <LoginForm />
  )}
</div>
```

### 리스트 렌더링하기

for, map 문법과 같이 사용

```typescript
const products = [
  { title: 'Cabbage', id: 1 },
  { title: 'Garlic', id: 2 },
  { title: 'Apple', id: 3 },
];
```

컴포넌트 내에서 map() 함수를 사용하여 제품 배열을 `<li>` 항목 배열로 변환합니다.

```typescript
const listItems = products.map(product =>
  <li key={product.id}>
    {product.title}
  </li>
);

return (
  <ul>{listItems}</ul>
);
```

`key` : 항목을 삽입, 삭제 또는 재정렬할 때 어떤일이 일어났는지 알기 위해 key 사용

### 이벤트에 응답하고 화면을 업데이트하는 방법

### 이벤트에 응답하기

컴포넌트 내부에 이벤트 핸들러 함수를 선언하여 이벤트에 응답할 수 있음 + 함수만 전달하기에 소괄호가 필요하지 않음

```typescript
function MyButton() {
  function handleClick() {
    alert('You clicked me!');
  }

  return (
    <button onClick={handleClick}>
      Click me
    </button>
  );
}
```

**`참고` onClick={f}, onClick={() ⇒ f()}**

onClick={handleClick}

- **렌더링 시**:

  `handleClick`은 이미 정의된 함수이므로, 함수 **참조만** 넘겨줍니다.

  추가로 새로운 함수 객체를 만들지 않음.

- **메모리 점유**:

  기존 함수 객체 1개만 유지.

- **GC 영향**:

  함수 참조가 바뀌지 않으므로 불필요한 GC 대상이 거의 없음.

onClick={() ⇒ handleClick()}

- 렌더링 시 :

  매번 새로 화살표 함수를 생성함

  → 함수 객체는 렌더마다 새 메모리 주소를 가짐

- 메모리 점유:

  리렌더링 횟수만큼 새로운 함수 객체가 계속 생성된다.

- GC 영향:

  이전 렌더링에서 만든 함수 객체는 더 이상 참조되지 않으면 GC로 수거됨

  하지만 리렌더링이 잦을수록 GC 빈도와 부하가 증가됨

### 화면 업데이트 하기

```typescript
function MyButton() {
  const [count, setCount] = useState(0);

  function handleClick() {
    setCount(count + 1);
  }

  return (
    <button onClick={handleClick}>
      Clicked {count} times
    </button>
  );
}

import { useState } from 'react';

export default function MyApp() {
  return (
    <div>
      <h1>Counters that update separately</h1>
      <MyButton />
      <MyButton />
    </div>
  );
}
```

→ 각 버튼은 고유의 State를 가진다. (state가 각각 다름)

### Hook 사용하기

use로 시작하는 함수를 Hook이라고 함.

### 컴포넌트 간에 데이터를 공유하는 방법

`MyButton`에 독립적인 `count`가 있었고, 각 버튼을 클릭하면 클릭한 버튼의 `count`만 변경되었다.

두 MyButton 컴포넌트가 동일한 count를 표시하고 함께 업데이트하는 방법 ⇒ 상단 컴포넌트에서 상태 관리하기 / State 끌어올리기

```typescript
export default function MyApp() {
  const [count, setCount] = useState(0);

  function handleClick() {
    setCount(count + 1);
  }

  return (
    <div>
      <h1>Counters that update together</h1>
      <MyButton count={count} onClick={handleClick} />
      <MyButton count={count} onClick={handleClick} />
    </div>
  );
}
```
