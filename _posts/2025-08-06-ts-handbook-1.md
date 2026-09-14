---
title: "TS Handbook 1. The Basics"
date: 2025-08-06 18:31:00 +0900
categories: [typescript, ts-handbook]
tags: [typescript]
---

### 정적 타입 검사

```typescript
const message = "hello!";
 
message(); // This expression is not callable.
```

정적타입 검사 : 코드를 실행하기 전에 미리 버그를 발견하게 하는 도구 (By ECMAScript 명세)

### **예외가 아닌 실행 실패**

- JS

```typescript
const user = {
  name: "Tanon",
  age: 26,
};
user.location; // undefined 를 반환 
```

- TS

```typescript
const user = {
  name: "Tanon",
  age: 26,
};
 
user.location;
// Property 'location' does not exist on type '{ name: string; age: number; }'.
```

### **프로그래밍 도구로서의 타입**

타입 검사기는 우리가 변수 또는 다른 프로퍼티 상의 올바른 프로퍼티에 접근하고 있는지 여부를 검사할 수 있도록 관련 정보들을 가지고 있다. 이 정보를 활용하면 타입 검사기는 우리가 사용할 수 있는 프로퍼티를 제안할 수 있게 된다.

### tsc, TypeScript 컴파일러

```typescript
// 설치
npm install -g typescript
```

- hello.ts

```typescript
console.log("Hello world!");
```

`tsc hello.ts`

→ hello.js가 만들어짐

### **명시적 타입**

```typescript
function greet(person: string, date: Date) {
  console.log(`Hello ${person}, today is ${date.toDateString()}!`);
}


greet("Maddison", new Date()); // OK
greet("Maddison", Date()); // Argument of type 'string' is not assignable to parameter of type 'Date'.
```

Date() : string을 반환함

### **지워진 타입**

- tsc로 컴파일된 JavaScript

```typescript
"use strict";
function greet(person, date) {
    console.log("Hello ".concat(person, ", today is ").concat(date.toDateString(), "!"));
}
greet("Maddison", new Date());
```

1. `person`과 `date` 인자는 더 이상 타입 표기를 가지지 않는다.
2. 템플릿 문자열(\`\`)은 연결 연산자(+)로 이루어진 일반 문자열로 변환되었다.

### 다운레벨링

앞서 언급된 또 다른 차이점은, 바로 템플릿 문자열(ECMAScript 2015)이 아래의 내용에서

```typescript
`Hello ${person}, today is ${date.toDateString()}!`;
```

아래의 내용으로 다시 작성되었다는 점이다.

```typescript
"Hello " + person + ", today is " + date.toDateString() + "!";
```

TypeScript는 새 버전의 ECMAScript의 코드를 ECMAScript 3 또는 ECMAScript 5와 같은 보다 예전 버전의 것들로 다시 작성해 준다. ⇒ `다운레벨링`

### 엄격도

타입 검사는 선택 사항이며, 타입 추론은 가장 관대한 기준으로 이루어지고, 잠재적인 `null`/`undefined` 값에 대한 검사는 이루어지지 않는다. 

- 엄격도 플래그

  CLI에서 —strict플래그 설정

  tsconfig.json에서 `“strict":true` 추가

- noImplicitAny

  any로 암묵적으로 추론되는 변수에 대해 오류를 발생시킴

- strictNullChecks

  null, undefined를 보다 명시적으로 처리
