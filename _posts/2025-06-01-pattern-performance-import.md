---
title: "Performance Patterns 2 : import"
date: 2025-06-01 00:10:00 +0900
categories: [design-pattern, performance]
tags: [design-pattern, dynamic-import]
---

## 1. Static Import

`import` 키워드는 다른 모듈이 export한 코드들을 import할 수 있도록 해준다. 기본적으로 정적으로 가져온 모듈들은 초기 번들에 포함된다.

예) ES2015의 import문

```javascript
import module from 'module'
```

예) 정적 import를 포함하는 예제

```javascript
import React from "react";
import "./styles.css";

// Static한 import : Chatlist, ChatInput and UserInfo
import UserInfo from "./components/UserInfo";
import ChatList from "./components/ChatList";
import ChatInput from "./components/ChatInput";

const App = () => {
  return (
    <div className="App">
      <UserInfo />
      <ChatList />
      <ChatInput />
    </div>
  );
};

export default App;


// ChatInput
import React from "react";
import Send from "./icons/Send";
import Emoji from "./icons/Emoji";

import Picker from "./EmojiPicker";

const ChatInput = () => {
  const [pickerOpen, togglePicker] = React.useReducer(state => !state, false);

  return (
    <div className="chat-input-container">
      <input type="text" placeholder="Type a message..." />
      <Emoji onClick={togglePicker} />
      {pickerOpen && <Picker />}
      <Send />
    </div>
  );
};

console.log("ChatInput loaded", Date.now());

export default ChatInput;

```

자바스크립트 엔진은 정적으로 포함시킨 `EmojiPicker` 모듈을 즉시 실행하기 시작한다.

어떤 컴포넌트가 정적으로 포함될 경우 웹팩은 이 코드를 초기 번들에 포함시킨다.

예) 번들 시 메세지

```
Asset           Size      Chunks            Chunk Names
main.bundle.js  1.5 MiB    main  [emitted]   main
```

해당 소스코드는 `main.bundle.js`라는 하나의 번들에 포함된다. 번들 크기가 커지면 앱이 로딩되기까지 시간에 악영향을 주며, 기기나 네트워크 환경에 따라 더 영향을 줄 수 있다. App 컴포넌트가 화면에 그려지기 전까지 모든 모듈들이 로드되어 파싱이 끝나야 한다.

- 로딩 타임을 줄이는 방법

  App 컴포넌트가 렌더되고 인터렉션이 일어난 이후 동적으로 모듈을 import

## 2. Dynamic Import (Suspense)

**가정**

**UserInfo**, **ChatList**, **ChatInput**, **EmojiPicker** 컴포넌트가 존재한다. 그러나 페이지가 처음 로드될 때에는 이 중에 EmojiPicker를 제외한 세가지 컴포넌트만 있어도 된다. 따라서 EmojiPicker를 초기 번들에 포함하여 불필요하게 로딩 타임을 증가시킬 필요가 없다.

**해결**

EmojiPicker를 Suspense로 동적으로 포함하면 된다. 정적으로 포함시키는 대신 해당 요소가 보여져야 할 때(이모지 버튼을 눌러 화면에 보여져야 할 때) 다운로드 되도록 하는 것이다.

React.Suspense는 동적으로 로드외어야 하는 컴포넌트를 받고, App 컴포넌트가 렌더링할 때 EmojiPicker를 가져오는 것을 일시적으로 중단하도록 하여 빠르게 화면을 그릴 수 있다. 사용자가 이모지 버튼을 누를 때, 즉 화면에 보여져야 할 때 EmojiPicker 컴포넌트가 렌더링된다.

EmojiPicker 컴포넌트는 원래 렌더링되어야 하는 모듈을 prop으로 받은 `Suspense` 컴포넌트를 렌더링한다. Suspense 컴포넌트는 동적으로 렌더링 될 컴포넌트가 로딩중일 때 렌더링될 컴포넌트(fallback)를 받는다.

**결과**

불필요하게 초기 번들에 포함되는 것을 막고, 별도의 번들에 포함시켜 초기에 로딩해야 하는 스크립트의 사이즈를 줄일 수 있게 된다.

초기에 로딩되는 번들 사이즈가 더 작으면 앱이 더 빨리 로딩된다. 사용자가 빈 화면에서 기다리는 시간이 줄어든다. fallback 컴포넌트는 사용자에게 앱이 멈춘게 아니라 추가적인 모듈이 처리되고 실행되기까지 조금만 더 기다리면 된다는 것을 인지시킨다.

## 3. Import On visibility

사용자의 상호작용 외에 초기 렌더링 된 페이지에서도 보이지 않는 컴포넌트들이 있을 수 있다. 예를 들어 뷰포트 바깥에 있는 이미지를 처음부터 로드하지 않고, 사용자가 스크롤하여 보게 되었을 때 로드하는 `이미지 지연 로딩`이 있다.

모든 이미지를 페이지 로딩과 함께 즉시 로드하지 않기 때문에 초기 로딩 속도를 줄일 수 있다. 이 패턴을 컴포넌트에도 적용할 수 있다.

모든 이미지를 페이지의 로딩과 함께 즉시 로드하지 않기 때문에 페이지의 초기 로딩 속도를 줄일 수 있다. 이 패턴을 컴포넌트에도 적용할 수 있다. 컴포넌트가 뷰포트 안에 있는지 여부를 알아내고 필요 모듈을 동적으로 로드하기 위해 **`IntersectionObserver`** API 사용하거나 react-lazyload 혹은 react-loadable-visibility같은 라이브러리를 사용할 수 있다.

