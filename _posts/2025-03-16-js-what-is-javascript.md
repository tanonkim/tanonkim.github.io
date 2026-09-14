---
title: "자바스크립트 언어는 어떤 언어인가?"
date: 2025-03-16 16:54:00 +0900
categories: [javascript, basics]
tags: [javascript, paradigm]
---

### 1. 자바스크립트는 무엇을 하는 언어?

브라우저를 제어하는 언어였으나, Node.js라는 새로운 환경의 등장으로 자체 어플리케이션을 만들 수 있게 된 언어, 프론트엔드 백엔드 모두 가능

**Java”Script”**

본래 스크립트 언어, 스크립트 언어 : 이미 존재하는 프로그래밍을 제어하기 위해 사용되는 언어

HTML : 기본 구조, CSS : 디자인, JS : 웹페이지의 동작

**V8 엔진**

C++로 작성된 자바스크립트 엔진으로 실행 속도를 획기적으로 빠르게 만듬. 독립형으로 개발되었기에 브라우저 밖에서도 JS를 사용할 수 있는 기반이 됨

Node.js

Node.js는 V8 엔진을 기반으로 하는 JS 런타임 프로그램. 브라우저라는 실행환경에서만 동작하는 언어지만, Node.js 실행환경 위에서도 작동이 가능해짐. (웹서버 구축 등.)

### 2. 어떻게 사용하는 언어?

명령형, 함수형, 객체지향 프로그래밍을 지향하는 멀티 패러다임 언어

명령형 VS 선언형 프로그래밍

명령형 : 내가 어떻게 할 것인지(절차적)

선언형 : 내가 어떤 일을 할 것인지 (함수형)

예) 명령형 프로그래밍

```javascript
const arr = ['yellow banana', 'red apple', 'black banana'];
function getBananas() {
  const bananas = [];
  for (let i = 0; i < arr.length; i++){
  	if (arr[i].includes('banana'))
      bananas.push(arr[i]);
  }
  return bananas;
}
getBananas(); // ["yellow banana", "black banana"]
```

예) 선언형 프로그래밍

```javascript
const arr = ['yellow banana', 'red apple', 'black banana'];
const getBananas = (array) => array.filter(el => el.includes('banana'));
getBananas(arr); // ["yellow banana", "black banana"]
```

### 3. 함수형 프로그래밍

JS의 함수는 `일급객체`

- 일급객체란?

  다음의 조건을 만족하는 객체를 일급 객체라고 한다.

  1. 무명의 리터럴로 생성할 수 있다. 즉 런타임에 생성이 가능하다.

     **의미**

     ```javascript
     // 익명함수 예시
     const sayHello = function () {
       console.log("Hello, world!");
     };

     sayHello(); // "Hello, world!"
     ```

     익명함수를 sayHello변수에 저장함 → 함수를 런타임에 생성하고 변수에 할당함

     ```javascript
     // 즉시 실행함수
     (function () {
       console.log("즉시 실행 함수!");
     })(); // "즉시 실행 함수!"
     ```

     즉시 실행함수 : 이름 없는 함수가 런타임에 생성되고 즉시 실행함

  2. 변수나 자료구조에 저장할 수 있다.
  3. 함수의 매개변수에 전달할 수 있다.
  4. 함수의 반환값으로 사용할 수 있다.

     **의미**

     함수가 또 다른 함수를 반환할 수 있다

     ```javascript
     function createMultiplier(multiplier) {
       return function (num) {
         return num * multiplier;
       };
     }

     const double = createMultiplier(2);
     console.log(double(5)); // 10
     ```

일급객체로서 함수가 가지는 가장 큰 특징 : 함수의 매개변수에 전달, 함수의 반환값으로도 사용할 수 있다.

**함수형 프로그래밍의 필수 요소**

