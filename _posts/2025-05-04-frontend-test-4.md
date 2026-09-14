---
title: "Frontend Test 4. Mock 객체"
date: 2025-05-04 15:56:00 +0900
categories: [test, frontend-test]
tags: [test, frontend-test]
---

4.5, 4.6, 4.7 다시 읽기

## 4.1 Mock 객체를 사용하는 이유

테스트는 실제 실행환경과 유사할수록 재현성이 높다. 이를 높이다 보면 시간이 너무 많이 걸리거나, 환경 구축이 어려워지는 경우가 있다.

웹 API에서 취득한 데이터를 다루어야 할 때는 성공하는 경우, 실패하는 경우도 테스트 해야 한다. 이 때 취득한 데이터 대역으로 사용할 수 있는게 Mock 객체이다.

### 4.1.1 스텁, 스파이

스텁, 스파이 등은 Mock 객체를 상황에 따라 세분화한 객체의 명칭이다.

**스텁**

스텁은 주로 대역으로 사용한다.

- 의존 중인 컴포넌트의 대역
- 정해진 값을 반환하는 용도
- 테스트 대상에 할당하는 입력값

테스트 대상이 의존 중인 컴포넌트에 테스트하기 어려운 부분이 있을 때 사용하는 예이다. 웹 API에 의존중인 대상을 테스트하는게 대표적으로, 웹 API에서 이런 값을 반환받았을 때는 이렇게 작동해야 한다와 같은 테스트에 스텁을 사용한다. 테스트 대상이 스텁에 접근하면 스텁은 정해진 값을 반환한다.

**스파이**

스파이는 주로 기록한는 용도이다.

- 함수나 메서드의 호출 기록
- 호출된 횟수나 실행 시 사용한 인수 기록
- 테스트 대상의 출력 확인

스파이는 테스트 대상 외부의 출력을 검증할 때 사용한다. 대표적인 경우가 인수로 받은 콜백함수를 검증하는 것이다. 콜백 함수가 실행된 횟수, 실행 시 사용한 인수 등을 기록하기에 의도대로 콜백이 호출됐는지 검증할 수 있다.

## 4.2 Mock 모듈을 활용한 스텁

```typescript
export function greet(name: string) {
  return `Hello! ${name}.`;
}

export function sayGoodBye(name: string) {
  throw new Error("미구현");
}
```

- greet.test.ts

```typescript
import { greet } from "./greet";

jest.mock("./greet");

test("인사말을 반환하지 않는다(원래 구현과 다르게)", () => {
  expect(greet("Taro")).toBe(undefined);
});
```

**모듈을 스텁으로 대체하기**

```typescript
import { greet, sayGoodBye } from "./greet";

// 모듈을 스텁으로 대체한다
jest.mock("./greet", () => ({
  sayGoodBye: (name: string) => `Good bye, ${name}.`,
}));

test("인사말이 구현되어 있지 않다(원래 구현과 다르게)", () => {
  expect(greet).toBe(undefined);
});

test("작별 인사를 반환한다(원래 구현과 다르게)", () => {
  const message = `${sayGoodBye("Taro")} See you.`;
  expect(message).toBe("Good bye, Taro. See you.");
});
```

sayGoodBye 함수는 Error을 반환하지만, 스텁으로 대체해 테스트가 가능하도록 만들었다.

jest.mock()에서 sayGoodBye가 구현되어있지 않다면 fail이 발생함

`jest.requireActual`함수를 사용하면 원래 모듈의 구현을 import가 가능함

**라이브러리 대체하기**

모듈을 불러오는 방법에는 ESM(ES Module)과 CJS(CommomJS Modules)가 존재한다. (해당 책은 import를 사용하는 ESM을 적용)

## 4.3 `웹 API` 목 객체 기초

웹 API 클라이언트의 대역인 스텁으로 대체하여 테스트하기

예) 테스트할 함수

```typescript
export async function getGreet() {
  // 테스트하고 싶은 것은 이 라인에서의 데이터 취득 여부와
  const data = await getMyProfile();
  // 취득한 데이터를 이 라인에서 사용할 수 있는지다.
  if (!data.name) {
    return `Hello, anonymous user!`;
  }
  return `Hello, ${data.name}!`;
}


// src/fetchets.ts
export function getMyProfile(): Promise<Profile> {
  return fetch(host("/my/profile")).then(handleResponse);
}

async function handleResponse(res: Response) {
  const data = await res.json();
  if (!res.ok) {
    throw data;
  }
  return data;
}
```

1. **웹 API 클라이언트 스텁 구현**

jest.mock 함수를 호출하여 불러온 모듈을 대체하기

```typescript
import * as Fetchers from "../fetchers";
jest.mock("../fetchers");
```

jest.spyOn으로 테스트할 객체를 대체

테스트할 객체는 import * as로 불러온 Fetchers

```javascript
jest.spyOn(Fetchers, "getMyProfile")
```

1. 데이터 취득 성공을 재현한 테스트

데이터 취득을 성공했을 때, 즉 resolve되었을 때 `mockResolvedValueOnce`에 지정한다.

```typescript
// 데이터 취득 시 사용자 이름이 없는 경우
jest.spyOn(Fetchers, "getMyProfile").mockResolvedValueOnce({
      id: "xxxxxxx-123456",
      email: "test@myapi.testing.com",
  });
```

