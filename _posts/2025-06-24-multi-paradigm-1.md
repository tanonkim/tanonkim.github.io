---
title: "멀티패러다임 프로그래밍 1장. 멀티패러다임이 현대 언어를 확장하는 방법"
date: 2025-06-24 22:12:00 +0900
categories: [javascript, multi-paradigm]
tags: [multi-paradigm, typescript]
---

## 멀티패러다임의 시대

대부분의 프로그래밍 언어는 멀티 패러다임 언어로 발전했다. 과거에는 함수형 프로그래밍을 위해 함수형 언어를, 객체지향 프로그래밍을 위해 객체지향 언어를 선택해야 했지만, 하나의 언어에서 명령형/객체지향/함수형 패러다임을 모두 활용할 수 있다.

## 멀티 패러다임의 협력

함수형, 객체지향, 명령형 패러다임을 모두 제공하는 멀티패러다임 언어를 효율적으로 사용하는 방법을 배워야 하며, 여러 패러다임을 섞어야 할 때이다. 각 패러다임은 그 자체로 유용하지만, 특정 문제에 가장 적합한 패러다임을 선택하거나 필요하다면 `조합할 수 있는 유연성`이야 말로 멀티패러다임의 장점이다.

예) 객체지향 프로그램이은 상태와 행동을 한데 묶는데 유리하고, 함수형 프로그램이은 순수 함수와 불변성을 통해 코드의 예측 가능성을 높인다.

## 반복자 패턴과 일급 함수

**객체지향 디자인 패턴 중 하나**인 `반복자 패턴`은 컬렉션 요소를 순차적으로 접근하는 규약을 제시한다. 이를 통해 다양한 형태의 컬렉션 데이터를 일관된 방식으로 순회할 수 있다.

**함수형 프로그래밍의 핵심 요소인 일급함수**는 함수를 값처럼 취급해 다른 함수의 인자로 전달하거나 반환값으로 사용할 수 있다.

이 두개념을 결합하면 멀티패러다임 언어에서 함수형 프로그래밍의 핵심인 `지연평가`와 `리스트 프로세싱`을 효율적으로 구현이 가능하다.

### 1.1 객체지향 디자인 패턴의 반복자 패턴과 일급 함수

일급함수가 추가되면서 이를 바탕으로 map, filter, reduce, take와 같은 이터레이터 헬퍼 함수들이 구현될 수 있었다.

**GoF의 반복자 패턴 / Iterator**

반복자 패턴은 객체지향 디자인 패턴 중 하나이다.

예) iterator 인터페이스

```typescript
interface IteratorYieldResult<T> {
	done?: false;
	value: T;
}

interface IteratorReturnResult {
	done: true;
	value: undefined
}

interface Iterator<T> {
	next() : IteratorYieldResult<T> | IteratorReturnResult
}
```

반복자 패턴은 컬렉션의 내부 구조를 노출하는 대신 next()와 같은 public메서드로 내부요소에 접근할 수 있도록 설게되었다.

예) ArrayLike로부터 Iterator를 생성하는 클래스

```typescript
class ArrayLikeIterator<T> implements Iterator<T> {
	private index = 0;
	constructor(private arrayLike: ArrayLike<T>) {}
	
	next() : IteratorResult<T> {
		if (this.index < this.arrayLike.length) {
			return {
				value : this.arrayLike[this.index++],
				done : false
			};
		} else {
			return {
				value : undefined,
				done : true
			}
		}
	}
}

const arrayLike:ArrayLike<number> = {
	0:10,
	1:20,
	2:30,
	length : 3
}

const iterator : Iterator<number> = new ArrayLikeIterator(arrayLike);

console.log(iterator.next());
console.log(iterator.next());
console.log(iterator.next());
console.log(iterator.next()); // {value : undefined, done : true}
```

이터레이터는 필요할 때마다 값을 하나씩 꺼내는 `지연평가`를 지원한다.

## 지연 평가와 고차함수

`지연평가`는 필요한 시점까지 계산을 미루어 메모리 사용량과 연산 비용을 줄여 성능을 향상하는 기법이다. (44p)

`고차함수`는 함수를 인자로 받거나 반환하는 함수로써 map, filter, reduce와 같은 함수형 함수들은 지연 평가와 고차 함수를 활용해 더 나은 성능과 가독성을 제공한다.

`일급함수` : 함수를 값으로 다루는 함수

함수를 값으로 다루어 변수에 담거나 다른 함수의 인자로 전달하거나 함수의 반환값으로 사용할 수 있는 함수, 일급함수의 이런 특징으로 고차함수를 만들 수 있음

반복자 패턴의 지연성은 지연 평가가 가능한 객체를 생성하게 해주고, 일급 함수는 고차함수를 정의할 수 있도록 한다.

## 제너레이터 함수

