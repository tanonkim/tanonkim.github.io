---
title: "멀티패러다임 프로그래밍 2장. 함수형 프로그래밍과 타입 시스템, LISP"
date: 2025-06-30 21:30:00 +0900
categories: [javascript, multi-paradigm]
tags: [multi-paradigm, typescript]
---

### 타입스크립트의 타입 추론과 함수 타입

타입스크립트는 자바스크립트에 타입을 추가함으로써 코드의 안정성 + 가독성을 높인다.

타입 추론 기능으로 명시적인 타입 선언 없이도 안전한 코드 작성이 가능, 변수와 함수의 변환 타입을 자동으로 추론한다. let a = 10; → a가 number로 추론, (a : number, b : number) ⇒ a+b; 와 같은 함수에 반환 타입이 자동으로 number로 결정됨

### 변수와 상수의 타입 추론

변수와 상수를 초기화할 때. 타입스크립트는 해당 값으로 부터 타입을 추론함.

const selected = true; → selected는 리터럴 타입

let checked = true; → checked는 boolean타입

### 제네릭을 통한 타입 추론

제네릭 함수는 다양한 타입에 대응할 수 있는 다형적인 함수를 구현하는데 활용.

예를 들어 `function identity<T>(arg:T): T;`와 같은 제네릭 함수는 인자의 타입에 따라 반환 타입을 유연하게 결정함

**예)**

```typescript
function identity<T>(arg: T): T {
	return arg;
}

const a = identity("hi");
const b = identity(1);
const c = identity<string>("a");
const d = identity<number>(1);

class User {}
const e = identity(new User()); // [const e: User]

const f = identity((n : number) => n % 2 === 1); // [const f: (n : number) => boolean]
```

### 함수 타입과 제네릭

타입스크립트는 함수형 프로그래밍을 지원하기 위해 고차 함수와 함수 타입, 제네릭과 같은 기능을 제공. 이를 통해 함수의 입력과 출력 타입을 명확히 하고 다양한 타입을 수용하는 범용적인 함수를 손쉽게 구현할 수 있다.

### 이터러블 헬퍼 함수 (코드 모습을 추상화)

이터러블 헬퍼 함수는 이터러블을 중심으로 고차 함수를 구성하는 패턴이다. 타입스크립트를 사용하면 이러한 함수에 대해 정확한 타입 정보를 제공할 수 있어 코드의 안정성과 가독성을 높일 수 있다.

예) `function* map<A,B> (f: (a:A) ⇒ B, iterable : Iterable<A>) : IterableIterator<B>` 와 같이 **제네릭**을 활용하면 입력과 출력 타입을 명확히 지정한 이터레이션 함수를 만들 수 있다.

**참고**

for…of 문으로 순회할 수 있는 값은 오직 이터러블이다.

`전개 연산자`로 배열을 만들 수 있는 값도 오직 이터러블이다.

제너레이터로도 이터레이터를 만들수 있고, 제너레이터의 실행 결과는 IterableIterator이다.

**forEach타입**

```typescript
function forEach<A>(f: (a:A) => void, iterable: Iterable<A>) : void {
	for (const a of iterable) {
		f(a);
	}
}

const array = [1,2,3,4]
forEach(a => console.log(a.toFixed(2)), array);
```

**map과 타입**

```typescript
function* map<A, B>(f: (a:A) => B, iterable: Iterable<A>) : IterableIterator<B> {
	for (const a of iterable) {
		yield f(a);
	}
}

const array = ['1', '2', '3'];
const mapped = map(a => parseInt(a), array); 
```

**filter와 타입**

```typescript
function* filter<A>(f: (a:A) => boolean, iterable : Iterable<A>) : IterableIterator<A> {
	for (const a of iterable) {
		if (f(a)) {
			yield a;
		}
	}
}

const array = [1,2,3,4];
const filtered = filter(a => a % 2 === 0, array) // type : IterableIterator<number>

const array2 : number[] = [...filtered]; 
```

**reduce와 타입**

```typescript
function reduce<A, Acc> (
	f : (acc : Acc, a : A) => Acc, acc : Acc, iterable : Iterable<A>
) : Acc {
	for (const a of iterable) {
		acc = f(acc, a);
	}
	return acc;
}

const array = [1,2,3];
const sum = reduce((acc, a) => acc + a, array);
```

reduce 다시보기

### LISP와 메타프로그래밍

LISP 계열 언어는 코드와 데이터를 동일하게 취급하여 메타프로그래밍을 손쉽게 구현할 수 있다. LISP의 S-표현식은 리스트 형태의 구문으로 코드 자체를 데이터로 다뤄 동적으로 변경하고 생성할 수 있는 강력한 기능을 제공한다.

**LISP**

LISP는 프로그램이 언어의 계열로, 코드가 데이터고 데이터가 코드라는 개념이다.

