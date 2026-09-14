---
title: "JavaScript 이벤트 루프 2"
date: 2025-03-22 13:20:00 +0900
categories: [javascript, async]
tags: [javascript, event-loop]
---

### (1) 내용 요약

싱글스레드인 자바스크립트에서 작업의 동시처리를 가능하게 하는 점은 이벤트 루프가 자바스크립트 엔진과 브라우저의 Web API를 연결하여 비동기적인 태스크를 처리하는게 가능하기 때문이다. 

단 모든 동작이 비동기처리가 가능한게 아닌, 비동기 전용 함수가 존재하고 setTimeout, fetch, addEventListener가 대표적인 예이다.

![](/assets/img/2025-03-22/js-event-loop-2-1.png)

### 이벤트 루프

이벤트 루프는 비동기 함수작업을 Web API에 옮기는 역할을 하고, 작업이 완료되면 callback을 Callback Queue에 적재했다가 다시 JavaScript 엔진에 적재해 수행시키는 **작업을 옮기는 역할**을 한다. 

작업을 처리하는 주체는 JavaScript 엔진과 Web API이다. 그래서 이벤트 루프는 CallStack에 현재 실행중인 작업이 있는지, MacroTask Queue에 대기중인 작업이 있는지 반복적으로 확인하는 일종의 무한 루프만을 돌고, 대기 작업이 있다면 작업을 옮겨주는 형태로 동작한다.

```javascript
// 이벤트 루프의 동작을 나타내는 가상의 코드
while (queue.waitForMessage()){ // 큐에 메세지가 있을 때까지 대기
	queue.processNextMessage();	// 가장 오래된 메세지를 큐에서 꺼내서 호출 스택으로 이동
}
```

### setTimeout 내부 동작

```javascript
function bar() {
  setTimeout(() => {
  	console.log("Second")
  }, 500);
}

function foo() {
  console.log("First");
}

function baz() {
  console.log("Third");
}

bar();
foo();
baz();
```

1. bar()가 호출되고, setTimeout()함수가 호출되어 CallStack에 쌓인다.
2. setTimeout()함수의 매개변수(파라미터)에 할당된 콜백함수를 Timer Web API에 전달하고, Timer Web API에서는 백그라운드로 500ms초 센다.
3. foo()함수가 호출되고 콘솔에 출력한다.
4. 이때 500ms 시간이 만료되면서 이벤트루프는 Timer Web API에서 가지고 있던 콜백함수를 MacroTask Queue로 옮긴다.
5. baz()가 호출되면서 Thrid가 출력된다.
6. 스택에 있는 메인 자바스크립트 코드가 실행완료되어 CallStack이 비워지게 된다.
7. 이벤트 루프는 CallStack이 비워짐을 탐지하여, MacroTask Queue에 있는 콜백 함수를 CallStack으로 옮긴다.
8. CallStack에서 콜백함수를 실행한다

⇒ 궁금했던 점 : 5ms로 지정해도 콜 스택이 비워지지 않는다면 대기 === 5ms 이상이 될 수 있다. (최소 5ms이후에 실행된다)

### 웹브라우저와 Node.js의 WebAPI 차이

이 둘은 구성은 비슷하지만 동작 측면에서 차이가 있다. 웹브라우저의 Web APIs는 비동기 작업이 끝나면 스스로 callback queue에 적재하지만, Node.js API는 이벤트 루프가 직접 옮겨준다. 

예) 

- Node.js : Timer API가 타이머 완료 이벤트를 발생시키고, 이벤트 루프가 이를 감지하여 MacroTaskQueue에 콜백함수를 추가한다.
- 웹브라우저 : Timer API가 스스로 Task Queue에 콜백함수를 추가한다.

### Promise 내부 동작 과정

**Callback Queue**

Callback Queue는 Web API가 수행한 비동기 함수를 넘겨 받아 EventLoop가 해당 함수를 CallStack에 넘겨주기 전까지 쌓아두는 곳이다. Callback Queue에는 MacroTask Queue, MicroTask Queue 두 종류가 있고, Promise의 객체의 콜백이 쌓이는 곳이 MicroTask Queue이다. 

MicroTask Queue는 그 어떤 곳보다 가장 우선적으로 콜백이 처리된다. (심지어는 브라우저 화면 렌더링 전보다 더 빠르게 처리된다.)

**MicroTask Queue 처리과정**

```javascript
console.log('Start!');

setTimeout(() => {
	console.log('Timeout!');
}, 0);

Promise.resolve('Promise!').then(res => console.log(res));

console.log('End!');
```

**답**

```
Start!
End!
Promise! // MicroTask Queue
Timeout! // MacroTask Queue
```

1. CallStack에 console.log('Start!');를 쌓고 콘솔창에 출력한다.
2. setTimeout이 CallStack에 적재되고 실행, 콜백함수가 이벤트 루프에 의해 WebAPI로 옮겨지고 타이머가 작동된다. (0ms라서 바로 타이머는 바로 종료된다)
3. 타이머 종료 시 setTimeout의 콜백함수는 MacroTask Queue에 이벤트 루프에 의해 저장된다.
4. Promise코드가 CallStack에 적재되어 실행, then의 콜백함수 인수가 이벤트 루프에 의해 MicroTask Queue에 저장된다.
5. CallStack에 console.log('End!');를 쌓고 콘솔창에 출력한다.
6. 메인 쓰레드에 자바스크립트 코드가 실행되어 CallStack엔 실행할 스택이 없다.
7. 이벤트핸들러가 이를 감지하여 Callback Queue에 남아있는 콜백함수를 CallStack에 저장한다.
8. MicroTask Queue, MacroTask Queue 순서로 콜백이 처리된다.
   1. Promise.then의 콜백함수는 MicroTask Queue에, setTimeout의 콜백함수는 MacroTask Queue에 저장된다.

