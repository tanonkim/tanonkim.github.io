---
title: "React 공식문서 - React 19 - Reference"
date: 2025-08-13 18:06:00 +0900
categories: [react, react-docs]
tags: [react]
---

# React

## Hook

### 내장 React Hook

### State Hooks

- useState
- useReducer : Reducer함수 내부의 업데이트 로직을 사용해서 State 변수를 선언함

### Context Hooks

- useContext : Props를 전달하지 않고도 멀리 있는 부모 컴포넌트로부터 정보를 전달받게 한다.

  ```typescript
  function Button() {
    const theme = useContext(ThemeContext);
    // ...
  ```

### Ref Hooks

- useRef : 렌더링에 사용되지 않는 일부 정보를 보유할 수 있다. 대부분 DOM 노드를 담는데 사용된다.

  ```typescript
  function Form() {
    const inputRef = useRef(null);
    // ...
  ```

### Effect Hooks

- useEffect : Effect를 통해 컴포넌트를 외부 시스템에 연결하고 동기화할 수 있다.

### Performance Hooks

- useMemo : 비용이 많이 드는 계산 결과를 캐시할 수 있다.
- useCallback : 함수 정의를 최적화된 컴포넌트에 전달하기 전에 캐시할 수 있다.

### `useActionState`

- 폼 액션이 실행될 때 업데이트되는 컴포넌트

```typescript
const [state, formAction, isPending] = useActionState(fn, initialState, permalink?);
const [state, formAction, isPending] = useActionState(updateUserInfo, null); // action 함수, 초기값

return (
	<form action={formAction} className="flex items-start flex-col w-full gap-8 text-base font-semibold">
		...중략
	</form>
)
```

**예) 서버함수 updateUserInfo**

```typescript
'use server';

export async function updateUserInfo(_: any, formData: FormData) {
  const name = formData.get('name')?.toString();
  const email = formData.get('user_email')?.toString();

  if (!name || !email)
    return {
      status: false,
      error: 'Not valid data',
    };

  const token = await getAccessToken();

  await fetch(`${process.env.NEXT_PUBLIC_API_URL}/users/me`, {
    method: 'PATCH',
    headers: {
      // eslint-disable-next-line @typescript-eslint/naming-convention
      'Content-Type': 'application/json',
      Authorization: 'Bearer ' + token,
    },
    body: JSON.stringify({
      real_name: name,
      email: email,
    }),
  });

  return {
    status: true,
    error: '',
  };
}
```

### `useCallback`

리렌더링 간에 함수 정의를 캐싱해주는 Hook

```typescript
const cachedFn = useCallback(fn, dependencies)
```

```typescript
  const handleRoutes = useCallback(() => {
    if (data.link_to.startsWith('product')) {
      const productId = data.link_to.split('product/')[1];
      router.push(`/store/${productId}`);
    }
    if (data.link_to.startsWith('leaderboard')) {
      router.push(`/leaderboard?type=card`);
    }
  }, [data.link_to, router]);
```

사용법 :

- 컴포넌트의 리렌더링 건너뛰기
- Memoized 콜백에서 상태 업데이트 하기
- Effect가 너무 자주 실행되는 것을 방지하기
- 커스텀 Hook 최적화하기

**왜 사용하는가?**

```typescript
function ProductPage({ productId, referrer, theme }) {
  // React에게 리렌더링 간에 함수를 캐싱하도록 요청합니다...
  const handleSubmit = useCallback((orderDetails) => {
    post('/product/' + productId + '/buy', {
      referrer,
      orderDetails,
    });
  }, [productId, referrer]); // 의존성이 변경되지 않는 한

  return (
    <div className={theme}>
      {/* ...ShippingForm은 같은 props를 받게 되고 리렌더링을 건너뛸 수 있습니다.*/}
      <ShippingForm onSubmit={handleSubmit} />
    </div>
  );
}
```

