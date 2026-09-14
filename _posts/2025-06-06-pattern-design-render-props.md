---
title: "Design Patterns 3 : Render Props Pattern"
date: 2025-06-06 18:45:00 +0900
categories: [design-pattern, design]
tags: [design-pattern, render-props]
---

{% raw %}
여러 컴포넌트가 동일한 데이터나 동일한 로직을 포함해야 할 때 컴포넌트의 로직을 재사용할 수 있게 되면 편해질 수 있다.

컴포넌트를 재사용 가능하게 할 수 있는 또 다른 방법으로 render prop 패턴을 사용하는 방법이 있다.

render prop은 컴포넌트의 prop으로 함수이며, JSX 엘리먼트를 리턴한다. 컴포넌트 자체는 아무것도 렌더링하지 않지만, render prop 함수를 호출한다.

예) Title 컴포넌트

Title 컴포넌트는 prop으로 넘어온 함수를 호출하여 반환하는 것 이외에는 아무 동작을 하지 않는다.

```javascript
<Title render={() => <h1>I am a render prop!</h1>} />
```

Title컴포넌트 내에서는 단순히 prop의 render 함수를 호출하여 반환한다.

```javascript
const Title = props => props.render()
```

예) 사용 예시

```javascript
import React from "react";
import { render } from "react-dom";

import "./styles.css";

const Title = (props) => props.render();

render(
  <div className="App">
    <Title
      render={() => (
        <h1>
          <span role="img" aria-label="emoji">
            ✨
          </span>
          I am a render prop!{" "}
          <span role="img" aria-label="emoji">
            ✨
          </span>
        </h1>
      )}
    />
  </div>,
  document.getElementById("root")
);
```

### Render Prop의 장점

1. render prop패턴의 장점은 prop을 받는 컴포넌트가 재사용성이 좋다.

Title 컴포넌트는 render prop만 바꿔가며 여러번 사용이 가능하다.

```javascript
import React from "react";
import { render } from "react-dom";
import "./styles.css";

const Title = (props) => props.render();

render(
  <div className="App">
    <Title render={() => <h1>✨ First render prop! ✨</h1>} />
    <Title render={() => <h2>🔥 Second render prop! 🔥</h2>} />
    <Title render={() => <h3>🚀 Third render prop! 🚀</h3>} />
  </div>,
  document.getElementById("root")
);
```

예) 최적화

```javascript
import React from "react";
import { render } from "react-dom";
import "./styles.css";

const Title = (props) => (
  <>
    {props.renderFirstComponent()}
    {props.renderSecondComponent()}
    {props.renderThirdComponent()}
  </>
);

render(
  <div className="App">
    <Title
      renderFirstComponent={() => <h1>✨ First render prop! ✨</h1>}
      renderSecondComponent={() => <h2>🔥 Second render prop! 🔥</h2>}
      renderThirdComponent={() => <h3>🚀 Third render prop! 🚀</h3>}
    />
  </div>,
  document.getElementById("root")
);
```

1. 인자 전달

render prop을 받는 컴포넌트는 단순히 함수를 호출해 JSX엘리먼트를 렌더링 하는 것 외에도 많은 동작을 할 수 있다.

함수를 호출하는 것 이외에도, render prop 함수를 호출할 때 인자를 전달할 수 있다.

```javascript
function Component(props) {
  const data = { ... }

  return props.render(data)
}
```

```javascript
<Component render={data => <ChildComponent data={data}>}/>
```

### 예시) Kelvin, Fahrenheit

```javascript
import React, { useState } from "react";
import "./styles.css";

function Input() {
  const [value, setValue] = useState("");

  return (
    <input
      type="text"
      value={value}
      onChange={e => setValue(e.target.value)}
      placeholder="Temp in °C"
    />
  );
}

export default function App() {
  return (
    <div className="App">
      <h1>☃️ Temperature Converter 🌞</h1>
      <Input />
      <Kelvin />
      <Fahrenheit />
    </div>
  );
}

function Kelvin({ value = 0 }) {
  return <div className="temp">{value + 273.15}K</div>;
}

function Fahrenheit({ value = 0 }) {
  return <div className="temp">{(value * 9) / 5 + 32}°F</div>;
}
```

