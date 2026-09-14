---
title: "React 공식문서 - Section2 틱택토"
date: 2025-08-06 11:56:00 +0900
categories: [react, react-docs]
tags: [react]
---

JSX : JS + HTML 태그의 조합

react-dom/client의 의미

### Array().fill()

```typescript
// ...
export default function Board() {
  const [squares, setSquares] = useState(Array(9).fill(null));
  // [null,null,null,null,null,null,null,null,null]
  return (
    // ...
  );
}
```

### Closure

```typescript
export default function Board() {
  const [squares, setSquares] = useState(Array(9).fill(null));

  function handleClick(i) {
    const nextSquares = squares.slice();
    nextSquares[i] = "X";
    setSquares(nextSquares);
  }

  return (
    // ...
  )
}
```

자바스크립트는 클로저 지원 → handleClick 외부 함수, 즉 Board에 정의된 변수, 함수에 접근이 가능함

### Too- many re-renders

```typescript
<Square value={squares[0]} onSquareClick={handleClick(0)} />
```

Closure 예시의 코드에서 Square에 handleClick(0)으로 지정하면 아래의 오류가 발생함.

> Too many re-renders. React limits the number of renders to prevent an infinite loop.

이유 : handleClick(0)의 호출은 보드 컴포넌트 렌더링의 일부이다. handleClick(0)은 setSquare를 호출하여 State를 변경하기에 Board 컴포넌트 전체가 다시 리렌더링된다. 리렌더링 후 다시 handleClick(0)이 실행되기에 무한루프에 빠지게 된다.

⇒ 함수를 호출한 것이 아니라, `함수를 Prop으로 전달`함.

**해결 방법**

```typescript
<Square value={squares[0]} onSquareClick={() => handleClick(0)} />
```

### 불변성

handleClick함수에서 기존 배열을 수정하지 않고 `.slice()`를 호출하여 사본을 생성하였다.

아래는 `squares` 배열을 변형하지 않고 데이터를 변경한 경우의 모습이다.

```typescript
const squares = [null, null, null, null, null, null, null, null, null];
const nextSquares = ['X', null, null, null, null, null, null, null, null];
```

### **한 번 더 State 끌어올리기 - 다시 읽기**

[https://ko.react.dev/learn/tutorial-tic-tac-toe#lifting-state-up-again](https://ko.react.dev/learn/tutorial-tic-tac-toe#lifting-state-up-again)