- useMemo와 useCallback의 차이점

  [https://ko.react.dev/reference/react/useCallback#how-is-usecallback-related-to-usememo](https://ko.react.dev/reference/react/useCallback#how-is-usecallback-related-to-usememo)

### `useContext`

```typescript
const value = useContext(SomeContext)
```

컴포넌트에서 Context를 읽고 구독할 수 있는 Hook

사용법

- 트리의 깊은 곳에 데이터 전달하기

### useDebugValue

React DevTools에서 커스텀 훅에 라벨을 추가하게 하는 React Hook

```typescript
import { useDebugValue } from 'react';

function useOnlineStatus() {
  // ...
  useDebugValue(isOnline ? 'Online' : 'Offline');
  // ...
}
```

Notice : 모든 커스텀 Hook에 디버그 값을 추가하지 마세요. 이는 공유 라이브러리의 일부이고 내부 구조가 복잡하여 검사하기 어려운 커스텀 Hook에 가장 유용합니다.

### useDefferedValue

일부 UI 업데이트를 지연할 수 있는 Hook

```typescript
import { useState, useDeferredValue } from 'react';

function SearchPage() {
  const [query, setQuery] = useState('');
  const deferredQuery = useDeferredValue(query);
  // ...
}
```

컴포넌트 상위 레벨에서 useDeferredValue를 호출하여 지연된 버전의 값을 가져옴

사용법

- 새 콘텐츠가 로딩되는 동안 오래된 콘텐츠 표시하기
- 콘텐츠가 오래되었음을 표시하기
- UI일부에 대해 리렌더링 지연하기 with memo(function () {})

```typescript
export default function App() {
  const [query, setQuery] = useState('');
  const deferredQuery = useDeferredValue(query);
  const isStale = query !== deferredQuery; // stale인지에 대한 여부 확인

  return (
    <>
      <label>
        Search albums:
        <input value={query} onChange={e => setQuery(e.target.value)} />
      </label>
      <Suspense fallback={<h2>Loading...</h2>}>
        <SearchResults query={deferredQuery} />
      </Suspense>
    </>
  );
}

import {use} from 'react';
import { fetchData } from './data.js';

export default function SearchResults({ query }) {
  if (query === '') {
    return null;
  }
  const albums = use(fetchData(`/search?q=${query}`)); // 이 코드 유심히 보기
  if (albums.length === 0) {
    return <p>No matches for <i>"{query}"</i></p>;
  }
  return (
    <ul>
      {albums.map(album => (
        <li key={album.id}>
          {album.title} ({album.year})
        </li>
      ))}
    </ul>
  );
}

```

### useId

접근성 attribute에 전달할 수 있는 고유 ID를 생성하기 위한 React Hook

```typescript
import { useId } from 'react';

function PasswordField() {
  const passwordHintId = useId();
  return (
    <>
      <label>
        Password:
        <input
          type="password"
          aria-describedby={passwordHintId}
        />
      </label>
      <p id={passwordHintId}>
        The password should contain at least 18 characters
      </p>
    </>
  );
}

export default function App() {
  return (
    <>
      <h2>Choose password</h2>
      <PasswordField />
      <h2>Confirm password</h2>
      <PasswordField />
    </>
  );
}

```

사용법 :

- 접근성 attribute를 위한 고유 ID 생성하기
- 여러개의 연관된 엘리먼트의 ID 생성하기
- 생성된 모든 ID에 대해 공유 접두사 지정하기
- 클라이언트와 서버에서 동일한 ID 접두사 사용하기

주의점 : 리스트의 key는 데이터로부터 생성해야 한다.

### useImperativeHandle

Ref로 노출되는 핸들을 사용자가 직접 정의할 수 있게 해주는 Hook

가정 : `<input>` DOM 노드를 노출하지 않고 focus와 scollIntoView의 두 메서드만을 노출하고 싶다

```typescript
import { useImperativeHandle } from 'react';

function MyInput({ ref }) {
	const inputRef = useRef(null);

  useImperativeHandle(ref, () => {
    return {
				focus() {
					inputRef.current.focus();
				},
				scrollIntoView() {
					inputRef.current.scrollIntoView();
				}
    };
  }, []);

	return <input ref={inputRef}/>;
};
```

```typescript
import { useRef } from 'react';
import MyInput from './MyInput.js';

export default function Form() {
  const ref = useRef(null);

  function handleClick() {
    ref.current.focus();
    // 이 작업은 DOM 노드가 노출되지 않으므로 작동하지 않습니다.
    // ref.current.style.opacity = 0.5;
  }

  return (
    <form>
      <MyInput placeholder="Enter your name" ref={ref} />
      <button type="button" onClick={handleClick}>
        Edit
      </button>
    </form>
  );
}

```

→ 부모 컴포넌트 Form이 MyInput에 대한 Ref를 가져오면 focus 및 scrollIntoView 메서드 호출이 가능하다.

사용법

- 부모 컴포넌트에 커스텀 Ref 핸들 노출
- 사용자 정의 Imperative 메서드 노출

참고 : React 19부터 ref를 Prop으로 받을 수 있다.

### useInsertionEffect

`참고` : useInsertionEffect는 CSS-in-JS 라이브러리 작성자를 위한 것이다.

CSS-in-JS 라이브러리 작업 중에 스타일을 주입할 위치가 필요한게 아니라면 useEffect또는 useLayoutEffect를 사용

예시

```typescript
// CSS-in-JS 라이브러리 안에서
let isInserted = new Set();
function useCSS(rule) {
  useInsertionEffect(() => {
    // 앞서 설명했듯이 <style> 태그의 런타임 주입은 권장하지 않습니다.
    // 하지만 꼭 주입해야 한다면 useInsertionEffect에서 주입하는 것이 중요합니다.
    if (!isInserted.has(rule)) {
      isInserted.add(rule);
      document.head.appendChild(getStyleForRule(rule));
    }
  });
  return rule;
}

function Button() {
  const className = useCSS('...');
  return <div className={className} />;
}
```

- 사용법

  CSS-in-JS 라이브러리에서 동적 스타일 주입하기

useEffect와 마찬가지로 useInsertionEffect는 서버에서 실행되지 않는다.

### useLayoutEffect

주의사항 : useLayoutEffect를 사용하면 성능이 저하될 수 있다. → 가능하면 useEffect를 사용해야 한다.

```typescript
import { useState, useRef, useLayoutEffect } from 'react';

function Tooltip() {
  const ref = useRef(null);
  const [tooltipHeight, setTooltipHeight] = useState(0); // 실제 높이를 모름

  useLayoutEffect(() => {
    const { height } = ref.current.getBoundingClientRect(); // 높이를 알았으니 다시 렌더링
    setTooltipHeight(height);
  }, []);
  // ...
```

사용법

- 브라우저가 화면을 다시 그리기 전에 레이아웃 계산하기

(useEffect와는 달리 실제로 보이는건 최종 결과 뿐임)

### `useMemo`

useMemo는 재렌더링 사이에 계산 반환값(결과)을 캐싱할 수 있게 해주는 React Hook

**`useMemo(calculateValueFunction, dependencies)`**

```typescript
import { useMemo } from 'react';

function TodoList({ todos, tab, theme }) {
  const visibleTodos = useMemo(() => filterTodos(todos, tab), [todos, tab]);
  // ...
}
```

콘솔로그로 1ms이상 계산을 확인 → useMemo로 계산을 감싸서 상호작용이 얼마나 줄어들었는지 확인

사용법 :

- 비용이 높은 로직의 재계산 생략하기
- 컴포넌트 재렌더링 건너뛰기
- Effect가 자주 실행되지 않도록 하기
- 다른 Hook의 종속성 메모화
- 함수 메모화

### useOptimistic

`useOptimistic` 는 비동기 작업이 진행중일 때 다른 상태를 보여줄 수 있게 해주는 Hook

```jsx
import { useOptimistic, useState, useRef, startTransition } from "react";
import { deliverMessage } from "./actions.js";

function Thread({ messages, sendMessageAction }) {
  const formRef = useRef();
  function formAction(formData) {
    addOptimisticMessage(formData.get("message"));
    formRef.current.reset();
    startTransition(async () => {
      await sendMessageAction(formData);
    });
  }
  const [optimisticMessages, addOptimisticMessage] = useOptimistic(
    messages,
    (state, newMessage) => [
      {
        text: newMessage,
        sending: true
      },
      ...state,
    ]
  );

  return (
    <>
      <form action={formAction} ref={formRef}>
        <input type="text" name="message" placeholder="Hello!" />
        <button type="submit">Send</button>
      </form>
      {optimisticMessages.map((message, index) => (
        <div key={index}>
          {message.text}
          {!!message.sending && <small> (Sending...)</small>}
        </div>
      ))}
      
    </>
  );
}

export default function App() {
  const [messages, setMessages] = useState([
    { text: "Hello there!", sending: false, key: 1 }
  ]);
  async function sendMessageAction(formData) {
    const sentMessage = await deliverMessage(formData.get("message"));
    startTransition(() => {
      setMessages((messages) => [{ text: sentMessage }, ...messages]);
    })
  }
  return <Thread messages={messages} sendMessageAction={sendMessageAction} />;
}

```

### useReducer - 채우기

컴포넌트에 `reducer`₩를 추가하는 Hook

reducer : state가 여러 이벤트 핸들러로 분산되는 경우 → state를 업데이트 하는 로직을 reducer로 사용해 통합해서 관리

### useRef

렌더링에 필요하지 않은 값을 참조할 수 있는 React Hook

```jsx
import { useRef } from 'react';

function Stopwatch() {
  const intervalRef = useRef(0);

  intervalRef.current = intervalId;

```

1. 컴포넌트의 최상위 레벨에서 useRef를 호출하여 ref를 선언
2. useRef는 처음에 초기값으로 설정된 current 프로퍼티가 있는 ref객체를 반환함

사용법

- ref로 값 참조하기
- ref로 DOM 조작하기
- ref로 콘텐츠 재생성 피하기

### useState

컴포넌트에 state 변수를 추가할 수 있는 React Hook

사용법

- 컴포넌트에 state 추가하기
- 이전 state를 기반으로 state 업데이트 하기
- 객체 및 배열 state 업데이트 하기
- 이전 렌더링에서 얻은 정보 저장
- key로 state 초기화

참고 - 에러

**에러가 발생했습니다: “리렌더링 횟수가 너무 많습니다”**

```jsx
// 🚩 잘못된 방법: 렌더링 동안 핸들러 요청
return <button onClick={handleClick()}>Click me</button>

// ✅ 올바른 방법: 이벤트 핸들러로 전달
return <button onClick={handleClick}>Click me</button>

// ✅ 올바른 방법: 인라인 함수로 전달
return <button onClick={(e) => handleClick(e)}>Click me</button>
```

### useSyncExternalStore - 채우기

외부 store를 구독할 수 있는 React Hook

### useTransition

UI의 일부를 백그라운드에서 렌더링 할 수 있도록 해주는 React Hook이다.

## Component

### Fragment

래퍼 노드 없이 엘리먼트를 그룹화 가능

사용법

- 여러 엘리먼트 반환하기
- 변수와 여러 엘리먼트 할당
- 텍스트와 함께 엘리먼트 그룹화

### `<Profiler>`

React 트리의 렌더링 성능을 프로그래밍 방식으로 측정 가능

```jsx
<Profiler id="App" onRender={onRender}>
  <App />
</Profiler>
```

사용법

- 프로그래밍 방식으로 렌더링 성능 측정
- 애플리케이션의 부분별 측정

**참고) onRender**

