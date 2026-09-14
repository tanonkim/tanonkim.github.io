---
title: "TS Handbook 3. Narrowing"
date: 2025-08-18 13:49:00 +0900
categories: [typescript, ts-handbook]
tags: [typescript]
---

타입을 좁혀 나간다의 의미이다. TS에서 제공하는 Type guards에 대해 알아보자

타입스크립트는 타입 안정성을 위해 자바스크립트 코드를 최대한 쉽게 작성하는 것을 목표로 한다.

예) 

```typescript
function padLeft(padding: number | string, input: string) {
  if (typeof padding === "number") {
    return " ".repeat(padding) + input;
  }
  return padding + input;
}
```

타입스크립트는 typeof라는 타입 가드의 형태로 구체적인 유형으로 한번 더 Narrowing해주는 작업을 진행한다.

### typeof type guards

자바스크립트 typeof를 지원하여 런타임에 가지고 있는 값의 유형에 대해 정보를 제공한다. TS는 이를 통해 특정 문자열 집합을 반환할 것으로 예상한다.

- string
- number
- bigint
- boolean
- symbol
- undefined
- object
- function

TS에서 typeof에 반환되는 값에 대해 체크하는 것이 type guard이다. 

```typescript
function printAll(strs: string | string[] | null) {
  if (typeof strs === "object") {
    for (const s of strs) {
      // 'strs' is possibly 'null'
      console.log(s);
    }
  } else if (typeof strs === "string") {
    console.log(strs);
  } else {
    // do nothing
  }
}
```

'strs' is possibly 'null' 에러가 발생하는데, 그 이유는 object 타입으로 타입을 좁히면 string[], null이 narrowing된다. (null의 타입도 object이다.)

### Truthiness narrowing

Javascript에서 조건에 대한 표현식을 사용한다. &&, \|\|, if, ! 등

자바스크립트에서는 if와 같은 분기문으로 해당 조건에 따라 강제 분기를 한다.

false로 강제되는 값

- 0
- NaN
- “” (empty string)
- null
- 0n(the bigint version of Zero)
- null
- undefined

위의 값은 false로 강제되고, 나머지는 다 true로 강제된다.

```typescript
function printAll(strs: string | string[] | null) {
  if (strs && typeof strs === "object") {
    for (const s of strs) {
      console.log(s);
    }
  } else if (typeof strs === "string") {
    console.log(strs);
  }
}
```

위의 코드로 truthy한 값을 확인하면서 나머지 에러들은 자연스럽게 제거한다. typeof 키워드는 Null 체크를 못하기 때문에 조건에서 falsy를 필터링해서 narrowing을 한다. 

```typescript
function multiplyAll(
  values: number[] | undefined,
  factor: number
): number[] | undefined {
  if (!values) {
    return values;
  } else {
    return values.map((x) => x * factor);
  }
}
```

truthiness를 `!`으로 부정함으로써 좁히는 것도 방법이다.

### Equally narrowing

TS 또한 switch문을 사용하고, `===`, `≠=`, `==`, `≠` 와 같은 동등성 검사를 좁은 유형으로 사용한다. 

```typescript
if (x === y) {
    // We can now call any 'string' method on 'x' or 'y'.
    x.toUpperCase(); //(method) String.toUpperCase(): string
    y.toLowerCase(); //(method) String.toLowerCase(): string
  } else {
    console.log(x); //(parameter) x: string | number
    console.log(y); //(parameter) y: string | boolean
  }
}
```

x,y가 같은 경우는 string이여야 동일함을 알고 있다. 따라서 첫 if문에서 TS는 x,y를 string으로 narrowing한다. else문에서는 다양한 경우의 수가 오는 것 까지 TS가 자동으로 추론한다.

### The `in` operator narrowing

JS에는 개체에 이름이 있는 속성인 in연산자가 있는지 여부를 결정하는 Type Guard가 있다. 잠재적으로 타입을 in으로 narrowing하여 고려한다. 

```typescript
type Fish = { swim: () => void };
type Bird = { fly: () => void };

function move(animal: Fish | Bird) {
	if ("swim" in animal) {
		return animal.swim();
	}
	return animal.fly();
}
```

