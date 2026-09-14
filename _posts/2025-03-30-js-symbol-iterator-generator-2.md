---
title: "Symbol/Iterator/Generator (2)"
date: 2025-03-30 19:56:00 +0900
categories: [javascript, es6]
tags: [javascript, iterator]
---

### For Loop

자바스크립트에는 많은 종류의 for 루프가 있다.

예) 명령형 프로그래밍

```javascript
for (let index = 0; index < array.length; index++) {
	const element = array[index];
}
```

예) 선언형 프로그래밍 (ES5)

```javascript
array.forEach(function(element) {
	console.log(element);
});

array.forEach(element => {
	console.log(element);
});
```

선언형으로 코드를 작성이 가능하지만 break, return으로 루프를 중단할 수 없다.

**`for .. in`**

```javascript
array = [1,2,3,4,5];

for (var index in array) {
	console.log(array[index]);	
}
```

`for .. in` 루프에서 index값에 할당되는건 숫자가 아닌 **문자열**이다. (’0’, ‘1’, ‘2’, ‘3’…) 또한 이 루프는 배열의 인덱스만 순회하는 것이 아닌, 프로토타입 체인을 포함한 모든 속성을 순회한다. 원래는 `for .. in`은 배열이 아니라 **객체**를 순회하기 위해서 만들어진 루프이다.

**`for .. of`**

```javascript
for (var value of array) {
    console.log(value);
}
```

for .. of 루프는 for .. in으로 배열을 순회할 때 생기는 모든 단점들을 해결했다. 

더 중요한 건 for .. of문이 배열만을 위한 것이 아니라, 모든 순회가능한 객체를 대상으로 사용할 수 있다는 것이다.

### Iterable

**순회가능한 객체**란 **Symbol.iterator 심볼**을 속성으로 가지고 있고, **iterator 객체를 반환**하는 객체를 뜻한다. 이런 스펙을 iterable 프로토콜이라고 하고, 이 프로토콜을 지킨 객체를 iterable 객체라고 한다.

### Iterator

그러면 iterable 객체가 `[Symbol.iterator]()` 메서드로 반환하는 iterator 객체는 무엇인가? `next()`메서드를 구현하고 있고, `done`과 `value` 속성을 가진 객체를 반환하는 객체이다. 이 스펙을 iterator protocol이라고 한다.

for of 루프는 순회를 시작하기 전 `[Symbol.iterator]()` 메서드를 호출하여 iterator 객체를 얻은 후 next() 메서드를 호출하면서 하나씩 순회하는 것이다. 보통은 iterable 프로토콜과 iterator 프로토콜을 하나의 객체에 모두 구현하는 것이 일반적이다.

예) 영원히 0을 반환하는 무한 이터레이터

```javascript
var zeroesIterator = {
    [Symbol.iterator]: function () {
        return this;
    },
    next: function () {
        return {done: false, value: 0};
    }
};
```

이터레이터 객체는 필요에 따라서 return 메서드나 throw 메서드를 구현할 필요도 있다. 

- return
  - for .. of 루프가 break, return 때문에 done이 true인 정상적인 상황보다 먼저 루프를 빠져나갈 때 호출한다. 주로 이터레이터 객체에 정리할 자원이 있는 경우 사용된다.
- throw
  - for .. of루프에서 발생한 예외를 이터레이터 안으로 전달하고 싶은 경우 사용

iterator를 사용하면 다양한 타입의 객체를 하나의 프로토콜로 순회하며 다룰 수 있고, 커다란 크기의 데이터도 메모리의 부담없이 Lazy-Evaluation이 가능하다는 성능상의 이점도 있다.

### Generator

제너레이터는 iterable/ iterator 객체를 만드는 손쉬운 방법이다.

예) iterable/iterator 프로토콜을 구현해서 순회하는 객체 생성

```javascript
class RangeIterator {
	constructor (start, stop) {
		this.value = start;
		this.stop = stop;
	}
	[Symbol.iterator()]() {
		return this;
	}

	next() {
		var value = this.value;

		if (value < this.stop) {
			this.value += 1;
			return {done : false, value : value}
		} 
 	}
}

function range(start, stop) {
	return new RangeIterator(start, stop);
}
```

RangeIterator 클래스를 통해 이터러블/이터레이터 프로토콜을 구현해서 순회하는 객체를 생성했다. 아래는 같은 내용을 제너레이터를 이용해서 구현한 코드이다.

```javascript
function* range(start, stop) {
    for (var i = start; i < stop; i++) {
        yield i;
    }
}

const generator = range(1, 4);

console.log(generator.next().value); // 1
console.log(generator.next().value); // 2
console.log(generator.next().value); // 3
console.log(generator.next().value); // undefined
```

제너레이터는 function\*으로 시작하고 yield문이 있는 함수다. 제너레이터 함수의 작동 순서는 다음과 같다.

1. 제너레이터 함수를 처음 실행하도 아무일도 일어나지 않는다. 그냥 제너레이터 객체가 리턴된다.
2. 제너레이터 객체는 next() 메서드를 실행할 때마다 다음 yield문까지 실행되고 정지한다. 다시 next()를 실행하면 정지된 부분에서 다음 yield까지 실행하고 다시 정지한다.
3. 더이상 yield가 없으면 제너레이터의 실행은 완전히 종료된다. (done : true)