`onRender`: 프로파일링된 트리 내의 컴포넌트가 업데이트될 때마다 React가 호출하는 [`onRender`](https://ko.react.dev/reference/react/Profiler#onrender-callback)[ 콜백](https://ko.react.dev/reference/react/Profiler#onrender-callback). 렌더링된 내용과 소요된 시간에 대한 정보를 받는다.

```jsx
function onRender(id, phase, actualDuration, baseDuration, startTime, commitTime) {
  // 렌더링 시간 집계 혹은 로그...
}
```

### `<StrictMode>`

개발 중에 컴포넌트에서 일반적인 버그를 빠르게 찾을 수 있도록 한다.

```jsx
import { StrictMode } from 'react';
import { createRoot } from 'react-dom/client';

const root = createRoot(document.getElementById('root'));
root.render(
  <StrictMode>
    <App />
  </StrictMode>
);
```

동작

- 컴포넌트가 순수하지 않은 렌더링으로 인한 버그를 찾기 위해 추가로 다시 렌더링
- Effect 클린업이 누락되어 발생한 버그를 찾기 위해 Effect를 다시 실행
- Ref 클린업이 누락되어 발생한 버그를 찾기 위해 Ref 콜백을 다시 실행
- 더이상 사용되지 않는 API를 사용하는지 확인

### `<Suspense>`

자식 요소를 로드하기 전까지 화면에 대체 UI : Fallback를 보여준다.

```jsx
<Suspense fallback={<Skeleton />}>
  <SomeComponent />
</Suspense>
```

**Suspense가 가능한 데이터**

- Relay, Next.js와 같이 Suspense가 가능한 프레임워크를 사용한 데이터 가져오기
- lazy를 활용한 지연 로딩 컴포넌트
- use를 사용해서 캐시된 Promise값 읽기

### `<Activity>`

참고 : 실험 버전(프로덕션 사용 X)

일부 UI를 숨기거나 보여줄 수 있다.

```jsx
import {unstable_Activity as Activity} from 'react';

<Activity mode={isVisible ? 'visible' : 'hidden'}>
  <Page />
</Activity>
```

## API

### act

“테스트” 헬퍼로, 대기중인 React 업데이트를 모두 적용한 뒤 Assert(단언)할 수 있게 도와준다.

```javascript
it ('renders with button disabled', async () => {
  await act(async () => {
    root.render(<TestComponent />)
  });
  expect(container.querySelector('button')).toBeDisabled();
});
```

### cache

cache를 통해 가져온 데이터나 연산의 결과를 캐싱한다. + cache는 오직 React `서버` 컴포넌트와 함께 사용한다.

```javascript
import {cache} from 'react';
import calculateUserMetrics from 'lib/user';

const getUserMetrics = cache(calculateUserMetrics);

function Profile({user}) {
  const metrics = getUserMetrics(user);
  // ...
}

function TeamReport({users}) {
  for (let user in users) {
    const metrics = getUserMetrics(user);
    // ...
  }
  // ...
}
```

사용법

- 고비용 연산 캐싱하기
- 데이터의 스냅샷 공유하기
- 사전에 데이터 받아두기

참고 : 입력을 기반으로 반환 값 캐싱을 최적화하는 것을 메모이제이션이라고 함

`비교`

```javascript
- useMemo
	클라이언트 컴포넌트에서 렌더링에 걸쳐 고비용의 계산을 캐싱

- cache
	서버 컴포넌트에서 컴포넌트 간에 공유할 수 있는 작접을 메모화

- memo
	프로퍼티가 변경되지 않았을 때 컴포넌트가 렌더링되는 것을 막기 위해 사용
```

### captureOwnerStack - 채우기

### createContext - context 의미 파악하기

컴포넌트가 Context를 제공하거나 읽을 수 있다.

```javascript
import { createContext } from 'react';

const ThemeContext = createContext('light');

function App() {
  const [theme, setTheme] = useState('light');
  // ...
  return (
    <ThemeContext value={theme}>
      <Page />
    </ThemeContext>
  );
}
```

사용법

- 컨텍스트 생성
- 파일에서 컨텍스트 가져오기 및 내보내기

### lazy

컴포넌트가 처음 렌더링될 때까지 해당 컴포넌트의 코드를 로딩하는 것을 지연

```javascript
import { lazy } from 'react';

const MarkdownPreview = lazy(() => import('./MarkdownPreview.js'));

<Suspense fallback={<Loading />}>
  <h2>Preview</h2>
  <MarkdownPreview />
</Suspense>
```

사용법

- Suspense와 지연 로딩 컴포넌트

### memo

`memo`를 사용하면 컴포넌트의 `Props`가 변경되지 않은 경우 리렌더링을 건너뛸 수 있다.

```javascript
import { memo, useState } from 'react';

export default function MyApp() {
  const [name, setName] = useState('');
  const [address, setAddress] = useState('');
  return (
    <>
      <label>
        Name{': '}
        <input value={name} onChange={e => setName(e.target.value)} />
      </label>
      <label>
        Address{': '}
        <input value={address} onChange={e => setAddress(e.target.value)} />
      </label>
      <Greeting name={name} />
    </>
  );
}

const Greeting = memo(function Greeting({ name }) {
  console.log("Greeting was rendered at", new Date().toLocaleTimeString());
  return <h3>Hello{name && ', '}{name}!</h3>;
});

```

사용법

- Props가 변경되지 않았을 때 리렌더링 건너뛰기
- State를 사용해 메모이제이션 된 컴포넌트 업데이트 하기
- Context를 사용해 메모화된 컴포넌트 업데이트하기 → Context가 변경중일 때는 컴포넌트는 리렌더링 됨.
- Props 변경 최소화하기
- 사용자 정의 비교 함수 지정하기

### startTransition

UI의 일부를 백그라운드에서 렌더링할 수 있다.

```javascript
import { startTransition } from 'react';

function TabContainer() {
  const [tab, setTab] = useState('about');

  function selectTab(nextTab) {
    startTransition(() => {
      setTab(nextTab);
    });
  }
  // ...
}
```

사용법

- State 업데이트를 Non-Blocking Transition으로 표시

참고

`startTransition`은 [**`useTransition`**](https://ko.react.dev/reference/react/useTransition)과 매우 유사하지만, Transition이 대기 중인지 추적하는 `isPending` 플래그를 제공하지 않는다.

### use

Promise나 Context와 같은 데이터를 참조하는 React API이다.

```javascript
import { use } from 'react';

function MessageComponent({ messagePromise }) {
  const message = use(messagePromise);
  const theme = use(ThemeContext);
  // ...
```

사용법

- use를 사용하여 Context 참조하기
- 서버에서 클라이언트로 데이터 스트리밍하기
- 거부된 Promise 처리하기

예) 데이터 스트리밍 하기

```javascript
import { fetchMessage } from './lib.js';
import { Message } from './message.js';

export default function App() {
  const messagePromise = fetchMessage();
  return (
    <Suspense fallback={<p>waiting for message...</p>}>
      <Message messagePromise={messagePromise} />
    </Suspense>
  );
}


"use client";

import { use, Suspense } from "react";

function Message({ messagePromise }) {
  const messageContent = use(messagePromise);
  return <p>Here is the message: {messageContent}</p>;
}

export function MessageContainer({ messagePromise }) {
  return (
    <Suspense fallback={<p>⌛Downloading message...</p>}>
      <Message messagePromise={messagePromise} />
    </Suspense>
  );
}

```

예) 거부된 Promise 처리하기

