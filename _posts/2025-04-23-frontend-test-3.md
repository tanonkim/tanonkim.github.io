---
title: "Frontend Test 3. Unit Test"
date: 2025-04-23 23:38:00 +0900
categories: [test, frontend-test]
tags: [test, frontend-test]
---

## 3.2 테스트 구성 요소

### 기본 형식

테스트는 jest가 제공하는 API인 test함수로 정의한다.

`expect(검증값).toBe(기대값)`

- index.ts

```javascript
export function add(a: number, b: number) {
  return a + b;
}

export function sub(a: number, b: number) {
  return a - b;
}
```

- index.test.ts

```typescript
import { add } from "."

test("add : 1 + 2 = 3", () => {
    expect(add(1, 2)).toBe(3)
})
```

### 테스트 그룹

연관성이 있는 테스트들을 그룹화하고 싶을 떄는 describe를 활용한다.

```typescript
describe('add', () => {
  test('1 + 2', () => {
    expect(add(1, 2)).toBe(3);
  });
  test('1 + 3', () => {
    expect(add(1, 3)).toBe(4);
  });
});
describe('sub', () => {
  test('1-1', () => {
    expect(sub(1, 1)).toBe(0);
  });
  test('1-3', () => {
    expect(sub(1, 3)).toBe(-2);
  });
});
```

## 3.3 테스트 실행 방법

예) package.json()

```json
{
  "name": "frontend-testing-book-unittest",
  "version": "1.0.0",
  "private": true,
  "license": "MIT",
  "scripts": {
    "test": "jest",
    ...
```

`npm test` ⇒ 모든 테스트 실행

`npm test 'src/03/02/index.test.ts'` ⇒ 지정된 파일 테스트

`jest runner` 등의 확장 프로그램을 통해 특정 테스트를 실행하면 편하다

![](/assets/img/2025-04-23/frontend-test-3-1.png)

**테스트가 성공한 경우**

![](/assets/img/2025-04-23/frontend-test-3-2.png)

**테스트가 실패한 경우**

![](/assets/img/2025-04-23/frontend-test-3-3.png)

## 3.4 조건분기

테스트는 테스트할 때 모듈이 의도대로 구현되었는지 검증할 때 도움이 된다. 사양이 복잡할 수록 조건 분기에서 버그가 많이 발생하므로 주의해서 테스트를 해야 한다.

결론 : 테스트 명을 함수 기능에 부합하는 더 적절한 이름으로 작성해야 한다.

## 3.5 에지 케이스와 예외처리

모듈을 사용할 때 실수 등의 이유로 예상하지 못한 입력값을 보낼 때가 있다. 따라서 매개변수에 Type Annotation을 붙여야 한다.

```typescript
export function add(a: number, b: number) {
  const sum = a + b;
  if (sum > 100) {
    return 100;
  }
  return sum;
}
add(1, 2);
add('1', '2'); // Type Error
```

### 예외 발생시키기

```typescript
function checkRange(value: number) {
  if (value < 0 || value > 100) {
    throw new RangeError("0〜100 사이의 값을 입력해주세요");
  }
}

export function add(a: number, b: number) {
  checkRange(a);
  checkRange(b);
  const sum = a + b;
  if (sum > 100) {
    return 100;
  }
  return sum;
}
```

- test

해당 테스트는 조건에 충족되지 않는다.

```typescript
test("반환값의 상한은 '100'이다", () => {
  expect(add(-10, 110)).toBe(100);
});
```

아래와 같이 화살표 **함수로 감싸서** 작성해야 한다. (화살표 함수를 사용하면 함수에서 예외가 발생하는지 검증이 가능하다.)