`제너레이터` 함수는 **이터레이터를 생성**하는 문법적 장치로서 `명령형` 스타일로도 이터레이션 로직을 구현하게 한다. funtion\*키워드와 yield 키워드를 사용해 하나씩 반환함으로써 지연 평가가 필요한 로직을 간결하고 명확하게 표현할 수 있다.

**예) 간단한 generator 함수**

```typescript
function* generator() {
	yield 1;
	yield 2;
	yield 3;
}

const iter = generator();

console.log(iter.next());
console.log(iter.next());
console.log(iter.next());
console.log(iter.next()); // {value : undefined, done : true}
```

done 속성이 ture가 될 때까지 제너레이터는 yield 키워드가 있는 지점까지 실행하고 값을 반환하고 일시중지하는 과정을 반복한다.

## 타입스크립트와 이터레이션 프로토콜

이터레이션 프로토콜은 자바스크립트의 규약이다. ES6에서 도입된 이터레이션 프로토콜은 어떤 객체가 이터러블인지 여부를 나타내는 규칙과 해당 규칙을 따르는 문법을들 제공하는 언어 전반의 규약이다.

이터러블 객체는 이터레이터를 통해 순회가능한 구조를 가지고 있다. for … of문, 전개 연산자, 구조 분해 할당 등과 함께 활용하면 유연한 코드 작성이 가능하다. 타입스크립트는 이러한 `이터레이션 프로토콜`에 대해 정교한 타입 시스템을 제공하여 각 요소의 타입을 명확히 정의하고 코드 안정성을 한층 강화한다.

**이터레이터와 이터러블**

이터러블 : 어떤 객체가 이터레이터를 반환하는 \[Symbol.iterator\]() {return {next() {…}};} 메서드를 가지고 있으면 이터러블이다.

이터러블 객체는 반복자 패턴의 특성을 모두 갖추고 있고, 대표적으로 Array, Set, Map 등이 있다.

예)

```typescript
function naturals(end = Infinity) : IterableIterator<number> {
	let n = 1;
	return {
		next() : IteratorResult<number> {
			return n <= end
				? {value : n++, done : false} 
				: {value : undefined, done : true};
		},
		[Symbol.iterator]() {
			return this;
		}	
	};
}

const iterator = naturals(3);

for (const num of iterator) {
	console.log(num)
}
```

**Iterator, Iterable, IterableIterator, 이터레이션 프로토콜 정리**

Iterator : {value, done} 객체를 반환하는 next() 메서드를 가진 값

Iterable : iterator를 반환하는 \[Symbol.iterator\]() {} 메서드를 가진 값

IterableIterator : 이터레이터이면서 이터러블인 값

이터레이션 프로토콜 : 이터러블을 for…of문, 전개 연산자, 구조분해 할당 등과 함께 동작하도록 한 **규약**

→ 객체지향 디자인 패턴 중 하나로, 반복자 패턴에 기반함

**내장 이터러블 (Array, Map, Set)**

```typescript
// 1. array
const array = [1,2,3];
const arrayIterator = array[Symbol.itertator]();

// 2. set
const set = new Set([1,2,3]);
const setIterator = set[Symbol.iterator]();

// 3. map
const map = new Map(['a', 1], ['b', 2], ['c', 3]);
const mapIterator = map[Symbol.iterator]();
```

**구조 분해 할당과 이터러블**

```typescript
// 구조분해할당 1
const array = [1,2,3];
const [first, second] = array;

console.log(first) // 1
console.log(second) // 2

// 구조분해할당 2
const array = [1,2,3,4];
const [head, ...tail] = array;

console.log(head); // 1
console.log(tail); // [2,3,4]

// 구조분해할당 3
const map = new Map();
map.set('a', 1);
map.set('b', 2);
map.set('c', 3);

for (const [key, value] of map.entries()) {
	console.log(`${key} : ${value}`);
}
```

## 사용자 정의 이터러블

개발자는 직접 만든 객체에 이터레이션 프로토콜을 구현하여 사용자 정의 이터러블을 만들 수 있다. 이를 통해 언어의 기본 기능과 긴밀히 협업하고 다양한 문제 해결 방식을 시도하거나 언어 자체를 확장하는 것도 가능하다.

**제너레이터로 만든 map함수**

```typescript
function* map<A,B>(
	f: (value : A) => B, iterable : Iterable<A>
): IterableIterator<B> {
	for (const value of iterable) {
			yield f(value);
	}
}
```

map함수는 f함수와 이터러블 객체를 인자로 받는다. 그 후 이터러블 객체의 각 요소에 f함수를 적용한 결과를 yield키워드로 반환한다.

**이터러블을 다루는 함수형 프로그래밍 (forEach, map, filter)**

forEach, map, filter 세 함수를 다양한 방식으로 구현, 이를 통해 반복자 함패턴을 활용하고, 이터레이션 프로토콜을 준수하는 함수를 구현하는 방식에 익숙해지기.

