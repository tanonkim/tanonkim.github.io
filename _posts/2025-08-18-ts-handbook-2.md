---
title: "TS Handbook 2. Everyday Types"
date: 2025-08-18 12:43:00 +0900
categories: [typescript, ts-handbook]
tags: [typescript]
---

### 원시타입 : string, number, boolean

### 배열

`Array<number>`와 같은 형태로 작성이 가능하며, number[]와 동일한 의미를 지닌다.

### any

any : 특정 값으로 인하여 타입 검사 오류가 발생하는 것을 원하지 않을 때 사용

noImplicitAny : 컴파일러 플래그를 사용하면 any로 간주하는 모든 경우에 오류를 발생시킨다.

### **변수에 대한 타입 표기**

const, var, let 등을 사용하여 변수를 선언할 때 변수의 타입을 명시적으로 지정하기 위함,

```typescript
let myName: string = "Alice";
```

### 함수

- 매개변수 타입 표기

```typescript
// 매개변수 타입 표기
function greet(name: string) {
  console.log("Hello, " + name.toUpperCase() + "!!");
}
```

- 반환 타입 표기 (매개변수 목록 뒤에 표기)

```typescript
function getFavoriteNumber(): number {
  return 26;
}
```

- 익명함수

```typescript
// 아래 코드에는 타입 표기가 전혀 없지만, TypeScript는 버그를 감지할 수 있습니다.
const names = ["Alice", "Bob", "Eve"];
 
// 함수에 대한 문맥적 타입 부여
names.forEach(function (s) {
  console.log(s.toUppercase());
Property 'toUppercase' does not exist on type 'string'. Did you mean 'toUpperCase'?
});
 
// 화살표 함수에도 문맥적 타입 부여는 적용됩니다
names.forEach((s) => {
  console.log(s.toUppercase());
Property 'toUppercase' does not exist on type 'string'. Did you mean 'toUpperCase'?
});
```

### 객체 타입

해당 객체의 프로퍼티들과 각 프로퍼티의 타입들을 나열하기만 하면 된다.

```typescript
// 매개 변수의 타입은 객체로 표기되고 있습니다.
function printCoord(pt: { x: number; y: number }) {
  console.log("The coordinate's x value is " + pt.x);
  console.log("The coordinate's y value is " + pt.y);
}
printCoord({ x: 3, y: 7 });
```

- 옵셔널 프로퍼티

```typescript
function printName(obj: { first: string; last?: string }) {
  // ...
}
// 둘 다 OK
printName({ first: "Bob" });
printName({ first: "Alice", last: "Alisson" });
```

### 유니언 타입

기존 타입을 기반으로 다양한 연산자를 통해 새로운 타입을 만들 수 있다.

```typescript
function printId(id: number | string) {
  console.log("Your ID is: " + id);
}
// OK
printId(101);
// OK
printId("202");
// 오류
printId({ myID: 22342 }); // Argument of type '{ myID: number; }' is not assignable to parameter of type 'string | number'.
```

- 유니언 타입 사용하기

`string | number`라는 유니언 타입의 경우, `string` 타입에만 유효한 메서드는 사용할 수 없습니다. → 분기로 구체적인 타입을 좁혀야 한다.

```typescript
function printId(id: number | string) {
  console.log(id.toUpperCase());
// Property 'toUpperCase' does not exist on type 'string | number'.
// Property 'toUpperCase' does not exist on type 'number'.
}
```

```typescript
function printId(id: number | string) {
  if (typeof id === "string") {
    // 이 분기에서 id는 'string' 타입을 가집니다
 
    console.log(id.toUpperCase());
  } else {
    // 여기에서 id는 'number' 타입을 가집니다
    console.log(id);
  }
}
```

### 타입 별칭

해당 타입을 직접 표기하는 방법 ↔ 똑같은 타입을 한 번 이상 재사용하거나 다른 이름으로 부르고 싶을 때 타입 별칭을 사용한다.

```typescript
type Point = {
  x: number;
  y: number;
};
 
// 앞서 사용한 예제와 동일한 코드입니다
function printCoord(pt: Point) {
  console.log("The coordinate's x value is " + pt.x);
  console.log("The coordinate's y value is " + pt.y);
}
 
printCoord({ x: 100, y: 100 });
```