```typescript
// index.ts
function checkRange(value: number) {
  if (value < 0 || value > 100) {
    throw new RangeError("0〜100 사이의 값을 입력해주세요");
  }
}

// index.test.ts
describe("반환값의 상한은 '100'이다", () => {
  test("반환값의 상한은 '100'이다 Throw 메세지 제외", () => {
    expect(() => add(-10, 110)).toThrow();
  });
  test("반환값의 상한은 '100'이다 Throw 메세지 포함 PASS Case", () => {
    expect(() => add(-10, 110)).toThrow('0〜100 사이의 값을 입력해주세요'); // PASS
  });
  test("반환값의 상한은 '100'이다 Throw 메세지 포함 FAIL Case", () => {
    expect(() => add(-10, 110)).toThrow('10〜100 사이의 값을 입력해주세요'); // FAIL
  });
  test("반환값의 상한은 '100'이다 인스턴스 테스트 PASS Case", () => {
    expect(() => add(-10, 110)).toThrow(RangeError);
  });
});
```

![](/assets/img/2025-04-23/frontend-test-3-4.png)

## 3.6 용도별 매처

### 진리값 검증

```typescript
describe("진릿값 검증", () => {
  test("참인 진릿값 검증", () => {
    expect(1).toBeTruthy();
    expect("1").toBeTruthy();
    expect(true).toBeTruthy();
    expect(0).not.toBeTruthy();
    expect("").not.toBeTruthy();
    expect(false).not.toBeTruthy();
  });
  test("거짓인 진릿값 검증", () => {
    expect(0).toBeFalsy();
    expect("").toBeFalsy();
    expect(false).toBeFalsy();
    expect(1).not.toBeFalsy();
    expect("1").not.toBeFalsy();
    expect(true).not.toBeFalsy();
  });
  test("null과 undefined 검증", () => {
    expect(null).toBeFalsy();
    expect(undefined).toBeFalsy();
    expect(null).toBeNull();
    expect(undefined).toBeUndefined();
    expect(undefined).not.toBeDefined();
  });
});
```

### 수치 검증

toBeGreaterThan()

toBeGreaterThanOrEqual()

toBeLessThan()

toBeLessThanOrEqual()

등이 사용된다.

### 문자열 검증

toBe()

toEqual()

toContain() 일부 검증

toMatch() 정규표현식 검증

### 객체검증

```typescript
describe('객체 검증', () => {
  const author = { name: 'tanon', age: 10 };
  test('testMatchObject', () => {
    expect(author).toMatchObject({ name: 'tanon', age: 10 });
  });
  test('testMatchObject FailTest', () => {
    expect(author).toMatchObject({ name: 'Guri', age: 20 });
  });
});
```

toMatchObject()

## 3.7 비동기 처리 테스트

**Promise**

- index.ts

```typescript
export function wait(duration: number) {
  return new Promise((resolve) => {
    setTimeout(() => {
      resolve(duration);
    }, duration);
  });
}

export function timeout(duration: number) {
  return new Promise((_, reject) => {
    setTimeout(() => {
      reject(duration);
    }, duration);
  });
}
```

- index.test.ts

```typescript
// 방법1
test('지정 시간을 기다린 뒤 resolve', () => {
  return expect(wait(50)).resolves.toBe(50);
});

// 방법2
test('지정 시간을 기다린 뒤 resolve', () => {
  return wait(30).then((duration) => {
    expect(duration).toBe(30);
  });
});
```

**async/await**

```typescript
// 방법3
test('지정 시간을 기다린 뒤 resolve', async () => {
  expect(await wait(50)).toBe(50);
});


// 방법4
test('지정 시간을 기다린 뒤 resolve', async () => {
  await expect(wait(50)).resolves.toBe(50);
});
```

**reject**

```javascript
// 방법1
test('지정 시간을 기다린 뒤 resolve', () => {
  return wait(30).catch((duration) => {
    expect(duration).toBe(30);
  });
});

// 방법2
test('지정 시간을 기다린 뒤 resolve', () => {
  return expect(wait(50)).rejects.toBe(50);
});

// 방법3
test('지정 시간을 기다린 뒤 resolve', async () => {
	await expect(timeout(50)).rejects.toBe(50);
});
```

비동기 처리가 포함된 부분을 테스트 할 때는 테스트 함수를 async 함수로 만든다.

.resolves나 .rejects가 포함된 단언문은 await한다.

try-catch문의 예외 발생을 검증할 때는 expect.assertions를 사용한다.
