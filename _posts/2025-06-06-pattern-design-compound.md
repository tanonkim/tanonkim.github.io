---
title: "Design Patterns 1 : Compound Pattern"
date: 2025-06-06 18:00:00 +0900
categories: [design-pattern, design]
tags: [design-pattern, compound-component]
---

{% raw %}
애플리케이션을 개발하다 보면 종종 서로를 참조하는 컴포넌트를 만들기도 한다. 컴포넌트들은 서로 상태를 공유하기도 하고, 특정 로직과 사용되기도 한다. 컴파운트 컴포넌트 패턴은 여러 컴포넌트들이 모여 하나의 동작을 할 수 있게 한다.

## Context API

아래 예제에서는 사진 목록을 보여주고 있다. 버튼을 추가하여 각각의 사진을 수정하거나 삭제하려고 하고, Flyout 컴포넌트를 구현하여 사용자가 메뉴를 누르면 토글할 수 있도록 한다.

예)

```javascript
// index.js
import React from "react";
import ReactDOM from "react-dom";

import ImagesList from "./Images";

const rootElement = document.getElementById("root");
ReactDOM.render(
  <React.StrictMode>
    <div className="App">
      <ImagesList />
    </div>
  </React.StrictMode>,
  rootElement
);

// Images.js
import React from "react";
import FlyOutMenu from "./FlyOutMenu";

const sources = [
  "https://images.pexels.com/photos/939478/pexels-photo-939478.jpeg?auto=compress&cs=tinysrgb&dpr=2&h=750&w=1260",
  "https://images.pexels.com/photos/1692984/pexels-photo-1692984.jpeg?auto=compress&cs=tinysrgb&dpr=2&h=750&w=1260",
  "https://images.pexels.com/photos/162829/squirrel-sciurus-vulgaris-major-mammal-mindfulness-162829.jpeg?auto=compress&cs=tinysrgb&dpr=2&h=750&w=1260"
];

function Image({ source }) {
  return (
    <div className="image-item">
      <img src={source} alt="Squirrel" />
      <FlyOutMenu />
    </div>
  );
}

export default function ImageList() {
  return sources.map((source, i) => <Image source={source} key={i} />);
}
```

```javascript
// FlyOutMenu.js
import React from "react";
import "./styles.css";
import { FlyOut } from "./FlyOut";

export default function FlyoutMenu() {
  return (
    <FlyOut>
      <FlyOut.Toggle />
      <FlyOut.List>
        <FlyOut.Item>Edit</FlyOut.Item>
        <FlyOut.Item>Delete</FlyOut.Item>
      </FlyOut.List>
    </FlyOut>
  );
}
```

```javascript
// FlyOut.js
import React from "react";
import Icon from "./Icon";

const FlyOutContext = React.createContext();

export function FlyOut(props) {
  const [open, toggle] = React.useState(false);

  return (
    <div className={`flyout`}>
      <FlyOutContext.Provider value={{ open, toggle }}>
        {props.children}
      </FlyOutContext.Provider>
    </div>
  );
}

function Toggle() {
  const { open, toggle } = React.useContext(FlyOutContext);

  return (
    <div className="flyout-btn" onClick={() => toggle(!open)}>
      <Icon />
    </div>
  );
}

function List({ children }) {
  const { open } = React.useContext(FlyOutContext);
  return open && <ul className="flyout-list">{children}</ul>;
}

function Item({ children }) {
  return <li className="flyout-item">{children}</li>;
}

FlyOut.Toggle = Toggle;
FlyOut.List = List;
FlyOut.Item = Item;
```

Flyout 컴포넌트 내에서는 세가지 구현이 필요하다.

1. 토글 버튼과 메뉴 리스트를 포함한 Flyout 래퍼
2. 메뉴를 토글할 수 있는 Toggle 버튼
3. 메뉴를 포함한 List 컴포넌트

### Flyout 컴포넌트 구현

이 컴포넌트는 상태를 포함하고 또 자식 컴포넌트들이 받게 될 값을 FlyOutProvider 컴포넌트를 반환하고 있다.

```javascript
// FlyOut.js
import React from "react";
import Icon from "./Icon";

const FlyOutContext = React.createContext();

export function FlyOut(props) {
  const [open, toggle] = React.useState(false);

  return (
    <div className={`flyout`}>
      <FlyOutContext.Provider value={{ open, toggle }}>
        {props.children}
      </FlyOutContext.Provider>
    </div>
  );
}
```

메뉴가 열렸는지 여부를 나타내는 open, 토글이 가능한 toggle 함수를 상태로 포함한 FlyOut 컴포넌트이다.

Toggle 컴포넌트는 사용자가 토글 버튼을 눌렀을 때 나타날 메뉴를 렌더링하고 있다.

```javascript
function Toggle() {
  const { open, toggle } = React.useContext(FlyOutContext);

  return (
    <div className="flyout-btn" onClick={() => toggle(!open)}>
      <Icon />
    </div>
  );
}
```

Toggle 컴포넌트가 FlyOutContext 프로바이더에 접근할 수 있도록 해당 컴포넌트는 FlyOut의 자식 컴포넌트로 렌더링해야 한다. 따라서 단순히 자식 컴포넌트로 렌더링하면 되지만, 여기서는 FlyOut의 컴포넌트의 Static Property로 만들고 있다.