```javascript
"use client";

import { use, Suspense } from "react";
import { ErrorBoundary } from "react-error-boundary";

export function MessageContainer({ messagePromise }) {
  return (
    <ErrorBoundary fallback={<p>⚠️Something went wrong</p>}>
      <Suspense fallback={<p>⌛Downloading message...</p>}>
        <Message messagePromise={messagePromise} />
      </Suspense>
    </ErrorBoundary>
  );
}

function Message({ messagePromise }) {
  const content = use(messagePromise);
  return <p>Here is the message: {content}</p>;
}
```

promise가 거부될 때 오류를 표시하고 싶다면 Error Boundary를 사용.

use에 전달된 Promise가 거부되면 Error Boundary에 대한 Fallback이 표시된다.

---

# Reat DOM

## Hook

### useFormStatus()

Hook은 마지막 폼 제출의 상태 정보를 제공한다.

```javascript
const { pending, data, method, action } = useFormStatus();
```

```javascript
import { useFormStatus } from "react-dom";
import action from './actions';

function Submit() {
  const status = useFormStatus();
  return <button disabled={status.pending}>Submit</button>
}

export default function App() {
  return (
    <form action={action}>
      <Submit />
    </form>
  );
}
```

## Component

## API

### createPortal

일부 자식을 DOM의 다른 부분으로 렌더링 할 수 있다.

```javascript
<div>
  <SomeComponent />
  {createPortal(children, domNode, key?)}
</div>

<div>
  <p>This child is placed in the parent div.</p>
  {createPortal(
    <p>This child is placed in the document body.</p>,
    document.body // DOM노드의 위치
  )}
</div>
```

사용법

- DOM의 다른 부분으로 렌더링하기
- Portal이 있는 모달 상자 렌더링하기
- React 컴포넌트를 React가 아닌 서버 마크업으로 렌더링하기
- React 컴포넌트를 React가 아닌 DOM 노드로 렌더링하기

예) Portal이 있는 모달 상자 렌더링하기;

```javascript
import NoPortalExample from './NoPortalExample';
import PortalExample from './PortalExample';

export default function App() {
  return (
    <>
      <div className="clipping-container">
        <NoPortalExample  />
      </div>
      <div className="clipping-container">
        <PortalExample />
      </div>
    </>
  );
}

import { useState } from 'react';
import { createPortal } from 'react-dom';
import ModalContent from './ModalContent.js';

export default function PortalExample() {
  const [showModal, setShowModal] = useState(false);
  return (
    <>
      <button onClick={() => setShowModal(true)}>
        Show modal using a portal
      </button>
      {showModal && createPortal(
        <ModalContent onClose={() => setShowModal(false)} />,
        document.body
      )}
    </>
  );
}
```

### flushSync

React에 제공된 콜백 내부의 모든 업데이트를 동기적으로 처리하도록 강제한다. → DOM이 즉시 업데이트되는 것을 보장한다.

주의 : `flushSync`를 사용하는 것은 일반적이지 않으며 애플리케이션의 성능을 저하할 수 있다. (대부분의 경우 `flushSync`의 사용을 권장하지 않는다. `flushSync`는 최후의 수단으로 사용해야 한다.)

애플리케이션이 React API만 사용하고 서드파티 라이브러리와 통합하지 않는다면 flushSync는 필요하지 않는다.

### preconnect

리소스를 가져올 것으로 예상하는 서버에 연결할 수 있다.

사용법

- 렌더링 시 사전 연결
- 이벤트 핸들러에서 사전 연결

```javascript
import { preconnect } from 'react-dom';

function AppRoot() {
  preconnect("https://example.com");
  return ...;
}
```

```javascript
import { preconnect } from 'react-dom';

function CallToAction() {
  const onClick = () => {
    preconnect('http://example.com');
    startWizard();
  }
  return (
    <button onClick={onClick}>Start Wizard</button>
  );
}
```

### prefetchDNS

리소스를 가져올 것으로 예상하는 서버의 IP를 미리 조회할 수 있게 한다.