### 인터페이스

인터페이스 선언은 객체 타입을 만드는 또 다른 방법이다.

```typescript
interface Point {
  x: number;
  y: number;
}
 
function printCoord(pt: Point) {
  console.log("The coordinate's x value is " + pt.x);
  console.log("The coordinate's y value is " + pt.y);
}
 
printCoord({ x: 100, y: 100 });
```

`interface`가 가지는 대부분의 기능은 `type`에서도 동일하게 사용 가능하다. 이 둘의 가장 핵심적인 차이는, 타입은 새 프로퍼티를 추가하도록 개방될 수 없는 반면, 인터페이스의 경우 항상 확장될 수 있다는 점이다.

### 타입 단언

TS보다 개발자가 어떤 값의 타입에 대한 정보를 더 잘 아는 경우가 존재한다. 이 경우 타입 단언을 사용하면 좀 더 구체적으로 명시할 수 있다. 

```typescript
const myCanvas = document.getElementById("main_canvas") as HTMLCanvasElement;
```

- 두 번의 단언

  ```typescript
  const x = "hello" as number; // 강제 변환이 허용되지 않는 경우

  const a = (expr as any) as T;
  const a = (expr as unknown) as T;
  ```

  강제 변환이 허용되지 않는 경우 두번의 단언을 선언하면 원하는 타입으로 변환이 된다.

### 리터럴 타입

string과 number와 같은 타입 이외에도 구체적인 문자열과 숫자 값을 타입 위치에서 지정할 수 있다. var/let/const의 특징은 TS가 리터럴 값을 위한 타입을 생성하는 방식에 그대로 반영된다.

```typescript
let changingString = "Hello World";
changingString = "Olá Mundo";
changingString; // let changingString: string

const constantString = "Hello World";
constantString; // const constantString: "Hello World"
```

- 여러 종류의 값들만 인자로 받을 수 있는 함수

```typescript
function printText(s: string, alignment: "left" | "right" | "center") {
  // ...
}
printText("Hello, world", "left");
printText("G'day, mate", "centre");
```

- 리터럴 추론

```typescript
// 1

const req = { url: "https://example.com", method: "GET" };
// handleRequest(req.url, req.method); // Argument of type 'string' is not assignable to parameter of type '"GET" | "POST"'.
handleRequest(req.url, req.method as "GET");

// 2
const req = { url: "https://example.com", method: "GET" } as const;
handleRequest(req.url, req.method);
```

### null, undefined

JS에서 빈 값, 초기화 되지 않은 값을 가리키는 두 원시값이다. 각 타입의 동작 방식은 strictNullChecks 옵션의 설정 여부에 따라 달라진다. 

- strictNullChecks가 설정되지 않았을 때

  null, undefined는 모든 타입의 변수에 대입될 수 있다. → Null검사의 부재는 버그의 주요원인이 될 수 있기에 해당 옵셩르 설정하는 것을 권장한다.

- strictNullChecks가 설정되었을 때

  null, undefined일 때 메서드/프로퍼티를 사용하기 전에 테스트를 해야 한다.

  ```typescript
  function doSomething(x: string | undefined) {
    if (x === undefined) return
    console.log("Hello, " + x.toUpperCase());  
  }
  ```

  Null 아님을 알리는 단언 연산자 `!`

  ```typescript
  function liveDangerously(x?: number | undefined) {
    // 오류 없음
    console.log(x!.toFixed());
  }
  ```

### bigint, symbol

- bigint

  ES2020 이후 bigint가 JS에 추가되었다. 

  ```typescript
  // BigInt 함수를 통하여 bigint 값을 생성
  const oneHundred: bigint = BigInt(100);
   
  // 리터럴 구문을 통하여 bigint 값을 생성
  const anotherHundred: bigint = 100n;
  ```

- symbol

  symbol은 고유한 참조값을 생성하는데에 사용할 수 있는 원시 타입이다.

  ```typescript
  const firstName = Symbol("name");
  const secondName = Symbol("name");
   
  if (firstName === secondName) {
    // This comparison appears to be unintentional because the types 'typeof firstName' and 'typeof secondName' have no overlap.
    // 절대로 일어날 수 없습니다
  }
  ```
