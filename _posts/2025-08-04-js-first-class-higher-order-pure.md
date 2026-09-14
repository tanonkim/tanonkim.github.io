---
title: "일급함수, 고차함수, 순수함수"
date: 2025-08-04 13:06:00 +0900
categories: [javascript, functional]
tags: [javascript, functional-programming]
---

## 일급 객체(First Class Object)

> 다른 객체들에 일반적으로 적용가능한 연산을 지원하는 객체

특징

- **변수에 담을 수** 있다.

```typescript
let test = function() {
	return "Hello World";
}

console.log(test()) // 변수 test의 값은 Hello World
```

- **파라미터로 전달**할 수 있다.

```typescript
let test = function() {
	return "Hello World";
}

let dev = function(param) {
	console.log(param);
}

dev(test());
```

- **함수의 반환 값으로 사용**할 수 있다.

```typescript
function test() {
	return function() {
		console.log("Hello World")
	}
}

let hello = test();
hello();
```

- **동적 프로퍼티** 생성이 가능하다.

```typescript
function test() {
	console.log("HelloWorld")
}

test.me = "tanon";
test.job = "developer";

console.log(test.me);
console.log(test.job);
```

## 일급 함수(First Class Function)

함수를 일급 객체로 취급하는 것이다.

자바스크립트의 경우 함수도 객체로 표현하기 때문에 일급 객체 및 일급 함수라고 부른다.

자바스크립트가 일급 객체이기 때문에 아래의 것들이 가능하다.

- 콜백함수
- 고차함수
- 클로저

## 고차함수

**함수의 형태로 리턴할 수 있는 함수**이다. 즉 함수가 함수를 다루는 경우이다.

**예) 함수를 인자**로 받는 고차 함수

```typescript
function greet(name){
	return `Hello, ${name}`;
}

function saySomething(func, name) { // func : 콜백함수
	console.log(func(name));
}

saySomething(greet, 'Tanon');
```

saySomething함수는 다른 함수(greet)를 인자로 받음 → 고차 함수

**인자로 전달된** greet함수와 같은 함수 : 콜백 함수

예) **함수를 리턴**하는 고차 함수(커링 함수)

```typescript
function multiplier(x) {
	return function(y){
		return x*y;
	}
}

const double = multiplier(2);
console.log(double(5)); // 10
```

multiplier 함수는 다름 함수를 리턴 → 고차함수

함수를 리턴하는 패턴을 커리 함수라고 함

**콜백함수** : 다른 함수의 인자로 전달되는 함수

→ 특정 조건/시점일 때 실행됨

예) 

```typescript
setTimeout(() => {
	console.log("triggered after 3000 ms")
}, 3000); 
```

### 참고) 내장 고차 함수

- Array.filter()

```javascript
 Array.prototype.filter = function(func) {
    const arr = this;
    const newArr = []
    for(let i = 0; i < arr.length; i++) {
      if (func(arr[i]) === true) {
        newArr.push(this[i])
      }
    }
    return newArr;
  }
```

- Array.map()

```typescript
const cartoons = [
  {
    id: 1,
    bookType: 'cartoon',
    title: 'title',
    subtitle: 'subtitle',
    averageScore: 9.66,
  },
  {
    id: 2,
    // .. 이하 생략
  },
  // ... 이하 생략
]; // 만화책의 모음

const findSubtitle = function (cartoon) {
  return cartoon.subtitle;
}; // 만화책 한 권의 제목을 리턴하는 로직(함수)

const subtitles = cartoons.map(findSubtitle); // 각 책의 부제 모음
const subtitles = cartoons.map((cartoon) => cartoon.subtitle); // 각 책의 부제 모음
```

- Array.reduce

여러 데이터를 하나로 응축할 때 사용

```typescript
const cartoonAvgScore = cartoons.reduce((sum, c) => sum + c.averageScore, 0) / cartoons.length;
```

### 왜 고차함수를 사용해야 할까? - 추상화

추상화는 복잡한 어떤 것을 압축해서 핵심만 추출한 상태이다.

우리는 자바스크립트의 문법을 올바르게 사용하는 것만으로, 다양한 프로그램을 쉽게 작성이 가능하다. → 고민거리가 줄고 비즈니스에 집중할 수 있다. 

### Closure와 Currying의 차이

### Closure

closure는 내부 함수에서 외부 함수의 값을 사용하는 기법/현상이다.

```javascript
function outerFunc() {
  let outerVal = 2;
  function innerFunc() {
    let innerVal = 1;
    return globalVal + outerVal + innnerVal;
  }
  return innerVal;
}
let globalVal = 3;
let innerF = outerFunc();
innerF();
```

outerFunc : 종료된 이후에서 outerVal에서 계속 접근이 가능함 → closure

globalVal : 전역변수

### Currying

함수에 n개의 인자를 받는 대신, n개의 클로저 함수를 만들어 각각 인자를 받게 하는 방식

예) non-currying

```typescript
function multiply(x,y,z) {
	return x*y*z;
}

console.log(multiply(1,2,3)); // 1*2*3
```

예) currying

```typescript
function multiply(x) {
	return function(y) {
		return function(x) {
			return x * y * z
		}
	}
}

let multiply3 = multiply(3);
let multiply3And2 = multiply3(2);
console.log(multiply3And5(1)); // 3*2*1

let multiply3And5 = multiply3(5);
console.log(multiply3And5(1)); // 3*5*1
```

## 순수함수

동일한 입력에 대해 항상 동일한 출력을 반환, 외부 상태를 변경하지 않음

→ 함수형 프로그래밍 : 순수함수 중심으로 Side-Effect가 없도록 프로그래밍하는 기법

ex) 더하기 함수

```javascript
function add(x, y) {
  return x + y;
}
```

이 함수는 인자로 전달받은 x와 y를 더한 결과를 반환한다. 이 함수는 동일한 인자에 대해 항상 동일한 결과를 반환하며, 함수 내부에서 어떤 값도 변경하지 않는다.

ex) 배열을 정렬하는 함수

```php
function sortArray(array) {
  return array.slice().sort();
}
```

이 함수는 입력으로 받은 array를 변경하지 않고, 새로운 배열을 생성하여 그 배열을 정렬한다. array.slice() 호출은 array 배열의 **사본**을 생성하기 때문에, array 배열 자체가 변경되는 것을 방지한다. 따라서, sortArray() 함수는 외부 상태를 변경하지 않고, 항상 같은 입력에 대해 같은 출력을 반환하므로 순수 함수다.

### 장점

- 불변성 유지 → 코드의 예측가능성과 안정성 높여줌
- 동일한 결과를 반환 → 디버깅, 테스트가 쉬워진다.
- 순수 함수는 병렬처리, 메모이제이션 → 성능 향상

### 사이드 이펙트

사이드 이펙트는 함수 외부의 상태를 변경하는 것을 말한다. 

예) Array.push()함수

```typescript
const arr = [1,2,3];
arr.push(4);
```

배열에 새로운 요소 추가 → 외부 상태를 변경하는 사이드 이펙트.

**Pure Method**

- Array.map()
- Array.reduce()
- Array.find()
- Array.filter()
- Array.concat()
- Array.slice()

**Impure Method**

- Array.push()
- Array.pop()
- Array.splice()
- Array.sort()
- Date.now()
- Math.random()