```javascript
import { prefetchDNS } from 'react-dom';

function AppRoot() {
  prefetchDNS("https://example.com");
  // ...
}
```

### preinit

스타일 시트나 외부 스크립트를 미리 가져오고 실행할 수 있게 한다.

- 리소스가 다운되자마자 바로 실행/적용 → preinit
- 리소스가 다운하돼 바로 삽입하지 않음 → preload

참고 : React 기반 프레임워크는 리소스 로딩을 자동으로 처리하므로 해당 API를 직접 호출하지 않아도 된다.

사용법

- 렌더링 시 preinit하기
- 이벤트 핸들러 내에서 preinit하기

```javascript
import { preinit } from 'react-dom';

function AppRoot() {
  preinit("https://example.com/script.js", {as: "script"});
  return ...;
}

import { preinit } from 'react-dom';

function AppRoot() {
  preinit("https://example.com/style.css", {as: "style", precedence: "medium"});
  return ...;
}
```

### preinitModule

ESM 모듈을 미리 가져오고 평가할 수 있게 해준다. (ESM 모듈이 아닌 스크립트 → preinit)

참고 : React 기반 프레임워크에서는 리소스 로딩을 자동으로 처리해주는 경우가 많기에 해당 API를 직접 호출할 필요가 없을 수도 있다.

**ESM, ES Module :** ES6에 도입된 모듈 시스템 → import, export를 통해 분리된 자바스크립트 파일끼리 접근이 가능하다.

ES Module의 탄생 :

1. 초기에는 큰 스크립트가 필요하지 않음 → jQuery가 생겨나고 어플리케이션의 규모가 커지면서 script파일을 나누기 시작했고, 파일간의 변수/함수를 전달하고 받는 방법이 필요해짐
2. ESM 이전에는 script파일을 전역 스코프처럼 사용함 (HTML파일보다 위에 존재)

   → jQuery script를 최상단에 두고 순서를 구성

   ⇒ 파일 순서가 뒤틀리면 에러가 발생, 전역 오염 발생 가능성

사용법

- 렌더링 중 사전 로딩하기
- 이벤트 핸들러에서 사전 로딩하기

```javascript
import { preinitModule } from 'react-dom';

function AppRoot() {
  preinitModule("https://example.com/module.js", {as: "script"});
  // ...
}
```

### preload

사용할 스타일 시트, 글꼴, 외부 스크립트와 같은 리소스를 미리 가져올 수 있다.

```javascript
import { preload } from 'react-dom';

function AppRoot() {
  preload("https://example.com/script.js", {as: "script"});
  preload("https://example.com/style.css", {as: "style"});
  preload("https://example.com/font.woff2", {as: "font"});
  preload("/banner.png", {
    as: "image",
    imageSrcSet: "/banner512.png 512w, /banner1024.png 1024w",
    imageSizes: "(max-width: 512px) 512px, 1024px",
  });
  // ...
}
```

참고 : React 기반 프레임워크는 리소스 로딩을 대신 처리하는 경우가 많으므로 이 API를 직접 호출할 필요가 없을 수도 있다.

### preloadModule

사용할 ESM 모듈을 미리 가져올 수 있다.

```javascript
import { preloadModule } from 'react-dom';

function AppRoot() {
  preloadModule("https://example.com/module.js", {as: "script"});
  // ...
}
```

## Client API

### createRoot

브라우저 DOM 노드 안에 React 컴포넌트를 표시하는 루트를 생성할 수 있다.

사용법

- 온전히 React만으로 작성된 앱 렌더링하기
- React로 부분적으로 작성된 페이지 렌더링하기
- Root 컴포넌트 렌더링하기
- 프로덕션 환경에서 오류 로깅하기

```javascript
import { createRoot } from 'react-dom/client';

const domNode = document.getElementById('root');
const root = createRoot(domNode);

root.render(<App />);
```

domNode에 대한 루트를 생성 → root.render로 React 컴포넌트 표시

