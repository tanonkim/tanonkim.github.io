---
title: "Frontend Test 5. 컴포넌트 테스트"
date: 2025-05-05 01:57:00 +0900
categories: [test, frontend-test]
tags: [test, frontend-test]
---

## 5.1 MPA vs SPA

멀티 페이지 애플리케이션 : 여러 HTML 페이지와 HTTP 요청으로 만들어진 웹 애플리케이션

싱글 페이지 애플리케이션 : 한개의 HTML 페이지에서 개발하는 웹 애플리케이션

→ 이후에는 부분적으로 변경할 때 HTML을 변경, 변경할 때의 주요 대상이 UI 컴포넌트

SPA는 필요한 만큼만 데이터를 받기 때문에 응답이 빠르고, 데이터 획득에 사용되는 리소스도 적다

### UI 컴포넌트 테스트

UI 컴포넌트의 최소 단위는 개별 UI(버튼 등) 이다. 작은 UI를 결합하여 중간 크기의 UI 컴포넌트를 만들며, 하나씩 조합해 화면 단위의 UI를 만든다.

아래는 대표적인 UI 컴포넌트 기능이다.

- 데이터를 렌더링하는 기능
- 사용자의 입력을 전달하는 기능
- 웹 API와 연동하는 기능
- 데이터를 동적으로 변동하는 기능

### 웹 접근성 테스트

신체적/정신적 특성에 따른 차이 없이 정보에 접근할 수 있는 정도를 웹 접근성이라고 한다.

예) CSS로 input요소를 제거하는 경우 등

## 라이브러리 설치

- jest-environmnet-jsdom : jsdom : Node.js는 공식적으로 DOM API를 지원하지 않음
- @testing-library/react                   : 리액트용 테스트
- @testing-library/jest-dom             : 커스텀 매처라는 jest의 확장 기능 제공
- @testing-library/user-event          : 사용자의 입력에 가깝게 시뮬레이션이 가능한 라이브러리

@testing-library/dom : 코어로 사용되는 패키지, 여러 패키지가 의존하므로 직접 설치할 필요 X

```typescript
module.exports = {
  testEnvironment: "jest-environment-jsdom",
};
```

### 테스팅 라이브러리

UI 컴포넌트를 테스트하는 라이브러리이다.

- UI를 렌더링한다
- 렌더링된 요소에서 임의의 자식 요소를 취득한다
- 렌더링된 요소에 인터랙션을 일으킨다.

예) UI 컴포넌트

```typescript
type Props = {
  name: string;
  onSubmit?: (event: React.FormEvent<HTMLFormElement>) => void;
};
export const Form = ({ name, onSubmit }: Props) => {
  return (
    <form
      onSubmit={(event) => {
        event.preventDefault();
        onSubmit?.(event);
      }}
    >
      <h2>계정 정보</h2>
      <p>{name}</p>
      <div>
        <button>수정</button>
      </div>
    </form>
  );
};
```

```typescript
test("이름을 표시한다", () => {
  render(<Form name="taro" />);
  expect(screen.getByText("taro")).toBeInTheDocument();
});
```

jest.setup.ts에 import되었기에, 테스트 파일에 @testing-library/jest-dom을 명시적으로 import하지 않아도 된다.

## 아이템 목록 UI 컴포넌트 테스트 (Fixture Mock Data)

```typescript
import { ArticleListItem, ItemProps } from "./ArticleListItem";

type Props = {
  items: ItemProps[];
};

export const ArticleList = ({ items }: Props) => {
  return (
    <div>
      <h2>기사 목록</h2>
      {items.length ? (
        <ul>
          {items.map((item) => (
            <ArticleListItem {...item} key={item.id} />
          ))}
        </ul>
      ) : (
        <p>게재된 기사가 없습니다</p>
      )}
    </div>
  );
};

export const ArticleListItem = ({ id, title, body }: ItemProps) => {
  return (
    <li>
      <h3>{title}</h3>
      <p>{body}</p>
      <a href={`/articles/${id}`}>더 알아보기</a>
    </li>
  );
};
```

```typescript
// fixture.ts
// ArticleListItem에 목록을 표시하기 위한 배열ㅓㄷㄴㅅ
import { ItemProps } from "./ArticleListItem";

export const items: ItemProps[] = [
  {
    id: "howto-testing-with-typescript",
    title: "타입스크립트를 사용한 테스트 작성법",
    body: "테스트 작성 시 타입스크립트를 사용하면 테스트의 유지 보수가 쉬워진다",
  },
  {
    id: "nextjs-link-component",
    title: "Next.js의 링크 컴포넌트",
    body: "Next.js는 화면을 이동할 때 링크 컴포넌트를 사용한다",
  },
  {
    id: "react-component-testing-with-jest",
    title: "제스트로 시작하는 리액트 컴포넌트 테스트",
    body: "제스트는 단위 테스트처럼 UI 컴포넌트를 테스트할 수 있다",
  },
];
```

`<li>`요소는 listitem이라는 암묵적인 역할을 하기에 `getAllByRole`(”listitem”) 역할을 한다.

```typescript
test("items의 수만큼 목록을 표시한다", () => {
  render(<ArticleList items={items} />);
  expect(screen.getAllByRole("listitem")).toHaveLength(3);
});
```

### 목록에 표시할 데이터가 없는 경우

expect(list).not.toBeinTheDocument를 사용