animal에는 Fishfhk Bird가 있는데, animal 속성에 따라 다르게 move동작을 해야 할 때 in을 사용하여 narrowing한다.

### instanceof narrowing

자바스크립트에는 값이 다른 값의 인스턴스인지 여부를 확인하는 typeguard 연산자가 있다. instanceof에 의해 타입을 narrowing한다.

```typescript
function logValue(x: Date | string) {
  if (x instanceof Date) {
    console.log(x.toUTCString());
           //(parameter) x: Date
  } else {
    console.log(x.toUpperCase());
          //(parameter) x: string
  }
}
```

### Assignments, 할당을 통한 narrowing

예) 삼항연산자를 통한 string, number타입으로의 narrow

```typescript
let x = Math.random() < 0.5 ? 10 : "hello world!";  //let x: string | number
x = 1;
console.log(x);   //let x: number
x = true; // boolean 값을 넣으려고 하면 에러가 남!
//Type 'boolean' is not assignable to type 'string | number'.
//처음에 선언한 것만 사용이 가능하다.
console.log(x); // let x: string | number
```

### Using type predicates

사용자 정의 type guard를 직접 정의하기 위해서 is키워드를 통해 제어할 수 있다.

```typescript
function isFish(pet: Fish | Bird): pet is Fish {
  return (pet as Fish).swim !== undefined;
}
```

### Discriminated unions

```typescript
interface Shape {
  kind: "circle" | "square";
  radius?: number;
  sideLength?: number;
}
```

문자의 타입을 union을 사용하고 있다. 

예) 오류 : 넓이 구하기

```typescript
function getArea(shape: Shape) {
  return Math.PI * shape.radius ** 2; // 'shape.radius' is possibly 'undefined'.
}
```

radius가 정의되어있지 않을 수 있기 때문에 에러가 발생한다. 

```typescript
function getArea(shape: Shape) {
  if (shape.kind === "circle") {
    return Math.PI * shape.radius ** 2; // 'shape.radius' is possibly 'undefined'.
  }
}
```

radius를 확언하여 무조건 undefined가 아님을 알려야 한다.

```typescript
function getArea(shape: Shape) {
  if (shape.kind === "circle") {
    return Math.PI * shape.radius! ** 2;
  }
}
```

예) 더 개선한 코드

더 적절한 코드는 아래와 같다.

```typescript
interface Circle {
  kind: "circle";
  radius: number;
}
 
interface Square {
  kind: "square";
  sideLength: number;
}
 
type Shape = Circle | Square;

function getArea(shape: Shape) {
  if (shape.kind === "circle") {
    return Math.PI * shape.radius ** 2;                     
  }
}
```

### The `never` type

TS는 존재하지 않아야 하는 상태를 나타내기 위해 never 유형을 사용한다.

- Exhaustiveness checking(철저한 체크)

  Never유형은 모든 유형에 할당가능하지만, Never 유형은 할당할 수 없다. 

  ```typescript
  type Shape = Circle | Square;
   
  function getArea(shape: Shape) {
    switch (shape.kind) {
      case "circle":
        return Math.PI * shape.radius ** 2;
      case "square":
        return shape.sideLength ** 2;
      default:
        const _exhaustiveCheck: never = shape;
        return _exhaustiveCheck;
    }
  }
  ```

  Shape 타입에 Triangle을 추가하게 되면 TS에는 어떠한 타입도 와서는 안되는 never타입인데, 타입이 있다는 에러가 발생한다.

  ```typescript
  interface Triangle {
    kind: "triangle";
    sideLength: number;
  }
   
  type Shape = Circle | Square | Triangle;
   
  function getArea(shape: Shape) {
    switch (shape.kind) {
      case "circle":
        return Math.PI * shape.radius ** 2;
      case "square":
        return shape.sideLength ** 2;
      default:
        const _exhaustiveCheck: never = shape;
  Type 'Triangle' is not assignable to type 'never'.
        return _exhaustiveCheck;
    }
  }
  ```