⇒ `createRoot`는 [`render`](https://ko.react.dev/reference/react-dom/client/createRoot#root-render)와 [`unmount`](https://ko.react.dev/reference/react-dom/client/createRoot#root-unmount) 두 가지 메서드를 포함한 객체를 반환한다.

### hydrateRoot

이전에 react-dom/server로 생성된 HTML 콘텐츠를 가진 브라우저 DOM 노드 안에 React 컴포넌트를 표시할 수 있게 해준다.

사용법

- 서버에서 렌더링된 HTML을 Hydrate하기
- document 전체를 Hydrate하기
- Hydrate 불일치 오류 억제하기
- 서로 다른 클라이언트와 서버 컨텐츠 다루기
- Hydrate된 루트 컴포넌트 업데이트하기
- 프로덕션 환경에서 오류 로깅하기

```javascript
import './styles.css';
import { hydrateRoot } from 'react-dom/client';
import App from './App.js';

hydrateRoot(
  document.getElementById('root'),
  <App />
);

hydrateRoot(document, <App />); // document 전체를 Hydrate
```

domNode : 서버에서 루트 요소로 렌더링된 DOM 요소

reactNode : 기존 HTML로 렌더링하기 위한 React 노드. 주로 ReactDOM Server의 renderToPipeableStream(`<App />`)과 같은 메서드로 렌더링된 `<App />`과 같은 JSX 조각

## Server API

### renderToPipeableStream

React 트리를 파이프 가능한 Node.js 스트림으로 렌더링한다.

참고 : 이 API는 Node.js 전용이다. Deno 등 Web Stream의 환경에서는 renderToReadableStream을 대신 사용해야 한다.

```javascript
const { pipe, abort } = renderToPipeableStream(<App />, {
  bootstrapScripts: ['/main.js'],
  onShellReady() {
    response.setHeader('content-type', 'text/html');
    pipe(response);
  },
  onError(error) {
    console.error(error);
    logServerCrashReport(error);
  }
});

// 서버 렌더링 중단
setTimeout(() => {
  abort();
}, 10000);

```

사용법

- React 트리를 HTML로 Node.js 스트림에 렌더링하기
- 콘텐츠가 로드되는 동안 더 많은 콘텐츠 `스트리밍`하기 → Suspense 객체로 감싸기

  전체 Shell이 렌더링되면 onShellReady 콜백이 실행된다. → 이 때 스트리밍이 시작된다.

### renderToReadableStream

Readable Web Stream을 이용해 React 트리를 그린다.

클라이언트에서 hydrateRoot를 호출해 서버에서 생성된 HTML을 상호작용 가능하도록 만든다.

참고 : 해당 API는 Web Stream에 의존한다. Node.js의 경우 renderToPipeableStream을 대신 사용하기

사용법

- Readable Web Stream을 이용해 React 트리를 HTML처럼 렌더링하기
- 더 많은 컨텐츠를 스트리밍하면서 로드하기

```javascript
import { renderToReadableStream } from 'react-dom/server';

async function handler(request) {
  const stream = await renderToReadableStream(<App />, {
    bootstrapScripts: ['/main.js']
  });
  return new Response(stream, {
    headers: { 'content-type': 'text/html' },
  });
}
```

root 컴포넌트 : `<App />`와 함께, bootstrap `<script>` : `[’/main.js’]`경로 리스트를 제공해야 한다.

서버 렌더링 멈추기

```javascript
async function handler(request) {
  try {
    // 서버에게 강제로 렌더링 포기
    const controller = new AbortController();
    setTimeout(() => { 
      controller.abort();
    }, 10000);

    const stream = await renderToReadableStream(<App />, {
      signal: controller.signal,
      bootstrapScripts: ['/main.js'],
      onError(error) {
        didError = true;
        console.error(error);
        logServerCrashReport(error);
      }
    });
    // ...
  }
}
```

### renderToStaticMarkup

상호작용하지 않는 React 트리를 HTML 문자열로 렌더링한다.

```javascript
import { renderToStaticMarkup } from 'react-dom/server';

const html = renderToStaticMarkup(<Page />);
```

사용법

- 상호작용하지 않는 React트리를 HTML문자열로 렌더링하기

주의사항

- renderToStaticMarkup의 출력값은 Hydrate될 수 없다.
- 제한된 Suspense 지원
- 브라우저에서 동작하지만 클라이언트 코드에서 사용하는 것은 권장하지 않음

  → 브라우저에서 HTML로 렌더링해야 하는 경우 createRoot을 사용하고, DOM에서 HTML을 읽기ㄴ

### renderToString

React 트리를 HTML 문자열로 렌더링한다.

주의 : 스트리밍X, 데이터 대기 지원 X : 문자열을 즉시 반환하기 때문에 지원 X

→ Node.js : rendertoPipeableStream, Web Stream : renderToReadableStream

```javascript
import { renderToString } from 'react-dom/server';

const html = renderToString(<App />);
```

사용법

- React 트리를 HTML문자열로 렌더링하기

참고

- 일부 클라이언트 컴포넌트에서의 renderToString 사용
  - 클라이언트에서 react-dom/server를 가져오면 불필요하게 번들 크기가 커져서 피해야 한다.
  - 일부 컴포넌트를 HTML로 렌더링할 경우 createRoot를 사용하고 DOM에서 HTML 읽기

## Static API

### prerender

`Web Stream`을 사용하여 React 트리를 `정적` HTML문자열로 렌더링

→ Node.js에서는 prerenderToNodeStream을 대신 사용하기

SSG에 사용된다.

사용법

- React 트리를 정적 HTML 스트림으로 렌더링하기
- React 트리를 정적 HTML 문자열로 렌더링하기
- 모든 데이터 로드
- 사전 렌더링 중단

```javascript
import { prerender } from 'react-dom/static';

async function handler(request) {
  const {prelude} = await prerender(<App />, {
    bootstrapScripts: ['/main.js']
  });
  return new Response(prelude, {
    headers: { 'content-type': 'text/html' },
  });
}
```

루트 컴포넌트 : `<App />`과 함께 부트스트랩 경로 목록 : ['/main.js']을 제공해야 한다.

### prerenderToNodeStream

Node.js Stream을 사용해 React 트리를 정적 HTML 문자열로 렌더링한다.

```javascript
import { prerenderToNodeStream } from 'react-dom/static';

// 라우트 핸들러 문법은 사용하는 백엔드 프레임워크에 따라 다릅니다.
app.use('/', async (request, response) => {
  const { prelude } = await prerenderToNodeStream(<App />, {
    bootstrapScripts: ['/main.js'],
  });

  response.setHeader('Content-Type', 'text/plain');
  prelude.pipe(response);
});
```

## React의 규칙

### 컴포넌트와 Hook은 순수해야 한다.

순수 함수는 오직 계산만 수행하고 그 외의 작업은 하지 않는다. 이는 코드의 이해와 디버깅을 쉽게 만들어주며, React가 컴포넌트와 Hook을 자동으로 최적화할 수 있게 해준다.

### 1. 순수성이 중요한 이유

순수한 컴포넌트나 Hook은 아래와 같다.

- 멱등성 : 컴포넌트의 Props, State, Context, Hook의 인수에 대한 동일한 입력으로 항상 같은 결과를 얻는다.
- 렌더링 시 사이드 이펙트가 없어야 한다. :  사이드 이펙트가 있는 코드는 렌더링과 별도로 실행해야 한다.
  - 예: 이벤트 핸들러를 통한 UI 업데이트, 렌더링 후에 실행되는 Effect
- 지역 변수가 아닌 값 변경하지 않기 : 컴포넌트와 Hook은 렌더링 시 지역에서 생성되지 않은 값을 수정하지 않아야 한다.

### 1.1 React가 코드를 실행하는 방식

React는 선언적이다. 무엇을 렌더링할 지 말해주면 어떻게 최적으로 표시할지 알아낸다. 사용자는 React를 잘 사용하기 위해 모든 단계를 알 필요는 없다. 하지만 고수준에서 렌더링 중에 실행되는 코드와 그. ㅚ부에서 실행되는 코드에 대해 알아야 한다.

### 2. 멱등성 - 컴포넌트와 Hook은 멱등해야 한다.

컴포넌트는 항상 Props, State, Context와 같은 입력에 대해 동일한 출력을 반환해야 한다. (멱등성은 함수형 프로그래밍에서 널리 사용되는 용어이다.)

예) 멱등하지 않은 컴포넌트

```javascript
function Clock() {
  const time = new Date(); // 🔴 Bad: 항상 다른 결과를 반환
  return <span>{time.toLocaleString()}</span>
}
```

예) 멱등성을 지킨 컴포넌트

```javascript
import { useState, useEffect } from 'react';

function useTime() {
  // 1. 현재 날짜의 State를 추적합니다. `useState`는 초기 State로 초기화 함수를 받습니다.
  //    이 함수는 Hook이 호출될 때 한 번만 실행되므로, Hook이 호출되는 시점의 현재 날짜가
  //    처음으로 설정됩니다.
  const [time, setTime] = useState(() => new Date());

  useEffect(() => {
    // 2. `setInterval`을 사용하여 현재 날짜를 매초마다 업데이트합니다.
    const id = setInterval(() => {
      setTime(new Date()); // ✅ Good: 렌더링에서 비멱등 코드가 더 이상 실행되지 않음.
    }, 1000);
    // 3. `setInterval` 타이머가 누수되지 않도록 Cleanup 함수를 반환합니다.
    return () => clearInterval(id);
  }, []);

  return time;
}

export default function Clock() {
  const time = useTime();
  return <span>{time.toLocaleString()}</span>;
}

```

