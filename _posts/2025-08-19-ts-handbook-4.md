---
title: "TS Handbook 4. More on Functions"
date: 2025-08-19 15:05:00 +0900
categories: [typescript, ts-handbook]
tags: [typescript]
---

함수는 어느 어플리케이션에서도 기초적인 구성 요소의 역할을 한다. 함수는 값이다.  TS에서는 함수들이 호출될 수 있는 방법을 서술하는 방법이 많이 있다. 

### 함수 타입 표현식

함수를 설명하는 가장 간단한 방법은 함수 타입 표현식이다. 화살표 함수와 문법적으로 유사하다.

```typescript
function greeter(fn: (a: string) => void) {
  fn("Hello, World");
}
 
function printToConsole(s: string) {
  console.log(s);
}
 
greeter(printToConsole);
```

(a : string) ⇒ void라는 문법은 a를 하나의 매개변수로 가지고, 반환값이 없는 함수를 의미한다. 매개변수의 타입이 없다면 암묵적으로 any가 된다.

### 호출 시그니처

JS에서 함수들은 호출이 가능할 뿐만 아니라, 프로퍼티도 가질 수 있다. 하지만, 함수 타입 표현식 문법은 프로퍼티를 정의하는 것을 허락하지 않는다.

```typescript
type DescribableFunction = {
  description: string;
  (someArg: number): boolean;
};
function doSomething(fn: DescribableFunction) {
  console.log(fn.description + " returned " + fn(6));
}
```

매개변수 타입에 ⇒가 아닌 :를 사용해야 한다.

### 구성 시그니처

JS함수는 new연산자를 통해서도 호출될 수 있다. 새로운 객체를 생성하는데 사용되기 때문에 생성자로 간주한다. `new` 키워드를 붙임으로서 구성 시그니처를 작성할 수 있다. 

```typescript
type SomeConstructor = {
  new (s: string): SomeObject;
};
function fn(ctor: SomeConstructor) {
  return new ctor("hello");
}
```

### 제네릭 함수

입력값이 출력 값의 타입과 관련이 있거나, 두 입력값의 타입이 서로 관련이 있는 형태의 함수를 작성하는 것은 흔히 일어난다.

예) any[]

```typescript
function firstElement(arr: any[]) {
  return arr[0];
}
```

함수는 제 역할을 하지만 반환값이 any[]이다. 함수가 배열 원소의 타입을 반환한다면 더 나을 것 같다.

TS에서 제네릭 문법이 두 값 사이의 상관관계를 표현하기 위해 사용된다. 

```typescript
function firstElement<T>(arr: T[]): T | undefined {
  return arr[0];
}
```

타입 매개변수 `T`를 함수에 선언하고, 필요한 두 곳에 사용함으로써 입력값과 출력 사이에 연결고리를 만들었다. → 함수를 호출할 때 더 명확한 타입을 얻을 수 있다.

```typescript
// s는 "string" 타입
const s = firstElement(["a", "b", "c"]);
// n은 "number" 타입
const n = firstElement([1, 2, 3]);
// u는 "undefined" 타입
const u = firstElement([]); 
```

- 추론

  여기서 타입은 추론되었다. 즉 TS에 의해 자동적으로 선택되었다. 

  ```typescript
  function map<Input, Output>(arr:Input[], func: (arg:Input) => Output): Output[] {
  	return arr.map(func);
  }

  const parsed = map(["1", "2", "3"], (n) => parseInt(n));
  ```

  이 코드에서 TS는 Input 타입(string[])과 Output타입을 함수 표현식의 반환 값(number)을 통해서 추론가능하다.