텍스트 박스에 섭씨 온도를 받아서 켈빈과 화씨 온도로 표현해주는 앱이다. Input 컴포넌트는 값 입력을 받기 위해 state를 갖고 있는데, Fahrenheit, Kelvin 컴포넌트는 state를 전달받을 방법이 없다.

- 해결방법1 : 부모 컴포넌트로 상태 올리기

  `Fahrenheit`컴포넌트와 `Kelvin` 컴포넌트가 사용자가 입력한 값을 전달받기 위한 방법 중 하나는 상태를 부모 컴포넌트로 올려보내는 방법이 있다. → input 자체가 상태를 갖는 것 대신 세 컴포넌트의 부모 컴포넌트로 상태를 올려 보내는 것이다.

  ```javascript
  function Input({ value, handleChange }) {
    return <input value={value} onChange={e => handleChange(e.target.value)} />
  }

  export default function App() {
    const [value, setValue] = useState('')

    return (
      <div className="App">
        <h1>☃️ Temperature Converter 🌞</h1>
        <Input value={value} handleChange={setValue} />
        <Kelvin value={value} />
        <Fahrenheit value={value} />
      </div>
    )
  }
  ```

  이 방법도 유효하지만, 규모가 큰 앱이나 컴포넌트가 여러 자식을 갖는 경우 부모까지 상태를 올리는 방법은 까다롭다. 상태의 변경은 모든 자식의 컴포넌트의 리렌더링을 유발할 수 있고, 이런 상황이 쌓이면 앱의 전체적인 성능을 떨어트릴 수 있다.

  ⇒ Render Props 활용

- 해결방법 2 : Render Props 활용하기

  ```javascript
  const Input: React.FC<InputProps> = (props) => {
    const [value, setValue] = useState('')

    return (
      <>
        <input
          type="text"
          value={value}
          onChange={e => setValue(e.target.value)}
          placeholder="Temp in °C"
        />
        {props.render(value)}
      </>
    )
  }

  export default function App() {
    return (
      <div className="App">
        <h1>Temperature Converter</h1>
        <Input
          render={value => (
            <>
              <Kelvin value={value} />
              <Fahrenheit value={value} />
            </>
          )}
        />
      </div>
    )
  }
  ```

### 자식 컴포넌트를 함수로 받기 (props.render() → props.children())

일반적인 JSX 컴포넌트에서 자식 엘리먼트로 React 엘리먼트를 반환하는 함수를 전달해줄 수 있다.

예시에서는 children prop으로 사용가능하며, 이것도 역시 render prop에 해당된다.

수정사항 : Input 컴포넌트에 명시적으로 render prop을 넘기는 대신 자식 컴포넌트를 함수로 넘기도록 수정하기

```javascript
export default function App() {
  return (
    <div className="App">
      <h1>☃️ Temperature Converter 🌞</h1>
      <Input>
        {value => (
          <>
            <Kelvin value={value} />
            <Fahrenheit value={value} />
          </>
        )}
      </Input>
    </div>
  )
}
```

```javascript
function Input(props) {
  const [value, setValue] = useState('')

  return (
    <>
      <input
        type="text"
        value={value}
        onChange={e => setValue(e.target.value)}
        placeholder="Temp in °C"
      />
      {props.children(value)}
    </>
  )
}
```

이렇게 해서 render prop의 이름을 어떻게 지을지 고민하지 말고, Kelvin과 Fahrenheit 둘 다 사용자의 입력값에 접근이 가능하다.

### Hooks with Apollo Client

몇몇 상황에서 render props패턴은 hooks로 대체될 수 있다. 예시로 Apollo Client이다.

Apollo Client를 사용하는 방법 중 하나는 Mutation과 Query 컴포넌트를 사용하는 것이다. 아래 예시는 graphql()의 HOC를 사용하는 대신 Mutation 컴포넌트가 render prop을 받는 것을 알 수 있다.