멱등하지 않은 new Date()호출을 Effect로 감싸면 그 계산을 렌더링 외부로 이동시킨다.

### 3. 사이드 이펙트는 렌더링 외부에서 실행되어야 한다.

사이드 이펙트는 렌더링 중에 실행되어서는 안된다. → React가 여러번 렌더링 할 수 있다.

참고 : 사이드 이펙트는 이벤트 핸들러나 Effect안에서 작성한다. 렌더링 중에는 작성해서는 안된다.

### 3.1 변경이 허용되는 경우

지역 변경

자바스크립트에서의 변경(Mutation)은 원시 값이 아닌 값을 변경하는 것을 의미한다. React에서는 변경을 권하지 않지만, 지역 변경은 문제가 되지 않는다.

예)

```javascript
// const items = []; // 🔴 Bad: 컴포넌트 외부에서 생성됨.
function FriendList({ friends }) {
  const items = []; // ✅ Good: 지역 변수로 생성됨.

  for (let i = 0; i < friends.length; i++) {
    const friend = friends[i];
    items.push(
      <Friend key={friend.id} friend={friend} />
    ); // 🔴 Bad: 렌더링 외부에서 생성된 값을 변경합니다.
  }
  return <section>{items}</section>;
}
```

### 3.2 Lazy 초기화

Lazy 초기화는 완전히 순수하지 않더라고 괜찮다.

```javascript
function ExpenseForm() {
  SuperCalculator.initializeIfNotReady(); // ✅ Good: 다른 컴포넌트에 영향을 미치지 않는 경우
  // 계속 렌더링...
}
```

### 3.3 DOM 변경

사용자에게 직접적으로 보이는 사이드 이펙트는 React 컴포넌트의 렌더링 로직에서 허용되지 않는다. → 컴포넌트 함수를 호출하는 것만으로 화면에 변화를 일으켜서는 안된다.

```javascript
function ProductDetailPage({ product }) {
  document.title = product.title; // 🔴 Bad: DOM을 변경함
}
```

### 4. Props과 State는 불변이다.

컴포넌트의 Props와 State는 불변의 스냅샷이다. → 직접적으로 변경하면 안된다.

변경이 필요하면 새로운 Props를 전달하거나 useState에서 제공하는 Setter함수 사용하기

### 4.1 Props 변경하지 않기

```javascript
function Post({ item }) {
  // item.url = new Url(item.url, base); // 🔴 Bad: 절대 Props를 직접 변경하지 마세요.
  const url = new Url(item.url, base); // ✅ Good: 대신, 복사본을 만드세요.

  return <Link url={item.url}>{item.title}</Link>;
}
```

### 4.2 State 변경하지 않기

```javascript
function Counter() {
  const [count, setCount] = useState(0);

  function handleClick() {
    // count = count + 1; // 🔴 Bad: 절대 State를 직접 변경하지 마세요.
    setCount(count + 1); // ✅ Good: useState에서 반환된 setter 함수를 사용하세요
  }

  return (
    <button onClick={handleClick}>
      You pressed me {count} times
    </button>
  );
}
```

### 5. Hook의 반환값과 인수는 불변이다.

값이 Hook에 전달되면 이를 수정해서는 안된다.

```javascript
function useIconStyle(icon) {
  const theme = useContext(ThemeContext);
  const newIcon = { ...icon }; // ✅ Good: 대신 복사본을 만드세요.
  if (icon.enabled) {
    // icon.className = computeStyle(icon, theme); // 🔴 Bad: 절대 Hook 인수를 직접 변경하지 마세요.
    newIcon.className = computeStyle(icon, theme);
  }
  return newIcon;
}
```

### 6. JSX로 전달된 값은 불변이다.

JSX에 사용된 후에는 값을 변경하지 말아야 한다. → JSX가 생성되기 전에 변경

JSX에 전달된 후에 값을 변경하면 React가 업데이트의 여부를 알 지 못하기 때문에 오래된 UI로 이어질 수 있다.

### React가 컴포넌트와 Hook을 호출하는 방식

### 1. 컴포넌트 함수를 직접 호출하지 않기

컴포넌트는 JSX에서만 사용해야 한다. (일반 함수처럼 호출하지 않아야 한다.)

이유 : React는 렌더링하는 동안 컴포넌트 함수가 언제 호출될지 결정하는데 이를 JSX로 수행한다.

예)

```javascript
function BlogPost() {
  // return <Layout>{Article()}</Layout>; // 🔴 Bad: 컴포넌트 함수를 직접 호출하지 마세요.
  return <Layout><Article /></Layout>; // ✅ Good: 컴포넌트를 JSX에서만 사용합니다.
}
```

- 함수대신 JSX로 수행하는 이점
  - 컴포넌트가 함수 이상의 역할을 함
  - 컴포넌트 타입이 재조정 과정에 참여한다. : `<Feed />` → `<Profile />`로 컴포넌트 변경 시 React는 이를 재사용하지 않음
  - 재조정 과정이 더 효율적이다. : 어떤 컴포넌트가 다시 렌더링이 필요한지 결정,

### 2. Hook을 일반 값처럼 전달하지 않기

Hook은 컴포넌트나 Hook 내부에서 호출되어야 한다.

### 2.1 Hook을 동적으로 변경하지 않기

```javascript
function ChatInput() {
  const data = useDataWithLogging(); // ✅ Good: Hook을 새로 작성하세요.
  // const useDataWithLogging = withLogging(useData); // 🔴 Bad: 고차 Hook을 작성하지 마세요.

}

function useDataWithLogging() {
  // ... 여기에 새로운 Hook의 로직을 작성하세요
}
```

### 2.2 Hook을 동적으로 사용하지 않기

예) X

```javascript
function ChatInput() {
  return <Button useData={useDataWithLogging} /> // 🔴 Bad: Hook을 Props로 전달하지 마세요.
}
```

예)

```javascript
function ChatInput() {
  return <Button />
}

function Button() {
  const data = useDataWithLogging(); // ✅ Good: Hook을 직접 사용하세요.
}

function useDataWithLogging() {
  // Hook의 동작을 변경하는 조건부 로직이 있다면, 이는 Hook 내부에 포함해야 합니다.
}
```

### Hook의 규칙

### 1. Hook을 최상위 레벨에서만 호출하기

useXXX 함수를 Hook이라고 부른다.

Hook을 항상 React 함수의 최상위 레벨에서 호출하고, early return 이전에 사용해야 한다.