- 타입 제한 조건

  가끔 두 값을 연관시키기 원하지만, 특정한 값들의 부분집합에 한해서 동작하기를 원할 때가 있다. extends절을 이용해서 타입 매개변수를 그 타입으로 제한할 수 있다.

  예) **리턴 타입을 명시하지 않았지만**, TypeScript가 추론. 제네릭 function

  ```typescript
  function longest<T extends {length : number}>(a: T, b: T) {
  	if (a.length >= b.length) {
  		return a;
  	} else {
  		return b;
  	}
  }

  const longerArray = longest([1,2], [1,2,3]); // longerArray의 타입은 number[]
  const longerString = longest("alice", "bob"); // longerArray의 타입은 string
  const notOK = longest(10,100); // Number에는 length 프로퍼티가 없음!
  ```

  T를 {length : number}로 제한했기 때문에, a와 b 매개변수에 대해서 .length 프로퍼티에 접근이 가능했다. 타입 제한이 없었다면 length 프로퍼티를 가지지 않는 다른 타입일 수 있기 떄문에 그 프로퍼티에 접근할 수 없었을 것이다. (예) 10, 100)

- 제한된 값으로 작업하기

  제네릭 타입 제약 조건을 사용할 때 흔히 있는 실수이다. 

  ```typescript
  function minimumLength<Type extends { length: number }>(
    obj: Type,
    minimum: number
  ): Type {
    if (obj.length >= minimum) {
      return obj;
    } else {
      return { length: minimum }; // Type '{ length: number; }' is not assignable to type 'Type'.
    }
  }
  ```

- 타입 인수를 명시하기

  TS는 제네릭 호출에서 의도된 타입을 대체로 추론하지만 항상 그렇지는 않다.

  ```typescript
  function combine<Type>(arr1: Type[], arr2: Type[]): Type[] {
    return arr1.concat(arr2);
  }
  ```

  ```typescript
  const arr = combine([1, 2, 3], ["hello"]);
  Type 'string' is not assignable to type 'number'.
  ```

  → 수동으로 Type을 명시해야 한다.

  ```typescript
  const arr = combine<string | number>([1, 2, 3], ["hello"]);
  ```

### 선택적 매개변수

예) number의 toFixed 메서드는 선택적 자릿수를 사용한다. 

```typescript
function f(n : number) {
	console.log(n.toFixed()); // 0 arguments
	console.log(n.toFixed(3)); // 1 arguments
}
```

예) TS에서 매개변수를 ?로 표시하면서 선택적으로 만들 수 있다.

```typescript
function f(x?: number) {
  // ...
}
f(); // OK
f(10); // OKfunction f(x?: number) {
  // ...
}
f(); // OK
f(10); // OK
```

예) 누락된 인수 흉내내기

```typescript
declare function f(x?: number): void;
// cut
// All OK
f();
f(10);
f(undefined);
```

### 함수 오버로드

JS함수는 다양한 인수의 개수, 타입을 통해서 호출될 수 있다. 

```typescript
function makeDate(timestamp: number): Date;
function makeDate(m: number, d: number, y: number): Date;
function makeDate(mOrTimestamp: number, d?: number, y?: number): Date {
  if (d !== undefined && y !== undefined) {
    return new Date(y, mOrTimestamp, d);
  } else {
    return new Date(mOrTimestamp);
  }
}
const d1 = makeDate(12345678);
const d2 = makeDate(5, 5, 5);
const d3 = makeDate(1, 3); // No overload expects 2 arguments, but overloads do exist that expect either 1 or 3 arguments.
```

해당 예시에서 두 개의 오버로드를 작성했다. 하나는 한 개의 인수를 받고, 다른 하나는 인수 세개를 받는다. 이 두 시그니쳐들을 오버로드 시그니쳐라고 한다.

그런데 필수적인 매개변수 뒤에 두 개의 선택적 매개변수를 작성했지만, 두 개의 매개변수 만으로는 함수를 호출이 불가능하다.

- 오버로드 시그니쳐와 구현 시그니쳐

  일반적인 혼동의 원인이다. 

  ```typescript
  function fn(x: string): void;
  function fn() {
    // ...
  }
  fn(); // Expected 1 arguments, but got 0.
  ```

  함수 본문을 작성하기 위해 사용된 시그니쳐는 외부에서 보이지 않는다.

  ⇒ 오버로드된 함수를 작성할 때 두 개 이상의 시그니쳐를 함수 구현 위에 작성해야 한다.

  ```typescript
  function fn(x: string): string;
  function fn(x: number): boolean; // This overload signature is not compatible with its implementation signature.

  // 아래처럼 작성되어야 한다.
  function fn(x: string | number) {
    return "oops";
  }
  ```

  결론 : 가능하면 오버로드 대신 유니온 타입 사용하기