```javascript
const FlyOutContext = React.createContext();

export function FlyOut(props) {
  const [open, toggle] = React.useState(false);

  return (
    <div className={`flyout`}>
      <FlyOutContext.Provider value={{ open, toggle }}>
        {props.children}
      </FlyOutContext.Provider>
    </div>
  );
}

function Toggle() {
  const { open, toggle } = React.useContext(FlyOutContext);

  return (
    <div className="flyout-btn" onClick={() => toggle(!open)}>
      <Icon />
    </div>
  );
}

FlyOut.Toggle = Toggle
```

이렇게 하면 FlyOut 컴포넌트를 사용하는 쪽에서 토글 버튼이 필요한 경우라도 그냥 FlyOut 컴포넌트만 import하면 된다.

```javascript
import React from 'react'
import { FlyOut } from './FlyOut'

export default function FlyoutMenu() {
  return (
    <FlyOut>
      <FlyOut.Toggle />
    </FlyOut>
  )
}
```

토글 뿐만 아니라, 메뉴가 화면에 나와야 한다. Toggle과 같이 컨텍스트를 통해 상태를 가져와 처리할 수 있다.

```javascript
function List({ children }) {
  const { open } = React.useContext(FlyOutContext);
  return open && <ul className="flyout-list">{children}</ul>;
}

function Item({ children }) {
  return <li className="flyout-item">{children}</li>;
}
```

List 컴포넌트는 컨텍스트의 open 값에 따라 메뉴를 보여주거나 감추고 있다. FlyOut의 Static Property로 추가하자

```javascript
const FlyOutContext = createContext()

function FlyOut(props) {
  const [open, toggle] = useState(false)

  return (
    <FlyOutContext.Provider value={{ open, toggle }}>
      {props.children}
    </FlyOutContext.Provider>
  )
}


function List({ children }) {
  const { open } = useContext(FlyOutContext)
  return open && <ul>{children}</ul>
}
```

(open, toggle은 Context.Provider의 value때문에 useContext에서 사용이 가능하다)

지금까지 구현한 것 모두 FlyOut 컴포넌트만 가지고 사용할 수 있다. 예제에서는 수정 메뉴와 삭제 메뉴를 제공해야 하므로, FlyOut.Item을 두개 가진 FlyOut.List 컴포넌트를 사용하면 된다.

```javascript
import React from 'react'
import { FlyOut } from './FlyOut'

export default function FlyoutMenu() {
  return (
    <FlyOut>
      <FlyOut.Toggle />
      <FlyOut.List>
        <FlyOut.Item>Edit</FlyOut.Item>
        <FlyOut.Item>Delete</FlyOut.Item>
      </FlyOut.List>
    </FlyOut>
  )
}
```

컴파운트 패턴은 컴포넌트 라이브러리를 만들 때 유용하게 사용할 수 있다.

## React.Children.map

자식 컴포넌트들을 순회처리 하는데에도 컴파운드 패턴을 사용할 수 있다.

React.cloneElement를 사용하여 컴포넌트를 복제하여 open과 toggle 메서드를 넘길 수 있다.

```javascript
export function FlyOut(props) {
  const [open, toggle] = React.useState(false)

  return (
    <div>
      {React.Children.map(props.children, child =>
        React.cloneElement(child, { open, toggle })
      )}
    </div>
  )
}

function Toggle({ open, toggle }) {
  return (
    <div className="flyout-btn" onClick={() => toggle(!open)}>
      <Icon />
    </div>
  );
}

function List({ children, open }) {
  return open && <ul className="flyout-list">{children}</ul>;
}

function Item({ children }) {
  return <li className="flyout-item">{children}</li>;
}

FlyOut.Toggle = Toggle;
FlyOut.List = List;
FlyOut.Item = Item;
```

모든 메뉴 컴포넌트들은 복제되며 open, toggle 메서드를 전달받았다. 위에서 해당 값을 받기 위해 Context API를 사용했던 것에 비교하여 그냥 prop에서 두 값을 사용하면 된다.

### 장점

컴파운드 패턴은 동작 구현에 필요한 상태를 내부적으로 가지고 있는데, 사용하는 쪽에서는 드러나지 않아 걱정 없이 사용할 수 있다.

이런 패턴을 사용하면 자식 컴포넌트들을 일일히 import할 필요 없이 기능을 이용할 수 있다.

```javascript
import { FlyOut } from './FlyOut'

export default function FlyoutMenu() {
  return (
    <FlyOut>
      <FlyOut.Toggle />
      <FlyOut.List>
        <FlyOut.Item>Edit</FlyOut.Item>
        <FlyOut.Item>Delete</FlyOut.Item>
      </FlyOut.List>
    </FlyOut>
  )
}
```

### 단점

내부에서 React.Children.map을 사용하기 때문에 쓰는 쪽에서 자식 컴포넌트를 약속된 형태로 넘겨야 하는 제약이 생긴다.

엘리먼트를 복제하는 경우, 복제 대상 컴포넌트가 기존에 갖고 있는 prop과 이름이 충돌할 수 있는데, React.cloneElement를 사용할 때 넘어간 값으로 해당 prop은 덮어써질 것이다.
{% endraw %}