```javascript
function Counter() {
  // ✅ Good: 함수 컴포넌트의 최상위 레벨에서 사용합니다.
  const [count, setCount] = useState(0);
  // ...
}

function useWindowWidth() {
  // ✅ Good: 커스텀 Hook의 최상위 레벨에서 사용합니다.
  const [width, setWidth] = useState(window.innerWidth);
  // ...
}
```

아래에서 호출하는 것은 지원되지 않는다.

- 조건문/반복문
- 조건부 return문 이후
- 이벤트 핸들러
- 클래스 컴포넌트
- useMemo, useReducer, useEffect 내부
- try/catch/finally

## React 서버 컴포넌트

### 서버 컴포넌트

서버 컴포넌트는 번들링 전에 클라이언트 앱이나 SSR 서버와는 분리된 환경에서 미리 렌더링되는 새로운 유형의 컴포넌트이다.

서버 컴포넌트는 빌드 시간에 CI 서버에서 한 번 실행되거나, 각 요청마다 웹 서버를 통해 실행할 수 있다.

### 1. 서버 없이 서버 컴포넌트 사용하기

예) 클라이언트에서 Effect를 사용한 정적 데이터 가져오기

```javascript
// bundle.js
import marked from 'marked'; // 35.9K (11.2K gzipped)
import sanitizeHtml from 'sanitize-html'; // 206K (63.3K gzipped)

function Page({page}) {
  const [content, setContent] = useState('');
  // NOTE: loads *after* first page render.
  useEffect(() => {
    fetch(`/api/content/${page}`).then((data) => {
      setContent(data.content);
    });
  }, [page]);

  return <div>{sanitizeHtml(marked(content))}</div>;
}
```

해당 패턴은 정적 콘텐츠를 렌더링하기 위해 페이지가 로드된 후 추가 라이브러리를 다운받고 파싱해야 한다. 또 데이터를 가져오기 위한 두 번째 요청을 기다려야 한다.

서버 컴포넌트를 사용하면 빌드 시간에 한 번만 렌더링이 가능하다.

예) 서버 컴포넌트를 통한 정적 데이터 가져오기

```javascript
import marked from 'marked'; // Not included in bundle
import sanitizeHtml from 'sanitize-html'; // Not included in bundle

async function Page({page}) {
  // NOTE: loads *during* render, when the app is built.
  const content = await file.readFile(`${page}.md`);

  return <div>{sanitizeHtml(marked(content))}</div>;
}
```

### 2. 서버와 함께 서버 컴포넌트 사용하기

예) 클라이언트에서 Effect를 통한 동적 데이터 가져오기

```javascript
// bundle.js
function Note({id}) {
  const [note, setNote] = useState('');
  // NOTE: loads *after* first render.
  useEffect(() => {
    fetch(`/api/notes/${id}`).then(data => {
      setNote(data.note);
    });
  }, [id]);

  return (
    <div>
      <Author id={note.authorId} />
      <p>{note}</p>
    </div>
  );
}

function Author({id}) {
  const [author, setAuthor] = useState('');
  // NOTE: loads *after* Note renders.
  // Causing an expensive client-server waterfall.
  useEffect(() => {
    fetch(`/api/authors/${id}`).then(data => {
      setAuthor(data.author);
    });
  }, [id]);

  return <span>By: {author.name}</span>;
}
```

예) 서버 컴포넌트를 통한 동적 데이터 가져오기

```javascript
import db from './database';

async function Note({id}) {
  // NOTE: loads *during* render.
  const note = await db.notes.get(id);
  return (
    <div>
      <Author id={note.authorId} />
      <p>{note}</p>
    </div>
  );
}

async function Author({id}) {
  // NOTE: loads *after* Note,
  // but is fast if data is co-located.
  const author = await db.authors.get(id);
  return <span>By: {author.name}</span>;
}
```

### 3. 서버 컴포넌트에 상호작용 추가하기

서버 컴포넌트는 브라우저로 전송되지 않기에 useState와 같은 API를 사용하는게 불가능하다.

“use client”지시어를 활용해서 클라이언트 컴포넌트와 함께 구성할 수 있다.

예)

```javascript
// Server Component
import Expandable from './Expandable';

async function Notes() {
  const notes = await db.notes.getAll();
  return (
    <div>
      {notes.map(note => (
        <Expandable key={note.id}>
          <p note={note} />
        </Expandable>
      ))}
    </div>
  )
}
```

```javascript
// Client Component
"use client"

export default function Expandable({children}) {
  const [expanded, setExpanded] = useState(false);
  return (
    <div>
      <button
        onClick={() => setExpanded(!expanded)}
      >
        Toggle
      </button>
      {expanded && children}
    </div>
  )
}
```

### 4. 서버 컴포넌트와 함께 비동기 컴포넌트 사용하기

비동기 컴포넌트에서 await를 사용할 때 React는 렌더링을 지연하고, Promise가 해결할 때까지 기다린 후 렌더링을 다시 시작한다. 서버-클라이언트의 경계를 넘어 동작하고 Suspense에 대한 스트리밍을 지원한다.

서버-클라이언트의 경계를 넘어 : 서버 컴포넌트 안에 클라이언트 컴포넌트가 존재하는 경우

```javascript
// Server Component
import db from './database';

async function Page({id}) {
  // Will suspend the Server Component.
  const note = await db.notes.get(id);

  // NOTE: not awaited, will start here and await on the client.
  const commentsPromise = db.comments.get(note.id);
  return (
    <div>
      {note}
      <Suspense fallback={<p>Loading Comments...</p>}>
        <Comments commentsPromise={commentsPromise} />
      </Suspense>
    </div>
  );
}

// Client Component
"use client";
import {use} from 'react';

function Comments({commentsPromise}) {
  // NOTE: this will resume the promise from the server.
  // It will suspend until the data is available.
  const comments = use(commentsPromise);
  return comments.map(commment => <p>{comment}</p>);
}
```

### 서버 함수

서버 함수를 사용하면 클라이언트 컴포넌트가 서버에서 실행되는 비동기 함수를 호출할 수 있다.

```javascript
// 서버 컴포넌트
import Button from './Button';

function EmptyNote () {
  async function createNote() {
    // 서버 함수
    'use server';

    await db.notes.create();
  }

  return <Button onClick={createNote}/>;
}

"use client";

export default function Button({onClick}) {
  console.log(onClick);
  // {$$typeof: Symbol.for("react.server.reference"), $$id: 'createNoteAction'}
  return <button onClick={() => onClick()}>Create Empty Note</button>
}
```

### useActionState를 이용한 서버 함수

```javascript
"use client";

import {updateName} from './actions';

function UpdateName() {
  const [state, submitAction, isPending] = useActionState(updateName, {error: null});

  return (
    <form action={submitAction}>
      <input type="text" name="name" disabled={isPending}/>
      {state.error && <span>Failed: {state.error}</span>}
    </form>
  );
}
```