- 함수 내에서 this 선언하기

  TS는 함수 안에서 this가 무엇이 되어야 할지 코드 흐름 분석을 통해 추론한다.

  ```typescript
  const user = {
    id: 123,
   
    admin: false,
    becomeAdmin: function () {
      this.admin = true;
    },
  };
  ```

  user.becomeAdmin이 외부객체 user에 대응하는 this를 가지고 있다. 

  ```typescript
  interface DB {
    filterUsers(filter: (this: User) => boolean): User[];
  }
   
  const db = getDB();
  const admins = db.filterUsers(function (this: User) {
    return this.admin;
  });
  ```

  예) 잘못된 예시

  ```typescript
  interface DB {
    filterUsers(filter: (this: User) => boolean): User[];
  }
   
  const db = getDB();
  // const admins = db.filterUsers(() => this.admin); // The containing arrow function captures the global value of 'this'.
  ```

  callback함수 안에서의 this는 globa/window이다. → 화살표 함수가 아닌 function으로 명시를 해야 한다.

### 알아야 할 다른 타입

- void

  return문이 없거나, 명시적으로 값을 반환하지 않을 때 추론되는 타입이다.

  ```typescript
  function noop() {
    return;
  }
  ```

- object

  object는 원시값이 아닌 모든 값을 지칭한다. **원시값** : string, number, null, undefined, symbol, bigint, boolean

  JS에서 함수 값은 객체이다. 프로퍼티가 있고, 프로토타입 체인에 Object.prototype이 있고, instanceof Object이면서, Object.keys를 호출할 수 있고, 기타 등등이 있다. 

- unknown

  unkonwn 타입은 모든 값을 나타낸다.

  any타입과 유사하지만, unkonwn타입에 어떤 것을 대입하는 것이 유효하지 않기 때문에 더 안전하다.

  ```typescript
  function f1(a: any) {
    a.b(); // OK
  }
  function f2(a: unknown) {
    a.b();
  'a' is of type 'unknown'.
  }
  ```

- never

  어떤 함수는 결코 값을 반환하지 않는다.

  ```typescript
  function fail(msg: string): never {
    throw new Error(msg);
  }
  ```

  또한 TS가 유니온에 아무것도 남지 않다고 판단했을때 또한 나타낸다.

  ```typescript
  function fn(x: string | number) {
    if (typeof x === "string") {
      // do something
    } else if (typeof x === "number") {
      // do something else
    } else {
      x; // 'never' 타입이 됨!
    }
  }
  ```

### 함수의 할당가능성

- void 반환 타입

  void 반환타입은 일반적이지는 않지만 예측할 수 있는 동작을 발생시킬 수 있다. 

  void 반환타입으로 문맥적 타이핑은 함수를 아무것도 반환하지 않도록 강제하지 않는다. 

  ```typescript
  type voidFunc = () => void;
   
  const f1: voidFunc = () => {
    return true;
  };
   
  const f2: voidFunc = () => true;
   
  const f3: voidFunc = function () {
    return true;
  };
  ```

  그리고 아래는 여전히 void타입을 유지한다.

  ```typescript
  const v1 = f1();
   
  const v2 = f2();
   
  const v3 = f3();
  ```

  이런 동작이 존재하기에, Array.prototype.push가 number를 반환하고, Array.prototype.forEach메서드가 void반환 타입을 가지는 함수를 예상함에도 아래의 코드는 유효할 수 있다. 

  ```typescript
  const src = [1, 2, 3];
  const dst = [0];
   
  src.forEach((el) => dst.push(el));
  ```

  유의점 : 리터럴 함수 정의가 void반환 값을 가지고 있다면 그 함수는 어떠한 것도 반환해서는 안된다.

  ```typescript
  function f2(): void {
    // @ts-expect-error
    return true;
  }
   
  const f3 = function (): void {
    // @ts-expect-error
    return true;
  };
  ```