### Async/Await 내부 동작 과정

JS의 Async/Await은 비동기 논블로킹 동작을 동기적으로 처리하기 위해 **ES7**부터 새롭게 도입하였다. 복잡한 콜백이나 then 핸들러의 지옥코드를 극복하는 핵심이다. 

```javascript
const One = () => Promise.resolve('One!');

async function myFunc(){
	console.log('In function!');
	const res = await One();
	console.log(res);
}

console.log('Before Function!');
myFunc();
console.log('After Function!');
```

**답**

```
1. Before Function!
2. In function!
3. After Function!
4. One!
```

1. 콘솔에 Before Function이 출력된다
2. myFunc()이 호출된다.
3. async 함수안에 있는 콘솔함수가 실행되어 콘솔 출력된다.
4. Promise 객체를 반환하는 One비동기 함수를 호출
5. One 함수의 **await에 의해 myFunc함수의 내부 실행은 잠시 중단**되고, CallStack에서 빠져나와 나머지 부분은 MicroTask Queue에 저장된다.

   이유 : JS엔진이 await 키워드를 인식하면 async 함수의 실행은 지연되는 것으로 처리하기 때문이다.

6. 콘솔에 After Function이 출력된다
7. 모든 메인 쓰레드에 코드가 실행되어 CallStack엔 스택이 비워지게 된다.
8. 이벤트핸들러가 이를 감지하고, MicroTask Queue에서 async함수를 빼와 CallStack에 저장한다.
9. Promise 객체의 결과를 식별자 res에 저장하고 출력한다.

- Async/Await 동작

```javascript
const one = () => Promise.resolve('One!');

async function myFunc(){
	console.log('In function!');
	const res = await one();
	console.log(res);
}

console.log('Before Function!');
await myFunc();
console.log('After Function!');
```

**답**

```
// Before Function
// In function
// One!
// After Fuction
```

await 키워드를 사용하는 Promise 반환 비동기 `one()`의 결과 콜백이 MicroTask Queue에 저장된다. 이때 MicroTask Queue는 메인 CallStack이 비워져야 이벤트 루프가 옮겨 처리한다. 

메인 CallStack에는 console.log('After Function!');이 남아있는데, 출력결과를 보면 await myFunc()가 먼저 처리되고 콘솔을 출력하고 있다. 그런데 console.log(’After Function!’)이 실행되기 전에 await myFunc()가 먼저 처리되고 콘솔을 출력한다.

**Async/Await의 동작**

Async/Await의 동작은 다음과 같다. 예를 들어 아래와 같은 코드가 존재한다.

```javascript
let x = await bar();
console.log(x);
console.log('Done');
```

위의 코드는 사실 다음과 같은 코드의 의미를 가진다.

```javascript
bar().then((x) => {
		console.log(x);
		console.lop('Done');
});
```

즉 await 키워드 다음에 나오는 동일 라인의 코드는 모두 `await bar()`의 `then` 핸들러의 콜백 함수로 들어간다는 뜻이다. 

실제 async/await 키워드는 promise.then() 메소드를 사용하지 않고도 비동기 코드를 작성하게 해주는 문법적인 편의 기능이다. 따라서 `async/await` 키워드는 `promise.then()` 메소드를 대체하는게 아니라, promise.then() 메서드를 간결하고 명확하게 작성하게 해주는 것이다.

따라서 최종 정리하자면 myFunc() 코드는 promise 핸들러로 교체하면 아래와 같고, 각 then의 핸들러들이 MicroTask Queue에 적재되면서 순서대로 출력된다.

```javascript
const one = () => Promise.resolve('One!');

async function myFunc() {
    console.log('In function!');
    const res = await one();
    console.log(res);
}

console.log('Before Function!');
await myFunc();
console.log('After Function!');

//-------변환 ------//;

const one = () => Promise.resolve('One!'); 

function myFunc(){
    console.log('In function!');
    return one().then(res => { // `await` 대신 `.then()`을 사용
        console.log(res);
    });
}

console.log('Before Function!');
myFunc().then(() => {  // myFunc()의 Promise가 완료된 후 실행됨
    console.log('After Function!');
});
```

예) then 체이닝

```javascript
const one = () => Promise.resolve('One!');

async function myFunc() {
    console.log('In function!');
    const res = await one();
    console.log(res);
}

console.log('Before Function!');
await myFunc();
console.log('After Function!');
console.log('After2 Function!');
console.log('After3 Function!');

// -- 변환 -- //
const one = () => Promise.resolve('One!'); 

function myFunc(){
    console.log('In function!');
    return one().then(res => {
        console.log(res);
    });
}

console.log('Before Function!');
myFunc().then(() => {
    console.log('After Function!');
}).then(() => {
    console.log('After2 Function!');
}).then(() => {
    console.log('After3 Function!');
});

```
