---
title: "TS Handbook 5. Object Types"
date: 2025-08-19 15:29:00 +0900
categories: [typescript, ts-handbook]
tags: [typescript]
---

JS에서 데이터를 그룹화하고 전달하는 가장 기본적인 방법은 객체를 사용하는 것이다.

타입스크립트에서 이러한 것을 객체 타입이라고 한다.

- 익명으로 사용이 가능하다.

  ```typescript
  function greet(person: { name: string; age: number }) {
    return "Hello " + person.name;
  }
  ```

- interface로 이름을 지정할 수 있다.

  ```typescript
  interface Person {
    name: string;
    age: number;
  }
   
  function greet(person: Person) {
    return "Hello " + person.name;
  }
  ```

- 타입 별칭을 사용할 수도 있다.

  ```typescript
  type Person = {
    name: string;
    age: number;
  };

  function greet(person: Person) {
    return "Hello " + person.name;
  }
  ```

### Property Modifiers

객체 타입의 각 속성은 타입, optional 쓰여질 수가 있는지 등을 지정할 수 있다. 

- Optional Properties

  예) 선택적으로 적용된 `xPos`와 `yPos`

  ```typescript
  interface PaintOptions {
    shape: Shape;
    xPos?: number;
    yPos?: number;
  }
   
  function paintShape(opts: PaintOptions) {
    // ...
  }
   
  const shape = getShape();
  paintShape({ shape });
  paintShape({ shape, xPos: 100 });
  paintShape({ shape, yPos: 100 });
  paintShape({ shape, xPos: 100, yPos: 100 });
  ```

  또 strictNullChecks가 설정되었을 때 타입스크립트는 해당 속성들이 잠재적으로 undefined라는 것을 보여준다.

  ```typescript
  function paintShape(opts: PaintOptions) {
    let xPos = opts.xPos;                 
  //(property) PaintOptions.xPos?: number | undefined
    let yPos = opts.yPos;
  //(property) PaintOptions.yPos?: number | undefined
    // ...
  }
  ```

  특정되지 않는 값의 기본값을 설정하는 것은 자바스크립트가 지원하는 문법이다.

  ```typescript
  function paintShape({ shape, xPos = 0, yPos = 0 }: PaintOptions) {
    console.log("x coordinate at", xPos);
  //(parameter) xPos: number
    console.log("y coordinate at", yPos);
  //(parameter) yPos: number
    // ...
  }
  ```