1. forEach함수

   ```typescript
   function foeEach(f, iterable) {
   	for (const value of iterable) {
   		f(value);
   	}
   }

   const array = [1,2,3];
   forEach(console.log, array);
   ```

   예) forEach함수를 while 루프를 통해 이터레이터를 직접 조작

   ```typescript
   function forEach(f, iterable) {
   	const iterator = iterable[Symbol.iterator]();
   	let result = iterator.next();
   	while (!result.done) {
   		f(result.value);
   		result = iterator.next();
   	}
   }

   const set = new Set([4,5,6]);
   forEach(console.log, set);
   ```

   참고 : set도 이터러블 객체이다.

2. map함수

   ```typescript
   function* map(f, iterable) {
   	for (const value of iterable) {
   		yield f(value);
   	}
   }

   const array = [1,2,3];
   const mapped = map(x => x*2, array);
   console.log([...mapped])

   const mapped2 = map(x => x*3, naturals(3));
   forEach(console.log, mapped2);
   ```

   예) function* map(f, iterable) {while}

   ```typescript
   function* map(f, iterable) {
   	const iterator = iterable[Symbol.iterator]();
   	while (true) {
   		const {value, done} = iterator.next();
   		if (done) break;
   		yield f(value);
   	}
   }

   const mapped = map(([k,v]) => `${k} : ${v}`, new Map([['a',1],['b',2]]))
   forEach(console.log, mapped);
   ```

   무한루프 생성 → next()결과 구조분해 할당 → done이 true라면 break → value에 인자로 받은 f를 적용한 결과를 yield키워드로 반환

3. filter함수

   ```typescript
   function* filter(f, iterable) {
   	for (const value of iterable) {
   		if (f(value)) {
   			yield value;
   		}
   	}
   }

   const array = [1,2,3,4,5];
   const filtered = filter(x=> x%2 === 0, array);
   console.log([...filtered]);
   ```

## 상속 대신 인터페이스

상속은 객체지향 프로그래밍에서 중요한 개념이지만, 다양한 자료구조를 일관성있게 처리하려면 인터페이스와 이터레이션 프로토콜을 활용하는게 효율적일 수 있다.

예를 들어 NodeList, Map, Set등은 서로 다른 특성과 동작 방식을 지닌다. 하지만, 상속 대신 인터페이스를 통해 공통 로직을 공유하고, `반복자 패턴`에 기반한 이터레이션 프로토콜을 활용하면 (상속X) 다양한 자료구조를 유연하게 다룰 수 있다.

**인터페이스, 상속**

상속 : 기존 클래스의 구성과 구현 모두를 물려받는 클래스 상속

인터페이스 : `시그니쳐`만 정의하고, 구현 클래스나 리터럴 객체가 채우도록 하는 설계

**상속이 아닌 인터페이스로 해결해야 하는 이유**

1. 디커플링/커플링

   OCP원칙 : 확장에는 열려있어야 하고(기능을 추가할 수 있어야 하고), 변경에는 닫혀야 한다. (수정하지 않고 새 기능이 추가되어야 한다.)

   interface 사용 시 OCP를 더 쉽게 충족 가능함 → Composition 패턴 사용

2. 이터러블을 사용해야 하는 이유

   Q1 : 이터러블 대신 Array의 map, filter를 사용하면 되지 않을까?

   자바스크립트에서 Array는 filter, map, forEach와 같은 고차 함수를 지원한다.

   예) `NodeList`에서 Array.prototype.map을 시도하는 예제

   ```typescript
   const nodes : NodeList = document.querySelectorAll('li');

   console.log(nodes[0], nodes[1], nodes.length);

   nodes.map(node => node.textContent); // node.map is not a function
   ```

   A1 : 오류가 발생한다.

   Q2 : 순회가 필요한 자료구조인데, 왜 Array를 상속받도록 만들지 않았을까?

   **자바스크립트나 타입스크립트의 표준 라이브러리에서 Array를 상속받은 내장 클래스는 없다.**

   Map, Set, NodeList는 Array와 동일한 기능이 필요하다고 해도 Array가 없다.

   A2 : 모두 서로 다른 자료구조를 나타내며, 각각 고유한 특성과 동작을 갖도록 설계 되었기 때문이다.

   **상속으로 연결해서 의존성을 만들게 하면 복잡성이 생기고, 최적화된 동작 보장 X**

   Q3 : NodeList는 index, length를 가진 ArrayLike인데 왜 상속을 받지 않을까?

   NodeList는 DOM과 관련되어있다. 라이브 모드, 스태틱 모드가 존재하는데 라이브 NodeList는 DOM이 변경될 때 자동으로 업데이트 된다.

   Array는 생성된 후에는 정적이며, 항상 수동으로 요소를 추가하거나 제거해야 한다. → 다른 구조와 용도가 의존성을 갖게 되면 복잡성이 발생하고, 최적화가 어려워진다.
