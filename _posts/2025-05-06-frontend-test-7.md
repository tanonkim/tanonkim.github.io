---
title: "Frontend Test 7. 웹 애플리케이션 통합 테스트(Next.js)"
date: 2025-05-06 22:38:00 +0900
categories: [test, frontend-test]
tags: [test, frontend-test]
---

{% raw %}
## 1. Next.js 애플리케이션 개발과 통합 테스트

git clone, npm install, open \_\_reports\_\_/jest.html

## 2. React Context와 통합 테스트

예) Context API 생성

```typescript
export type ToastState = {
  isShown: boolean;
  message: string;
  style: ToastStyle;
};

export const initialState: ToastState = {
  isShown: false,
  message: "",
  style: "succeed",
};

export const ToastStateContext = createContext(initialState);

//  
export const initialAction: ToastAction = {
  showToast: () => {},
  hideToast: () => {},
};

export const ToastActionContext = createContext(initialAction);
```

예) 최상위 컴포넌트 ToastProvider

```typescript
export const ToastProvider = ({
  children,
  defaultState,
}: {
  children: ReactNode;
  defaultState?: Partial<ToastState>;
}) => {
  const { isShown, message, style, showToast, hideToast } =
    useToastProvider(defaultState);
  return (
    <ToastStateContext.Provider value={{ isShown, message, style }}> {/* 하위 컴포넌트에서 isShown, message, style을 참조할 수 있다 */}
      <ToastActionContext.Provider value={{ showToast, hideToast }}> {/* 하위 컴포넌트에서 showToast, hideToast를 참조할 수 있다 */}
        {children}
        {isShown && <Toast message={message} style={style} />} {/* isShown이 true가 되면 표시된다 */}
      </ToastActionContext.Provider>
    </ToastStateContext.Provider>
  );
};
```

예) 하위 컴포넌트 사용 예시

```javascript
const { showToast } = useToastAction();
const onSubmit = handleSubmit(async () => {
    try {
        // ...웹 API에 값을 제출한다.
        showToast({ message: '저장됐습니다', style: 'succeed' });
    } catch (err) {
        showToast({ message: '에러가 발생했습니다', style: 'failed' });
    }
});
```

- 테스트용 컴포넌트를 만들어 인터랙션 실행하기

```typescript
const TestComponent = ({ message }: { message: string }) => {
  const { showToast } = useToastAction(); // <Toast>를 표시하기 위한 훅
  return <button onClick={() => showToast({ message })}>show</button>;
};
```

예) 테스트 코드

```typescript
test("showToast를 호출하면 Toast컴포넌트가 표시된다", async () => {
  const message = "test";
  render(
    <ToastProvider>
      <TestComponent message={message} />
    </ToastProvider>
  );
  // 처음에는 렌더링되지 않는다.
  expect(screen.queryByRole("alert")).not.toBeInTheDocument();
  await user.click(screen.getByRole("button"));
  // 렌더링됐는지 확인한다.
  expect(screen.getByRole("alert")).toHaveTextContent(message);
});
```

## 3. Next.js 라우터와 렌더링 통합 테스트

## next-router-mock

Next.js에서 라우트 부분을 테스트하려면 목 객체를 사용해야 한다.

`npm install —save-dev next-router-mock`

예) index.tsx

```typescript

function isCurrent(flag: boolean): AnchorHTMLAttributes<HTMLAnchorElement> {
    if (!flag) return {};
    return { 'aria-current': 'page' };
}


// 중략
      <Link href={`/my/posts`} legacyBehavior>
          <a
              {...isCurrent(
                  pathname.startsWith('/my/posts') && pathname !== '/my/posts/create'
              )}
          >
              My Posts
          </a>
      </Link>
```

예) 테스트 코드

```typescript
import mockRouter from "next-router-mock";

test("현재 위치는 'My Posts'이다", () => {
  mockRouter.setCurrentUrl("/my/posts");
  
  render(<Nav onCloseMenu={() => {}} />);
  const link = screen.getByRole("link", { name: "My Posts" });
  expect(link).toHaveAttribute("aria-current", "page");
});
```

### Next.js 라우터와 입력 통합 테스트

예) 테스트할 UI 컴포넌트

```javascript
?status=all : 모든 기사
?status=public : 공개 기사
?status=private : 비공개 기사
```

예) 코드

```typescript
const options = [
  { value: "all", label: "모두" },
  { value: "public", label: "공개" },
  { value: "private", label: "비공개" },
];

export const Header = () => {
  const { query, push } = useRouter();
  const defaultValue = parseAsNonEmptyString(query.status) || "all";
  return (
    <header className={styles.header}>
      <h2 className={styles.heading}>기사 목록</h2>
      <SelectFilterOption
        title="공개 여부"
        options={options}
        selectProps={{
          defaultValue,
          onChange: (event) => {
            const status = event.target.value;
            push({ query: { ...query, status } });
          },
        }}
      />
    </header>
  );
}; 
```

예) 테스트 코드 (setup)

```typescript
import userEvent from "@testing-library/user-event";

const user = userEvent.setup();

// setup
function setup(url = '/my/posts?page=1') {
    mockRouter.setCurrentUrl(url);
    render(<Header />);
    const combobox = screen.getByRole('combobox', { name: '공개 여부' });
    async function selectOption(label: string) {
        await user.selectOptions(combobox, label);
    }
    return { combobox, selectOption };
}

test("기본값으로 '모두'가 선택되어 있다", async () => {
    const { combobox } = setup();
    expect(combobox).toHaveDisplayValue('모두');
});

test("status?=public으로 접속하면 '공개'가 선택되어 있다", async () => {
    const { combobox } = setup('/my/posts?status=public');
    expect(combobox).toHaveDisplayValue('공개');
});

test("staus?=private으로 접속하면 '비공개'가 선택되어 있다", async () => {
    const { combobox } = setup('/my/posts?status=private');
    expect(combobox).toHaveDisplayValue('비공개');
});
```

`toHaveDisplayValue`

### React Hook Form으로 폼 쉽게 다루기

제어 컴포넌트 : useState 등을 사용해 컴포넌트 단위로 상태를 관리하는 컴포넌트
{% endraw %}