## 4. Import on Interaction

중요하지 않은 리소스는 사용자 상호작용에 따라 필요해질 때 로드하도록 한다.

바로 필요하지 않은 파싱 및 실행 비용이 높은 리소스들을 즉시 불러올 경우 메인 스레드를 block할 수 있으며, 사용자에게 중요한 영역이 상호작용이 가능해지기까지 시간을 지연시킬 수 있다. 이는 FID(First Input Delay), TBT(Total Block Time), TTI(Time To Interactive)에 영향을 줄 수 있다.

이런 리소스들을 즉시 로드하는 대신, 아래의 상황에 로드하도록 할 수 있다.

- 사용자가 컴포넌트에 처음 클릭했을 때
- 스크롤하여 컴포넌트가 뷰포트에 들어왔을 때
- requestIdleCallback을 통해 브라우저 프로세스가 유휴 상태일 때

리소스를 불러오는 방법들을 고수준에서 정리하면

- Eager : 리소스를 즉시 불러옴
- Lazy(Route based) : 특정 경로를 탐색할 때 로드
- Lazy(On interaction) : 사용자가 UI를 클릭했을 때 호출
- Lazy(In viewport) : 사용자가 스크롤을 움직여 컴포넌트가 뷰포트에 들어올 때
- `Prefetch` : 우선순위를 높여 로드, 단 중요한 리소스보다는 늦게 로드
- `Preload` : 중요 리소스로 지정하여 즉시 받기

참고 : 퍼스트 파티 코드를 상호작용 발생 시 동적으로 로드하게 하는 것은 상호작용 전에 미리 불러올 수 없는 경우에만 적용해야 한다.

### 상호작용 시 불러오기 구현

- Vanilla JS

  [**dynamic import()**](https://v8.dev/features/dynamic-import)를 사용하면 모듈을 지연로딩할 수 있으며 프로미스를 반환한다.

  ```javascript
  const btn = document.querySelector('button')

  btn.addEventListener('click', e => {
    e.preventDefault()
    import('lodash.sortby')
      .then(module => module.default)
      .then(sortInput()) // use the imported dependency
      .catch(err => {
        console.log(err)
      })
  })
  ```

- React

  ```
  import React, { lazy, Suspense } from 'react';import MessageList from './MessageList';import MessageInput from './MessageInput';const EmojiPicker = lazy(  () => import('./EmojiPicker'));const Channel = () => {  ...  return (    <div>      <MessageList />      <MessageInput />      {emojiPickerOpen && (        <Suspense fallback={<div>Loading...</div>}>          <EmojiPicker />        </Suspense>      )}    </div>  );};
  ```

  ```javascript
  import React, { lazy, Suspense } from 'react';
  import MessageList from './MessageList';
  import MessageInput from './MessageInput';

  const EmojiPicker = lazy(
    () => import('./EmojiPicker')
  );

  const Channel = () => {
    ...
    return (
      <div>
        <MessageList />
        <MessageInput />
        {emojiPickerOpen && (
          <Suspense fallback={<div>Loading...</div>}>
            <EmojiPicker />
          </Suspense>
        )}
      </div>
    );
  };
  ```

  코드 스플리팅을 통해 전체 로드를 분리시키는 과정은 간단하다. React.lazy는 React 앱 내의 컴포넌트 레벨에서 dynamic import를 적용해 코드를 나눌 수 있게 한다. `Suspense` 컴포넌트와 연결해주면 끝난다.

- Next.js

  import dynamic from 'next/dynamic';

  const EmojiPicker = dynamic(() => import('./EmojiPicker'));

  ```javascript
  import dynamic from 'next/dynamic';

  const EmojiPicker = dynamic(() => import('./EmojiPicker'));
  ```

  Suspense를 사용할 필요 없이 dynamic이 자체적으로 fallback 렌더링을 지원한다. React의 Suspense를 사용하지 않아도 된다.

  Suspense를 사용할 경우

  ```javascript
  const EmojiPicker = dynamic(() => import('./EmojiPicker'), 
  	{ ssr: false, loading: () => <div>Loading...</div> });
  ```

```
import React, { lazy, Suspense } from 'react';import MessageList from './MessageList';import MessageInput from './MessageInput';const EmojiPicker = lazy(  () => import('./EmojiPicker'));const Channel = () => {  ...  return (    <div>      <MessageList />      <MessageInput />      {emojiPickerOpen && (        <Suspense fallback={<div>Loading...</div>}>          <EmojiPicker />        </Suspense>      )}    </div>  );};
```

### **트레이드오프**

비용이 높은 코드들을 사용자의 **인터렉션 발생 시** 다운로드하도록 리펙토링하면 페이지의 초기 로딩 속도를 최적화할 수 있지만, 단점이 없는 것은 아니다.

사용자가 클릭한 뒤 스크립트를 다운로드 받는 시간이 오래 걸리면?  → 코드를 잘게 나누는 것은 코드와 데이터를 다운로드 받고 실행하는 시간을 줄여줄 수 있다. 하지만 의존하는 모듈이 많고 네트워크 속도가 느린 환경을 고려해보아야 한다.

이런 상황을 피하기 위해서는 중요한 리소스들을 받은 직후 이런 리소스를 `prefetch`하는것을 고려할 수 있다. 이를 측정하여 앱이 실제로 어느정도의 성능을 보이는지 확인해보는것이 좋다.
