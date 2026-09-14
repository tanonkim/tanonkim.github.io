---
title: "React 18 - Auto Batching"
date: 2025-05-12 21:26:00 +0900
categories: [react, react-study]
tags: [react, batching]
---

{% raw %}
> state가 많이 변경되어도 리렌더링이 한 번만 일어나는 이유

setState 함수는 리렌더링을 일으키는데, 여러번을 실행하도 한 번만 리렌더링이 발생한다. React에서 Batching을 일괄적으로 처리하는데, React 18에서 소개하는 Automatic Batching은 기존의 Batching을 개선하였다.

### Batching이란

state값이 변경되었을 때 React에서는 해당 컴포넌트를 리렌더링하며, 불필요한 리렌더링을 방지하기 위해 state를 변경하는 작업을 일괄적으로 처리한다.

업데이트 작업을 모아 처리하는 방식을 Batching이라고 하며, 이 덕에 불필요한 리렌더링을 방지할 수 있다.

예)

```javascript
import { useState } from "react";

function Counter() {
	const [count, setCount] = useState(0);

	function increaseCountThreeTimes() {
		// 아래의 작업은 모두 일괄적으로 처리되기에 한 번만 리렌더링이 발생한다
		setCount((prev) => prev + 1);
		setCount((prev) => prev + 1);
		setCount((prev) => prev + 1);
	}

	return (
		<div>
			<button onClick={increaseCountThree}>+1</button>
			<p>Count : {count}</p>
		</div>);
}

export default Counter;
```

위의 함수는 세 차례의 setState가 발생하지만, 실제로는 한 번의 리렌더링만 발생한다. state update의 작업을 Queue에 넣고, 순차적으로 일괄 시행하면서 불필요한 리렌더링을 방지한다.

### Automatic Batching (React 18)

React 18 이하에서는 오직 이벤트 핸들러 내부에서의 state update만 batch 처리가 가능했다.

즉 `Promise`, `setTimeout`, `Native DOM Event Handler` 내부에서의 state 변경 시 각각 렌더링을 유발했다.

이유 : 브라우저의 이벤트가 실행되는 동안만 Batch 작업을 수행했기 때문이다. 이벤트가 종료된 후에 실행되는 경우 Batch 작업이 불가능했다.

예) batch되지 않은 렌더링 (17 이하 버전)

```javascript
function App() {
  const [count, setCount] = useState(0);
  const [flag, setFlag] = useState(false);

  function handleClick() {
    fetchSomething().then(() => {
      // React 18 미만의 버전에서 batch처리 하지 않음
      // 이벤트가 실행되는 시점이 아닌 이벤트가 종료되는 시점에 실행됨 (메인 스택이 비워진 후)
      setCount(c => c + 1); // Causes a re-render
      setFlag(f => !f); // Causes a re-render
    });
  }

  return (
    <div>
      <button onClick={handleClick}>Next</button>
      <h1 style={{ color: flag ? "blue" : "black" }}>{count}</h1>
    </div>
  );
}
```

React 18버전 이후부터는 이벤트 핸들러 뿐만이 아닌, Proimse, SetTimeout, Native DOM event handler와 같은 작업에서도 batching작업을 자동으로 수행하게 해준다.

`ReactDOM.createRoot()` 메서드를 기반으로 렌더링을 진행하는 경우 state update는 자동으로 batch처리된다. : Automatic Batching

```javascript
function App() {
  const [count, setCount] = useState(0);
  const [flag, setFlag] = useState(false);

  function handleClick() {
    fetchSomething().then(() => {
      // React 18 and later DOES batch these:
      setCount(c => c + 1);
      setFlag(f => !f);
      // 한번만 리렌더링 (that's batching!)
    });
  }

  return (
    <div>
      <button onClick={handleClick}>Next</button>
      <h1 style={{ color: flag ? "blue" : "black" }}>{count}</h1>
    </div>
  );
}
```

### Batch의 그룹

중요한 것은 각각의 작업이 batch된다는 점이다.

예) trigger()

```javascript
import { useLayoutEffect, useRef, useState } from 'react';

function App() {
  const [test, setTest] = useState(1);
  const testRef = useRef<any>(0);

  const trigger = () => setTest((prev) => {
    console.log(prev + 1);
    return prev + 1;
  });


  const changeState = async () => {
    if (testRef.current) {
      clearTimeout(testRef.current);
      testRef.current = null;
    }

			// Batch 1
      // 한 개의 update function 은 하나로 batching 됨.
      trigger();
      trigger();

		  // Batch 2
      // 1.5초 후에는 4개의 trigger()을 하나로 batching 시킴
      setTimeout(() => {
        trigger();
        trigger();
      }, 1500);
      setTimeout(() => {
        trigger();
        trigger();
      }, 1500);

			// Batch 3
      testRef.current = setTimeout(() => {
        // 1초 후, 두 개의 trigger()를 하나로 batching 시킴
        trigger();
        trigger();
      }, 1000);
      return;
  }

  useLayoutEffect(() => {
    console.log('render');
  })

  return (<div>
    	<button onClick={changeState}>change it</button>
		<p> {test}</p>
	</div>);
}

export default App;
```

- 딜레이가 같은 두 setTimeout 내의 함수도 batching처리가 된다.
- 총 3차례의 배치 처리가 발생한다.

업데이트 일괄 처리의 시점

### ReactDOM.flushSync() 란?

Auto Batch를 무시하고 즉시 DOM을 렌더링한다. 필요할 상황이 있을 경우에만 사용해야 한다.

```javascript
import { flushSync } from 'react-dom'; // Note: react-dom, not react

function handleClick() {
  flushSync(() => {
    setCounter(c => c + 1);
  });
  // React has updated the DOM by now
  flushSync(() => {
    setFlag(f => !f);
  });
  // React has updated the DOM by now
}
```

상태 우선순위
{% endraw %}