```javascript
import React from "react";
import "./styles.css";

import { Mutation } from "react-apollo";
import { ADD_MESSAGE } from "./resolvers";

export default class Input extends React.Component {
  constructor() {
    super();
    this.state = { message: "" };
  }

  handleChange = (e) => {
    this.setState({ message: e.target.value });
  };

  render() {
    return (
      <Mutation
        mutation={ADD_MESSAGE}
        variables={{ message: this.state.message }}
        onCompleted={() =>
          console.log(`Added with render prop: ${this.state.message} `)
        }
      >
        {(addMessage) => (
          <div className="input-row">
            <input
              onChange={this.handleChange}
              type="text"
              placeholder="Type something..."
            />
            <button onClick={addMessage}>Add</button>
          </div>
        )}
      </Mutation>
    );
  }
}
```

Mutation 컴포넌트가 자식 엘리먼트에게 데이터를 전달 할 수 있도록 컴포넌트를 렌더하는 함수를 자식 요소로 제공했다.

```javascript
<Mutation mutation={...} variables={...}>
  {addMessage => <div className="input-row">...</div>}
</Mutation>
```

### Render Prop 형태의 단점

render prop 형태는 HOC에 비해 조금 더 선호되긴 하지만 단점이 있다.

1. 트리가 깊어진다.

   ```javascript
   <Mutation mutation={FIRST_MUTATION}>
     {firstMutation => (
       <Mutation mutation={SECOND_MUTATION}>
         {secondMutation => (
           <Mutation mutation={THIRD_MUTATION}>
             {thirdMutation => (
               <Element
                 firstMutation={firstMutation}
                 secondMutation={secondMutation}
                 thirdMutation={thirdMutation}
               />
             )}
           </Mutation>
         )}
       </Mutation>
     )}
   </Mutation>
   ```

   컴포넌트가 여러개의 mutation을 사용해야 하는 경우, Mutaion 컴포넌트나 Query 컴포넌트를 중첩해서 사용해야 한다.

   ⇒ render prop과 함께 사용하는 대신 useQuery 훅을 사용한다.

   ```javascript
   import React, { useState } from "react";
   import "./styles.css";

   import { useMutation } from "@apollo/react-hooks";
   import { ADD_MESSAGE } from "./resolvers";

   export default function Input() {
     const [message, setMessage] = useState("");
     const [addMessage] = useMutation(ADD_MESSAGE, {
       variables: { message }
     });

     return (
       <div className="input-row">
         <input
           onChange={(e) => setMessage(e.target.value)}
           type="text"
           placeholder="Type something..."
         />
         <button onClick={addMessage}>Add</button>
       </div>
     );
   }
   ```

### 장점

1. render prop을 사용하여 몇몇 컴포넌트 간 데이터를 공유하는 것은 간단하다. children prop을 활용하는 것으로 해당 컴포넌트를 재사용할 수 있게 된다. render prop은 HOC를 사용할 때 마주칠 수 있는 이슈를 해결할 수 있다.

1. props를 자동으로 머지하도록 구현하지 않기 때문에 HOC 패턴을 사용할 때 prop이 어디서 만들어져오는지 구별하기 힘든 이슈가 없다. → 부모로 부터 받은 prop을 명시적으로 받아 처리하기 때문이다.

1. 함수의 인자에서 명시적으로 prop이 전달되기 때문에 HOC를 사용할 때 prop이 모호한 문제가 해결된다. 이 때문에 prop이 어디서 부터 오는지 확실히 알 수 있다.

1. render props를 활용하여 렌더링 컴포넌트와 앱의 로직을 분리할 수 있다. 상태를 가진 컴포넌트는 render prop을 받고, 상태가 없는 컴포넌트를 렌더할 수 있다.

### 단점

1. render props로 해결하려는 문제는 React Hooks로 대체되었다. Hooks는 컴포넌트에 재사용성과 데이터 공유를 위한 방법 자체를 바꿔놓았다. 대부분의 render props는 Hooks로 대체 가능하다.

1. render props 내에서 생명 주기 함수를 사용할 수 없다. → 데이터를 수정할 필요가 없는 컴포넌트들에 대하여 사용할 수 있다.
{% endraw %}