```typescript
// 데이터 취득 시 사용자 이름이 있는 경우
  jest.spyOn(Fetchers, "getMyProfile").mockResolvedValueOnce({
      id: "xxxxxxx-123456",
      email: "taroyamada@myapi.testing.com",
      name: "taroyamada",
  });
  
 
 // 사용예시
   test("데이터 취득 성공 시: 사용자 이름이 있는 경우", async () => {
    jest.spyOn(Fetchers, "getMyProfile").mockResolvedValueOnce({
      id: "xxxxxxx-123456",
      email: "tanon@myapi.testing.com",
      name: "tanon",
    });
    await expect(getGreet()).toBe("Hello, tanon!");
  });
```

1. 데이터 취득 실패를 재현한 테스트

```typescript
// src/fetchets.ts
export function getMyProfile(): Promise<Profile> {
  return fetch(host("/my/profile")).then(handleResponse);
}

async function handleResponse(res: Response) {
  const data = await res.json();
  if (!res.ok) {
	  // 200번대 이외의 응답인 경우
    throw data;
  }
  return data;
}
```

```typescript
  test("데이터 취득 실패 시", async () => {
    // getMyProfile이 reject됐을 때의 값을 재현
    jest.spyOn(Fetchers, "getMyProfile").mockRejectedValueOnce(httpError);
    await expect(getGreet()).rejects.toMatchObject({
      err: { message: "internal server error" },
    });
  });
```

res.ok가 falsy인 경우 함수에서 예외를 발생시킨다. data와 함께 예외를 발생시키면 Promise는 reject된다

## 4.4 `웹 API` 목 객체 생성함수

```typescript
export async function getMyArticleLinksByCategory(category: string) {
  const data = await getMyArticles();
  // 취득한 데이터 중 지정한 태그를 포함한 기사만 골라낸다.
  const articles = data.articles.filter((article) =>
    article.tags.includes(category)
  );
  
  // 해당되는 기사가 없으면 null을 반환한다.
  if (!articles.length) return null;
    
  // 해당되는 기사가 있으면 목록용으로 가공해서 데이터를 반환한다.
  return articles.map((article) => ({
    title: article.title,
    link: `/articles/${article.id}`,
  }));
}
```

1. fixture_픽스처 만들기

픽스처란 응답을 재현하기 위한 테스트용 데이터를 의미한다.

```typescript
export const getMyArticlesData: Articles = {
  articles: [
    {
      id: "howto-testing-with-typescript",
      createdAt: "2022-07-19T22:38:41.005Z",
      tags: ["testing"],
      title: "타입스크립트를 사용한 테스트 작성법",
      body: "테스트 작성 시 타입스크립트를 사용하면 테스트의 유지 보수가 쉬워진다",
    },
    {
      id: "nextjs-link-component",
      createdAt: "2022-07-19T22:38:41.005Z",
      tags: ["nextjs"],
      title: "Next.js의 링크 컴포넌트",
      body: "Next.js는 화면을 이동할 때 링크 컴포넌트를 사용한다",
    },
    {
      id: "react-component-testing-with-jest",
      createdAt: "2022-07-19T22:38:41.005Z",
      tags: ["testing", "react"],
      title: "제스트로 시작하는 리액트 컴포넌트 테스트",
      body: "제스트는 단위 테스트처럼 UI 컴포넌트를 테스트할 수 있다",
    },
  ],
};
```

1.1 목 객체 생성함수를 사용하기

```typescript
function mockGetMyArticles(status = 200) {
  if (status > 299) {
    return jest.spyOn(Fetchers, "getMyArticles")
	    .mockRejectedValueOnce(httpError);
  }
  return jest.spyOn(Fetchers, "getMyArticles")
	  .mockResolvedValueOnce(getMyArticlesData);
}
```

장점 : jest.spyOn을 작성하지 않아도 된다.

1. 데이터 취득 성공을 재현한 테스트

```typescript
test("지정한 태그를 포함한 기사가 한 건도 없으면 null을 반환한다", async () => {
  mockGetMyArticles();
  const data = await getMyArticleLinksByCategory("playwright");
  expect(data).toBeNull();
});
```

```typescript
test("지정한 태그를 포함한 기사가 한 건 이상 있으면 링크 목록을 반환한다", async () => {
  mockGetMyArticles();
  const data = await getMyArticleLinksByCategory("testing");
  expect(data).toMatchObject([
    {
      link: "/articles/howto-testing-with-typescript",
      title: "타입스크립트를 사용한 테스트 작성법",
    },
    {
      link: "/articles/react-component-testing-with-jest",
      title: "제스트로 시작하는 리액트 컴포넌트 테스트",
    },
  ]);
});
```

1. 데이터 취득 실패를 재현한 테스트

```javascript
test("데이터 취득에 실패하면 reject된다", async () => {
  mockGetMyArticles(500);
  await getMyArticleLinksByCategory("testing").catch((err) => {
    expect(err).toMatchObject({
      err: { message: "internal server error" },
    });
  });
```

## 4.5 `목 함수`를 사용하는 스파이

jest.fn을 통해 목 함수를 작성한다. 테스트 코드에서 함수로 하용하며, toBeCalled 매처를 사용하면 실행 여부를 확인할 수 있다.

```typescript
test("목 함수가 실행됐다", () => {
  const mockFn = jest.fn();
  mockFn();
  expect(mockFn).toBeCalled();
});

test("목 함수가 실행되지 않았다.", () => {
	const mockFn = jest.fn();
	mockFn();
  expect(mockFn).not.toBeCalled();
});
```

예) 활용방법

```javascript
export function greet(name: string, callback?: (message: string) => void) {
  callback?.(`Hello! ${name}`);
}
```

```typescript
test("목 함수를 테스트 대상의 인수로 사용할 수 있다", () => {
  const mockFn = jest.fn();
  greet("Jiro", mockFn);
  expect(mockFn).toHaveBeenCalledWith("Hello! Jiro");
});
```
