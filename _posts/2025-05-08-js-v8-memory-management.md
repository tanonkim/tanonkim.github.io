---
title: "JavaScript의 메모리 관리"
date: 2025-05-08 21:26:00 +0900
categories: [javascript, v8]
tags: [javascript, garbage-collection]
---

C와 같은 저수준 언어에서 메모리 관리를 위해 `malloc()`과 `free()` 를 사용한다. 반면 JavaScript는 객체가 생성되었을 때 자동으로 메모리 할당을 하고, 더 이상 필요하지 않는다면 자동으로 가비지 컬렉터에 의해 해제된다.

## 메모리 생존주기

메모리 생존주기는 대부분 프로그래밍 언어에서 비슷하다.

1. 필요할 때 할당한다.
2. 할당된 메모리를 사용한다. (읽기, 쓰기)
3. 더 이상 필요하지 않으면 해제한다.

2번은 모든 언어에서 명시적으로 사용된다. 1번, 3번은 저수준 언어에서 명시적이며, JavaScript와 같은 고수준 언어에서는 암묵적으로 작동한다.

## JavaScript의 메모리 할당

예) 값 초기화 시 메모리 할당

```javascript
const n = 123;      // 정수 할당을 위한 메모리 할당 
const s = 'string'; // 문자열을 담기 위한 메모리 할당

const 0 = {         // 오브젝트와 값을 담기 위한 메모리 할당
	a : 1;
	b : null;
}

const a = [1, null, 'string']; // 배열, 값을 담기 위한 메모리 할당

function f(a) { // 함수를 위한 메모리 할당 (호출 가능한 오브젝트)
	return a + 2;
}

someElement.addEventListener( // 함수식 또한 오브젝트를 담기 위한 메모리 할당
	"click",
	() => {
		someElement.style.backgroundColor = "blue";
	},
	false
)
```

예) 함수 호출 시 메모리 할당

```javascript
const d = new Date(); // Date 객체를 위한 메모리 할당
const e = document.createElement("div"); // DOM 요소를 위한 메모리 할당
```

예) 새로운 값이나 오브젝트 할당

```javascript
const s = "string"
const s2 = s.substr(0,3); // 새로운 문자열

const a = ['1', '2'];
const b = ['3', '4'];
const c = a.concat(b); // 4개의 원소를 가진 새로운 배열
```

JavaScript에서 문자열은 immutable(불변)한 값이기 때문에, 메모리 할당을 하지 않고 [0,3]의 범위만 저장

## 가비지 콜렉션

### 참조

가비지 콜렉션 알고리즘의 핵심 개념은 참조이다. A 메모리를 통해 B메모리를 접근할 수 있다면 B는 A에 의해 참조된다라고 한다.

예) : 모든 Javascript 오브젝트는 prototype 및 그 객체의 속성에 대한 참조가 있다.

### Reference-counting 가비지 콜렉션

- 참조-세기 가비지 콜렉션

가장 단순하게 구현된 알고리즘으로, 더 이상 어떤 객체도 참조하지 않는 객체는 가비지라고 여긴다. 이를 참조하는 다른 객체가 하나도 없는 경우 수집이 가능하다.

예)

```javascript
// Case 1. x객체 a객체 생성 -> 카비지 콜렉션이 수행될 메모리는 없음
let x = {
	a : {
		b : 2, // x → { a → { b: 2 } }
	}
}

// 2.
let y = x; // y, x → { a → { b: 2 } }

x = 1 // y가 참조하는 유일한 변수, x → 1, y → { a → { b: 2 } }

// 3. 
let z = y.a // {b : 2}

// 4. 
y = "string" // a변수는 여전히 z에 의해 참조됨 (메모리 해제 X)

z = null // a속성에 참조가 없어서 메모리 해제 가능
```

참고 : 순환참조는 메모리 누수의 흔한 원인이다

예)

```javascript
x.a = y
y.a = x
```

### Mark-and-sweep 알고리즘

- 표시하고 쓸기 알고리즘

더 이상 필요없는 객체를 도달할 수 없는 객체로 정의한다.

roots라는 객체의 집합을 가지고 있는데, roots로 부터 시작하여 참조하는 객체들, 참조하는 객체가 참조하는 객체 등을 찾는다. → 도달할 수 없는 모든 객체들을 수집한다. 참조-세기(Refence-counting)알고리즘의 개선 형태인데, 참조가 없는 객체는 명확히 도달할 수 없기 때문이다.

장점

헤딩 방식은 주기가 문제가 되지 않는다. 함수 호출이 리턴되고, 두 객체는 더 이상 전역 객체에서 참조하고 있는 어떤 리소스에서도 참조하지 않는다. → 할당되었던 메모리를 회수한다.

단점

수동으로 조작할 수 없는 문제가 있다. 객체의 메모리 반환을 위해 명시적으로 도달할 수 없게 만들어야 한다.

## 메모리 관리를 돕는 데이터 구조

JS가 가비지 수집기 API를 직접적으로 노출하지 않더라도, 간접적으로 가비지 수집을 관찰하며 메모리 관리에 쓸 수 있는 데이터 구조를 제공한다.

### 1. WeakMaps / WeakSets

WeakMap ↔ Map, WeakSet

각 Map, Set을 모방한 데이터 구조이다.

- WeakMap : 키-쌍 모음
- WeakSet : 유일한 값들로 이뤄진 모음

WeakMap과 WeakSet의 키는 프로그램의 어떤 것도 키를 참조하지 않는다면 가비지 콜렉션이 될 수 있다.

### 2. WeakRefs와 FinalizationRegistry

가비지 수집 구조에 대한 직접적인 내부 탐색을 제공하지만, 런타임 시맨틱은 완전히 보장하지 않기에 사용하지 않는 것이 권장된다.

WeakRef의 사용 사례 : 문자열 URL을 큰 객체에 매핑하는 캐시 시스템

FinalizationRegistry : 객체를 등록하고 가비지 수집이 되었을 때 알림을 받을 수 있도록 함