- 클로저

  2007년에 개발된 함수형 프로그래밍 언어. LISP 계열에 속한다.

  - S-표현식

    리스트 형태의 구문 표현을 의미한다. → 코드와 데이터를 동일한 구조로 다룰 수 있다.

    예) (+ 1 2)

    첫 요소가 연산자, 나머지는 피연산자

예) 리스트를 평가하는 함수

```typescript
type Evaluatable<A, B> = [(...args: A[]) => B, ...A[]];

function evaluation<A, B>(expr : Evaluatable<A, B>) {
	const [fn, ...args] = expr; // 구조 분해할당
	return fn(...args); // 실제 함수로 평가하는 과정
}

const add = (a: number, b:number) => a+b;
const result : number = evaluation([add, 1, 2]);
```

LISP의 코드가 데이터라는 개념

**멀티패러다임 언어에서 클래스 + 함수형**

`개선 전`

```typescript
class FxIterable<A> {
	constructor(private iterable : Iterable<A>) {}

	toArray(): A[] {
		return [...this.iterable];
	}
}

const [first, second] = fx([1,2,3,4]).map(a => a+10).toArray();
console.log(first, second);
```

toArray : 이터러블 배열로 변환하여 반환

fx : FxIterable 인스턴스를 생성

`개선 후`

```typescript
class FxIterable<A> {
	constructor(private iterable : Iterable<A>) {}

	[Symbol.iterator]() {
		return this.iterable[Symbol.iterator]();
	}

}

const [first, second] = fx([1,2,3,4]).map(a => a+10).toArray();
console.log(first, second);
```

⇒ FxIterable을 이터레이션 프로토콜을 따르는 값으로 변경

**`LISP의 확장성 : 매크로와 메타프로그래밍`**

매크로 : LISP 계열 언어에서 매크로는 코드를 입력받아 코드를 반환하는 함수, 컴파일 타임에 작동

→ 코드가 실행되지 않은 구문상태일 때 원하는 형태로 재구성.

예) rejext 메서드를 FxIterable에 추가하기

```typescript
class FxIterable<A> {
	constructor(private iterable : Iterable<A> {})
	
	[Symbol.iterator]() {
		return this.iterable[Symbol.iterator]();
	}

	reject(f : (a:A) => boolean) : FxIterable<A> {
		return this.filter(a => !f(a));
	}
}

const isOdd = (a: number) => a % 2 === 1;
```

```typescript
// 사용예
const [first, second] = fx([1,2,3,4,5,6])
		.map(a => a+10)
		.reject(isOdd);
		
console.log(first, second);
```

구조분해할당 : const [first, second]

객체지향 메서드 체이닝 패턴 : fx().map().reject()

`함수형 고차함수와 LISP` : `map = (f : (a:A) ⇒ B, iterable : Iterable<A>) ⇒ Iterable<B>`

### 클래스 + 고차함수 + 이터러블

FxIterable 클래스 예) 클래스와 고차함수, 이터러블을 결합하면 높은 표현력을 가진 추상화를 구현할 수 있다. 예 : const [first, second] = fx([1,2,3,4]).map(a ⇒ a * 10);과 같이 사용자 정의 객체를 언어의 기능과 유연하게 결합하고 `체이닝 패턴`을 통해 가독성 높은 코드를 구현할 수 있다.

체인을 활용하면 이터러블을 받아 이터러블을 반환하는 어떤 함수이던지 동적으로 만들어 확장이 가능하다.

예) FxIterable 클래스에 chain 메서드 추가

### 현대 프로그래밍 언어에서 구현가능한 높은 수준의 추상화

**일급 함수의 도입 + 이터레이션 프로토콜**의 확산은 기존 클래스 기반 반복자 패턴을 넘어 현대 프로그래밍 언어에서 다양한 패러다임을 자연스럽게 결합하는 높은 수준의 추상화를 실현하고 있다. 주요 언어들은 일급 함수, 이터러블, 제네릭, 고차함수, 커링, 풍부한 타입 시스템 등 다양한 기능을 제공한다.

언어 스펙이나 컴파일러를 변경하지 않고도 클래스 등 객체지향 요소와 함수형 패러다임의 핵심 개념, 표준화된 이터레이션 방식 등을 유기적으로 결합할 수 있게 되었다.

→ 개발자는 특정 패러다임에 국한되지 않고 필요에 따라 원하는 추상화수준을 구현하며 다양한 상황에 대응할 수 있다. 탄탄한 개념적 기반은 여러 플랫폼과 문제 영역에서 폭넓게 응용가능한 실용적인 언어 확장과 패턴화를 가능하게 한다.

타입이론과 함수형 프로그래밍은 관련이 깊다.

복잡한 타입과 이론을 엮어서 설명 (두잇 - 타입스크립트)