```typescript
test("목록에 표시할 데이터가 없으면 '게재된 기사가 없습니다'를 표시한다", () => {
  // 빈 배열을 items에 할당하여 목록에 표시할 데이터가 없는 상황을 재현한다.
  render(<ArticleList items={[]} />); // items={items}, fixture로 부터 온 item이 아님
  // 존재하지 않을 것으로 예상하는 요소의 취득을 시도한다.
  const list = screen.queryByRole("list");
  // list가 존재하지 않는다.
  expect(list).not.toBeInTheDocument();
  // list가 null이다.
  expect(list).toBeNull();
  // '게재된 기사가 없습니다'가 표시됐는지 확인한다.
  expect(screen.getByText("게재된 기사가 없습니다")).toBeInTheDocument();
});
```

## 인터렉티브 UI 컴포넌트 테스트 (Input, Checkbox)

### 체크박스

체크박스를 클릭하면 onChange에서 이벤트 핸들러로 할당한 콜백 함수가 호출된다.

```typescript
type Props = {
  onChange?: React.ChangeEventHandler<HTMLInputElement>;
};

export const Agreement = ({ onChange }: Props) => {
  return (
    <fieldset>
      <legend>이용 약관 동의</legend>
      <label>
        <input type="checkbox" onChange={onChange} />
        서비스&nbsp;<a href="/terms">이용 약관</a>을 확인했으며 이에 동의합니다
      </label>
    </fieldset>
  );
};
```

`fieldset`요소는 `group`이라는 암묵적인 역할을 한다.

legend : fieldset의 하위 요소로 그룹에 제목을 붙이는데 사용

예)

```typescript
test("fieldset의 접근 가능한 이름을 legend에서 인용합니다", () => {
  render(<Agreement />);
  expect(
    screen.getByRole("group", { name: "이용 약관 동의" })
  ).toBeInTheDocument();
});
```

예) 렌더링 직후 체크되지 않음

```typescript
test("체크 박스가 체크되어 있지 않습니다", () => {
  render(<Agreement />);
  expect(screen.getByRole("checkbox")).not.toBeChecked();
});
```

### Input

문자열 입력은 실제 사용자 작동에 가깝게 입력을 재현하는 `@testing-library/user-event`를 사용한다.

1. userEvent.setup()으로 API를 호출할 user 인스턴스를 생성한다.
2. screen.getByRole로 입력란 데이터 취득하기.

```typescript
export const InputAccount = () => {
  return (
    <fieldset>
      <legend>계정정보 입력</legend>
      <div>
        <label>
          메일주소
          <input type="text" placeholder="example@test.com" />
        </label>
      </div>
      <div>
        <label>
          비밀번호
          <input type="password" placeholder="8자 이상" />
        </label>
      </div>
    </fieldset>
  );
};
```

```typescript
import userEvent from "@testing-library/user-event";


const user = userEvent.setup();

test.('메일 주소 입력', async () => {
	render(<InputAccount/>);
	
	// 메일 주소 입력
	const textbox = screen.getByRole("textbox", {name : "메일주소"});
	const value = "user@example.com";
	
	await user.type(textbox, value);
	// 초깃값이 입력된 폼 요소가 존재하는지 검증
	expect(screen.getByDisplayValue(value)).toBeInTheDocument();
})
```

주의 : `<input type=’password’/ >` 인 경우 getByRole이 적용이 안됨!

```javascript
const password = screen.getByPlaceholderText("8자 이상");
```

## 비동기 처리가 포함된 UI 컴포넌트 테스트 (112p`다시보기`)

예) 예제 코드

```typescript
export const RegisterAddress = () => {
  const [postResult, setPostResult] = useState("");
  return (
    <div>
      <Form
        onSubmit={handleSubmit((values) => {
          try {
            checkPhoneNumber(values.phoneNumber);
            postMyAddress(values)
              .then(() => {
                setPostResult("등록됐습니다");
              })
              .catch(() => {
                setPostResult("등록에 실패했습니다");
              });
          } catch (err) {
            if (err instanceof ValidationError) {
              setPostResult("올바르지 않은 값이 포함되어 있습니다");
              return;
            }
            setPostResult("알 수 없는 에러가 발생했습니다");
          }
        })}
      />
      {postResult && <p>{postResult}</p>}
    </div>
  );
};
```

1. 웹 API 클라이언트의 목 함수

```typescript
import * as Fetchers from ".";
import { httpError, postMyAddressMock } from "./fixtures";

export function mockPostMyAddress(status = 200) {
  if (status > 299) {
    return jest
      .spyOn(Fetchers, "postMyAddress")
      .mockRejectedValueOnce(httpError);
  }
  return jest
    .spyOn(Fetchers, "postMyAddress")
    .mockResolvedValueOnce(postMyAddressMock);
}

// fetchers/index.ts
export function postMyAddress(values: unknown): Promise<Result> {
  return fetch(host("/my/address"), {
    method: "POST",
    body: JSON.stringify(values),
    headers,
  }).then(handleResponse);
}


export const httpError: HttpError = {
  err: { message: "internal server error" },
};

export const postMyAddressMock: Result = {
  result: "ok",
};
```

## 암묵적 역할과 접근가능한 이름

`getByRole` 은 HTML의 요소를 참조한다. 역할은 W3C의 WAI-ARIA라는 사양에 포함된 내용 중 하나이다.

1. button태그의 role : button

2. input

   text → rold : textbox

   checkbox : checkbox

   radia : radio

   number :  spinbutton

3. 스크린리더 : 접근 가능한 이름을 읽어서 조작할 수 있는 기능

   `getByRole(”button”, {name : “전송”});`

   ```javascript
   <button>전송</button>
   <button><img alt="전송" src="./image.png"/></button>
   ```