1. 순수함수를 사용하여 side-effect를 지양

   **순수함수 VS side-effect**

   순수함수 : side-effect가 없으며 인풋에 대한 아웃풋이 항상 같은 함수

   side-effect : 함수 외부의 데이터를 변경하는 것

   예) pureFunc, sideEffect

   ```javascript
   let number = 0

   function pureFunc(num, offset) {
       return num + offset
   }

   function sideEffect(offset) { // 외부 값 조작
     number += offset;
     return number;
   }

   console.log(pureFunc(number, 1)); // 1
   console.log(number); // 0
   console.log(sideEffect(1));
   console.log(sideEffect(1));
   console.log(number); // 2
   ```

   - pureFunc : 외부의 상태를 변경하지 않는 함수
   - sideEffect : 외부 상태에 의존하기 때문에 같은 인자에 대해 다른 값을 반환이 가능

   sideEffect는 상태 변경을 추적하기 어렵게 만들고, 가독성을 해친다.

   프론트엔드 작업에서 필수적인 작업인 DOM 조작, 서버와의 통신도 side-effect에 해당됨

   ⇒ 순수함수만 사용하는 것은 불가능함.

   결론 : 최대한 순수함수 위조로 작성하여 의도치 않은 상태 변경을 막고 코드를 간결하게 만드는 것이 JS의 함수형 프로그래밍

2. 데이터의 불변성을 유지

   **데이터의 불변성**

   JS에서 함수에 인수를 전달할 때 원시값(number, string, boolean)은  passed by value로 전달

   객체는 passed by reference방식으로 전달됨 → 객체를 함수에 전달하면 객체의 내부를 바꾸는게 가능함

   예) 객체 변경

   ```javascript
   const obj = {name : "aaaaa", id: "123"};
   const addNickname = (o) => {
     o.nickname = "bbbbb";
   }
   addNickname(obj);
   console.log(obj); // {name: "aaaaa", id: "123", nickname: "bbbbb"}
   ```

   순수함수는 외부의 상태(obj)를 변경해서는 안됨

   예) 데이터의 불변성 유지_순수함수로 작성하기

   ```javascript
   const obj = {name : "aaaaa", id: "123"};
   const addNickname = (o) => {
     return {...o, nickname : 'bbbbb'}
   }

   const newObj = addNickname(obj)
   console.log(obj); // {name: "aaaaa", id: "123"}
   console.log(newObj); // {name: "aaaaaa", id: "123", nickname: "bbbbb"}
   ```

3. 순수함수를 조합하여 선언적 패턴을 가질 것

   **선언적 패턴을 통한 관심사 분리**

   예) 버튼을 클릭하면 증가하는 React 코드

   ```javascript
   function App() {
     const [count, setCount] = useState(0);
     const onClick = () => setState(count + 1);

     return <div>
       <div>{count}</div>
       <button onClick={onClick}>+</button>
     </div>
   }
   ```

   예) 만약 숫자가 감소하는 버튼을 넣는다면

   ```javascript
   function App() {
     const [count, setCount] = useState(0);
     const onIncreaseClick = () => setState(count + 1);
     const onDecreaseClick = () => setState(count - 1);

     return <div>
       <div>{count}</div>
       <button onClick={onIncreaseClick}>+</button>
       <button onClick={onDecreaseClick}>-</button>
     </div>
   }
   ```

   외부의 상태 count를 변경하는 함수가 두개나 생김

   예) 관심사의 그룹화

   ```javascript

   function App() {
     const [count, setCount] = useState(0);

     const increase = (value) => value + 1;
     const decrease = (value) => value - 1;

     const onClick = (calculate) => setCount((prev) => calculate(prev));

     return (
       <div>
         <div>{count}</div>
         <button onClick={() => onClick(increase)}>+</button>
         <button onClick={() => onClick(decrease)}>-</button>
       </div>
     );
   }
   ```

### 4. 프로토타입 기반 객체지향 프로그래밍

Java : 클래스 기반 객체지향

JS : 프로토타입 기반 객체지향 프로그래밍 지원

JS에서 prototype을 통해 생성된 instance는 prototype에 메서드를 두고 instance와 연결해서 사용하는 방식을 사용한다.

```javascript
function Bird() {
  this.eat = function () {
    console.log('eat');
  }
}

Bird.prototype.fly = function () {
  console.log('fly!');
}

const bird = new Bird();
console.log(bird); // Bird {eat: ƒ} // instance 내부에는 eat 함수밖에 없다.

bird.eat(); // eat
bird.fly(); // fly! : prototype 에만 선언된 fly 함수를 instance에서 사용할 수 있다.

Bird.prototype.jump = function () {
  console.log('jump');
}

bird.jump(); // jump : instance가 생성된 후에도 prototype 에 추가된 함수를 사용할 수 있다.

console.log(bird.__proto__); // { fly: f, jump: f }
```

인스턴스 내부에서는 this.eat이 포함되지만 프로토타입 체인을 통해 fly 사용이 가능함

이를 활용하면 메모리 절약, 같은 동작을 하는 메서드 공유가 가능함.
