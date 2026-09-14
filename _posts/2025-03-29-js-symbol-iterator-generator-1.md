---
title: "Symbol/Iterator/Generator (1)"
date: 2025-03-29 20:00:00 +0900
categories: [javascript, es6]
tags: [javascript, symbol]
---

### 원시값

JS에서 원시값은 객체가 아니면서 메서드도 가지지 않는 데이터이다. 원시값에는 7종류가 존재한다. string, number, undefined, null, bigint, boolean, **symbol**. Symbol은 ES6에서 새롭게 추가한 원시 타입니다.

symbol은 어플리케이션 레벨에서는 잘 쓰이지 않아도, 그 특유의 고유값을 만드는 로직으로 JS내부 구현에서 매우 중요하게 쓰이고 있었다.

### 1. 심볼 소개

생성자가 Symbol 원시값을 반환하는 내장 객체로 ES6에 도입되었다. Symbol은 객체에 속성을 추가할 때 고유한 키를 부여하여 **다른 코드(라이브러리)와 충돌하지 않도록** 할 때 많이 쓰이며, 프로그램 전체에서 유일한 값임을 보장한다. 이렇게 추가한 속성은 일반적으로 접근할 수 없으므로, 약한 형태의 캡슐화/정보 은닉을 제공한다.

모든 Symbol() 호출은 각각 고유한 심볼을 반환하는 것이 보장된다. 혼동을 막기 위해 Symbol 생성자 함수에 new를 쓰는 것은 지원하지 않는다.

예) Symbol 생성

```javascript
let id1 = Symbol();
let id2 = Symbol();
console.log(id1 === id2);

// TypeError: Symbol is not a constructor
let newSymbol = new Symbol(); 
```

**1.1 심볼의 생성**

- Symbol(’some’) : 고유한 심볼을 리턴한다
- Symbol.for(’some’) : 심볼 레지스트리에서 lookup을 하여 description이 같으면 매번 같은 심볼을 리턴한다(런타임 범위).
- Well-Known Symbol : Symbol.iterator과 같이 표준에 정의된 심볼을 사용한다. 잘 알려진 심볼은 특별한 용도를 위해 미리 만들어 둔 것이다.

### 2. 심볼의 설명인자

심볼을 생성할 때 생성자 `Symbol()`의 인수에 문자열을 전달함으로써 설명을 붙일 수 있다.

**심볼은 unique하기 때문에 description이 같아도 충돌하지 않는다.**

```javascript
let id1 = Symbol('id');
let id2 = Symbol('id');
console.log(id1); // Symbol(id)
console.log(typeof id1); // symbol

id1 === id2 // false
```

심볼은 문자형으로 자동 형변환될 수 없다.

```javascript
let id1 = Symbol('id');

alert(id1); // Uncaught TypeError: Failed to execute 'alert' on 'Window': Cannot convert a Symbol value to a string
alert(id1.toString()); // Symbol(id)
```

설명이 동일한 심볼을 여러개 만들어도 **심볼 각각은 유일**하기에 생성자 함수에 인수로 전달한 설명은 우리가 식별할 수 있는 이름표 역할을 할 뿐이다.

### 3. 전역 심볼 레지스트리

심볼은 생성자 인수로 전달한 문자열과 관계없니 Symbol() 생성자로 생성되었다면 모두가 고유하고 유일한 값임을 보장한다.

하지만 이는 여러 스트립트에서 공통으로 사용할 수 없고, 처음 생성한 곳 이외에 다른 곳에서는 접근하기 쉽지 않은 단점이 있다. 어떤 내부에서 심볼 키를 생성했다면 그 키를 어떻게 가져올까? 

⇒ 전역 심볼 레지스트리를 사용하면 이름이 같은 심볼이 같은 객체를 가리키도록 할 수 있다.

**3.1 Symbol.for(key)**

**`Symbol.for(key)`** 는 인수로 주어진 key를 사용해서 런타임 범위의 심볼 레지스트리에서 해당 키를 가진 심볼을 찾고, 존재할 경우 이를 반환한다. 만약 그런 심볼이 없는 경우에는 해당 키를 사용해 전역 심볼 레지스트리에서 새로운 심볼을 만든 후 리턴한다.

즉 `key`를 사용해서 함수를 호출한 경우 런타임 범위에서는 항상 같은 심볼을 반환한다.

```javascript
let id1 = Symbol.for('id'); // 전역 심볼 레지스트리에 id 심볼 등록
let id2 = Symbol.for('id'); // 등록된 심볼을 반환
console.log(id1 === id2);
```

**3.2 Symbol.keyFor(sym)**

Symbol.for(key)를 통해 만든 심볼은 Symbol.keyFor(sym)을 사용해 심볼을 생성할 때 사용했던 key를 얻을 수 있다.

```javascript
let id1 = Symbol.for('pizza'); // 전역 심볼 레지스트리에 id 심볼 등록
let id2 = Symbol.for('pasta');
console.log(Symbol.keyFor(id1)); // pizza
console.log(Symbol.keyFor(id2)); // pasta

let id1 = Symbol('pizza'); 
console.log(Symbol.keyFor(id1)); // undefined
console.log(id1.description); // pizza
```

이 함수는 전역 심볼 레지스트리를 찾아 인수로 받은 심볼의 이름을 얻어낸다. 전역 심볼 레지스트리에 등록되지 않은 심볼이라면 undefined를 반환한다. 만약 전역 심볼이 아닌 심볼의 생성 시 이름으로 전달한 문자열 인수를 얻고 싶다면 description 프로퍼티를 사용하면 된다.

### 4. 심볼형이 만들어진 목적

심볼의 원래 목적은 private property를 만드는 것이었다. 클래스 대신 객체를 프로토타입으로 사용하는 JS 특성 상 객체 내부에서만 접근할 수 있는 어떤 프로퍼티를 만드는게 목적이었을 것이다. 

그러나 Reflect.ownKeys등 객체 내부의 심볼 키를 알 수 있는 방법들이 생기면서 심볼은 원래 목적과는 멀어졌다. 그래도 키값의 중복이 `원천적으로 일어날 수 없는 고유값`이라는 이 심볼의 특성은 유용해서, 이름 충돌을 걱정하지 않고 어떤 객체의 특성을 만들 때 쓰인다.

### 5. 심볼의 사용

아래는 심볼을 어디에 사용할 수 있을지, 어플리케이션 레벨의 사용처이다.

**5.1 상수의 정의에 사용**

예) 값의 중복

```javascript
const Direction=Object.freeze({
  UP:'up',
  DOWN:'down',
  LEFT:'left',
  RIGHT:'right',
})
```

이때 Direction.UP은 문자열 ‘up’과 같기에 상수의 실제 값이 중복될 가능성이 있다. 예를 들어 Direction.UP===Command.UP이 같아지는 상황이 생길 수 있다.

이럴 때 값의 중복을 막기 위해 심볼을 사용하면 더 견고한 코드를 작성할 수 있다. 

```javascript
const Direction = Object.freeze({
    UP: Symbol('up'),
    DOWN: Symbol('down'),
    LEFT: Symbol('left'),
    RIGHT: Symbol('right'),
});
```

**참고 : React의 Symbol사용**

![](/assets/img/2025-03-29/js-symbol-iterator-generator-1-1.png)

### **6. 심볼형으로 숨김 프로퍼티 만들기**

심볼은 문자열과 함께 객체의 프로퍼티 키로 사용할 수 있다. 심볼 키는 `for..in`이나, `Object.getOwnProperties()`, `Object.Keys()`와 같은 일반적인 객체 속성 조회 방법에서는 무시된다.

예) 일반적인 객체 속성에서의 Symbol 값 무시

```javascript
const mySymbol = Symbol('mySymbol');

const obj = {
    nickname: 'tanon',
    age: 20,
    [mySymbol]: 'hello',
};

/* nickname tanon
age 18 */
for (let key in obj) {
    console.log(key, obj[key]); 
}

console.log(Object.keys(obj)); // [ 'nickname', 'age' ]
```

심볼의 이런 특성을 활용하면 객체 내부에 일반적으로 조회할 수 없는 프로퍼티를 만들 수 있다. 이를 이용해서 객체에 숨김 프로퍼티를 만들 수 있다.

**6.1 객체의 숨겨진 속성 만들기**

이런 숨김 프로퍼티는 먼저 객체의 메타 데이터를 기록하는데 쓰일 수 있다. 외부 코드에서 쉽게 접근할 수도, 값을 덮어쓸수도 없는 프로퍼티를 만드는 것이다.

예) 객체의 메타 데이터를 기록

```javascript
const userInternalKey = Symbol('useInternalKey');
let user = {
    nickname: 'tanon',
    [userInternalKey]: 12341234,
};
console.log(user); // { nickname: 'tanon', [Symbol(useInternalKey)]: 12341234 }
```

이를 클래스나 생성자 함수 단위에서 이용할 수 있다. 이러면 이 생성자나 클래스의 모든 인스턴스에 숨김 프로퍼티가 추가된다.

```javascript
const mySymbol = Symbol('mySymbol');

class Myclass {
    constructor() {
        this[mySymbol] = 'foo';
        this.prop = 'bar';
    }
    getMyValue() {
        return this[mySymbol];
    }
}

const myClass = new Myclass();
console.log(myClass.getMyValue()); // foo
console.log(Object.getOwnPropertyNames(myClass)); // ['prop']
```

Object.getOwnPropertyNames()은 프로퍼티 이름을 출력하기에 만약 myClass인스턴스가 심볼 속성이 없는 일반 객체라면 모든 속성을 반환해야 한다. ‘prop’만 출력된다.

**6.2 외부 객체에 속성 추가하기**

이를 실질적으로 활용하는 방법, 즉 객체에서 쉽게 조회할 수 없고 이름 충돌도 걱정할 필요가 없는 프로퍼티를 만들어야 하는 상황은 무엇일까? 외부 라이브러리 코드에서 가져온 객체에 나만의 속성을 추가해야 할 때가 예시이다. 

```javascript
const user = {
    name: 'tanon',
    nickname: 'tanon',
    position: 'developer',
    age: 20,
};
```

여기에 추가적으로 isWorking속성을 붙여주고 싶다고 하자.

`user.isWorking = true;` 로 바로 속성을 추가할 수 있지만, 이렇게 할 경우 몇 가지 문제가 발생할 수 있다.

가장 대표적인 분제로는 외부 라이브러리 코드에서 해당 객체를 `for..in`이나 `Object.keys()`와 같이 객체 속성들을 전부 조회하는 함수에 사용하면 내가 추가한 속성 때문에 문제가 발생할 수 있다. 추후에 라이브러리가 user객체에 isWorking 속성을 추가하는 바람에 내 속성이 추가된 속성가 충돌을 일으킬 수 있고, 표준위원회가 isWorking 속성을 추가하도록 결정할 수도 있다.

이런 문제들은 심볼을 외부 객체의 key로 사용하면 된다. 심볼은 프로그램 전체에서 유일한 값으로 보장된 값이고, for..in과 같은 객체 속성 조회 코드에서도 무시된다. 

```javascript
const user = {
    name: 'tanon',
    nickname: 'tanon',
    position: 'developer',
    age: 20,
};

const isWorking = Symbol('isWorking');
user[isWorking] = true;

console.log(user);
console.log(Object.getOwnPropertyNames(user)); // [ 'name', 'nickname', 'position', 'age' ]

```

위의 예시는 외부 스크립트는 심볼 키를 가진 프로퍼티에 쉽게 접근하지 못한다는 점을 이용한다. 

**6.3 다른 방법과의 비교**

이런 숨김 프로퍼티는 다른 코드에서 관여할 수 없는 프로퍼티를 만들거나, 각각을 식별할 수 있는 값을 만들어줘야 할 때 사용할 수 있다. 

예) 

```javascript
let id1 = Symbol('id');
let id2 = Symbol('id');

let cafe1 = {
    name: 'Starbucks',
    [id1]: 1, // 대괄호를 이용해서 객체 리터럴에 바로 심볼 키를 생성
};

let cafe2 = {
    name: 'Mega',
    [id2]: 2,
};
```

예) uuid 라이브러리를 활용한 동작

```javascript
const id = uuidv4();

function addRandomProperty(obj) {
    obj[id] = 1;
}

let user = {
    name: 'tanon',
};

addRandomProperty(user);

if (user[id]) {
    console.log('id exists');
}
```

uuid 라이브러리를 사용하면 Symbol을 사용하지 않더라도 위의 예시처럼 사용이 가능하다. 거의 똑같이 동작한다. 이런 방식, 즉 랜덤 문자열 생성 방식의 문제는 랜덤 문자열 키를 사용할 경우 외부에서 접근하기가 너무 쉬워진다. 심볼도 완전한 private는 아니지만 `for..in`을 통한 접근이나 `JSON.stringify,` `Object.keys()`와 같은 객체 속성 접근 방식으로부터는 안전하다

예) Symbol과 일반 문자열 생성 방식의 비교

```javascript
import { v4 as uuidv4 } from 'uuid';

const id = uuidv4();
const symbolId = Symbol('id');

function addRandomProperty(obj) {
    obj[id] = 1;
}

function addSymbolProperty(obj) {
    obj[symbolId] = 1;
}

let user1 = {
    name: 'tanon',
};

let user2 = {
    name: 'kimbab',
};

addRandomProperty(user);
addSymbolProperty(user);

console.log(JSON.stringify(user1)); // {'name' : 'tanon', "8f2aeb41-eb10-43f0-1234-fd994926b63e" : 1}
console.log(JSON.stringify(user2)); // {'name' : 'kimbab'}
```

물론 랜덤 문자열 속성을 사용자 접근으로부터 숨기는게 가능하다. Object.`defineProperty`를 사용하면 된다. 하지만 심볼을 쓸 수 있는 환경이라면 충돌이 일어날 수 있는 랜덤 문자열 생성과 복잡한 객체 사용보다는 심볼을 사용하는게 더 간단하고 안전하다.

[Object.defineProperty() - JavaScript \| MDN](https://developer.mozilla.org/ko/docs/Web/JavaScript/Reference/Global_Objects/Object/defineProperty)

```javascript
// 기본 값 명시하기
Object.defineProperty(obj, "key", {
  enumerable: false,
  configurable: false,
  writable: false,
  value: "static",
})
```

### 7. Well-Known Symbol (잘 알려진 심볼)

요약 : 자바스크립트의 기본 동작을 확장하고, 커스텀하는데 사용되는 **예약된 Symbol** 값들

심볼이 등장하기 전에 JS는 몇몇 내장 작업에 해당 객체의 함수 속성을 사용했다. `JSON.stringify` 함수는 지금도 객체의 `toJSON()`메서드를 사용한다. `toString()`메서드도 객체의 일반 속성으로 정의되어 있다.

하지만 이런 객체 내장 함수 속성이 늘어날수록 이름 충돌로 인해 하위 호환성이 깨질 가능성이 높아지고, 사용자가 객체 속성을 만들 때 고려해야 할 점이 늘어난다.

이런 내장 내장 함수를 심볼 키에 대응시킴으로써 해당 문제를 해결할 수 있게 되었다. 그리고 이때 쓰이는 심볼 키를 well-known symbol이라고 한다.

**Well-Known Symbol 소개**

Symbol 생성자 함수의 정적 속성은 모두 그 자체로 심볼이다. 이 심볼들은 Well-Kwown Symbol이라 불리며, JS내장 작업에서 일종의 프로토콜로 동작한다.

well-known symbol은 보통 이름 앞에  `@@toPrimitive`와 같은 방식처럼  `@@`를 붙여 구분한다. 

ECMA의 JS 명세를 읽다보면 ToPrimitive라는 내장 작업을 보게 된다. 이 작업은 객체를 원시형으로 변환할 때 사용하게 되는데, 이떄 가장 우선적으로 사용하는 메서드가 바로 `@@toPrimitive` 이다. 메서드가 객체에 없으면 `toString`과 `valueOf`를 사용한다.

다른 잘 알려진 심볼로는 `@@iterator`, `@@toStringTag` 등이 있다. 이런 식으로 JS는 **내장 작업**을 위해 Well-Known Symbol을 우선적으로 사용하는데, JS의 내부 구현에 많이 녹아져 있는 개념이다.

추가로 Well-Known Symbol은 프로그램의 수명 내내 고유하게 유지되기 때문에 **GC 여부와 같은 문제를 고려하지 않는다**. 늘 존재한다.

### 8. Well-Known Symbol 예시

**8.1 Symbol.unscopables**

해당 심볼은 특정 객체의 속성을 with 바인딩에서 제외하는데, with에서 발생하는 충돌을 해결하기 위해 만들어졌다.

```javascript
const human = {
    name: 'tanon',
    age: 20,
    [Symbol.unscopables]: {
        age: true,
    },
};

with (human) {
    console.log(age); // ReferenceError: age is not defined
}
```

다음과 같이 Symbol.unscopables를 이용해서 객체의 일반 속성을 with 바인딩에서 가릴 수 있다.

객체에 기존에 정의되어있는 프로퍼티 혹은 메서드와 with 바인딩의 이름이 충돌할 때 기존 프로퍼티를 가리기 위해 `@@unscopables`를 사용할 수 있다.

**8.2 Symbol.toPrimitive**

객체를 원시형으로 형 변환 시 사용되는 메서드이다. 

hint에 따라 다른 원시형으로 변환하도록 할 수 있고, 반환형이 원시형이라면 형에 대한 제약이 없다.

예)

```javascript
const user = {
    name: 'tanon',
    age: 20,
    [Symbol.toPrimitive](hint) {
        return this.age;
    },
};

console.log(String(user)); // 30
console.log(+user); // 30
```

빌트인 Date 객체 등이 커스텀 toPrimitive를 가지고 있다.

**8.3 Symbol.toStringTag**

ECMA5에는 모든 객체에 객체의 분류를 나타내는 `[[Class]]` 내부 프로퍼티가 존재했었다. 그리고 이를 toString() 메서드에 사용했다.

ES6부터는 `[[Class]]` 프로퍼티가 사라졌고, 호환성을 위해 `@@toStringTag` 가 정의되었다. 따라서 `@@toStringTag`를 오버로딩하면 `toString()`메서드를 호출할 때 사용된다. 

```javascript
class MyClass {
    get [Symbol.toStringTag]() {
        return 'my class';
    }
}

const myClass = new MyClass();
console.log(myClass.toString()); // [object my class]
```

**8.4 Symbol.iterator**

`for..of` 루프는 `obj[Symbol.iterator]()` 를 호출하면서 시작된다. 따라서 Symbol.iterator 메서드를 이용하면 반복도 오버로딩할 수 있다.

예) 객체를 반복가능하게 하는 Well-known Symbol

```javascript
const obj = {
  *[Symbol.iterator]() {
    yield 1;
    yield 2;
    yield 3;
  }
};

console.log([...obj]); // [1, 2, 3]
```

**8.4.1 iterator protocol**

이때 반복을 담당하는 함수는 iterator 프로토콜로 정의되는데, 객체가 next 메서드를 가지고 있고, 다음 규칙에 따라 구현되었으면 iterator 프로토콜을 따른다고 할 수 있다.

`done`(boolean) : Iterator(반복자)가 마지막 반복 작업을 마쳤을 경우 true. 만약 iterator(반복자)에 return 값이 있다면 value값으로 지정된다. 반복자의 작업이 남아있는 경우 false이다.

`value` : iterator로부터 반환되는 모든 자바스크립트 값. done이 true면 생략될 수 있다.

String의 기본 반복자를 통해 볼 수 있다.

예)

```javascript
let word = 'tanon';
let it = word[Symbol.iterator]();

console.log(it.next()); // { value: 't', done: false }
console.log(it.next()); // { value: 'a', done: false }
console.log(it.next()); // { value: 'n', done: false }
console.log(it.next()); // { value: 'o', done: false }
console.log(it.next()); // { value: 'n', done: false }
console.log(it.next()); // { value: 'undefined', done: true }
```

이를 이용해 Symbol.iterator 오버로딩을 하면 반복을 다르게 동작시킬 수 있다. 다음 코드를 보면 user객체의 Symbol.iterator를 오버로딩한 예이다.

예) 오버로딩 예시

```javascript
let user = {
    name: 'tanon',

    [Symbol.iterator]: function () {
        return {
            next: function () {
                if (this._first) {
                    this._first = false;
                    return { value: 'work', done: false };
                } else {
                    return { done: true };
                }
            },
            _first: true,
        };
    },
};

let it = user[Symbol.iterator]();
console.log(it.next()); // { value: 'work', done: false }
console.log(it.next()); // {done: true}

for (let i of user) {
    console.log(i); // work
}
```

user 객체를 `for..of`로 순회할 때 name이 아닌 전혀 다른 문자열이 나오게 출력되었다.

**8.4.2 제너레이터 생성하기**

제너레이터는 복잡한 데이터 구조를 순차 접근할 수 있는 배열처럼 쉽게 다룰 수 있게 해준다. 제너레이터가 데이터를 한 번에 하나씩 반환하도록 해주기 때문이다.

또 제너레이터는 함수가 호출되었을 때 끝까지 실행하지 않고 빠져나갔다가 다시 돌아올 수 있는 함수이기도 하다.여러개의 값을 필요한 만큼 반환할 수 있다는 것이다. ↔ 배열 : 원하지 않아도 끝까지 할 수 있다.

제너레이터는 function 키워드 뒤에 `*` 를 붙여 만들 수 있으며, yield 키워드를 통해 정보를 반환한다. next()메서드도 쓸 수 있는데, 데이터를 하나씩 가져오는 역할을 한다.

```javascript
function* getStudyMember() {
    yield 'AAA';
    yield 'BBB';
    yield 'CCC';
    yield 'DDD';
    yield 'EEE';
    yield 'FFF';
}

const member = getStudyMember();
console.log(member.next()); // { value: 'AAA', done: false }
console.log(member.next()); // { value: 'BBB', done: false }
console.log(member.next()); // { value: 'CCC', done: false }
console.log(member.next()); // { value: 'DDD', done: false }

for (const m of member) {
    console.log(m); // EEE, FFF
}
```

```javascript
function* getStudyMember() {
    yield 'AAA';
    yield 'BBB';
    yield 'CCC';
    yield 'DDD';
    yield 'EEE';
    yield 'FFF';
}

const member = getStudyMember();

for (const m of member) {
    console.log(m); // AAA, BBB, CCC, DDD, EEE, FFF
}

```

제너레이터를 이용하면 복잡한 데이터 구조를 다룰 때 **순차접근하는 순서를 정해 줄 수 있다**. 복잡하게 연관된 데이터를 배열처럼 순차 반복할 수 있게 되면 훨씬 간단하게 다룰 수 있다.

그리고 제너레이터 함수를 Symbol.iterator에 할당해주면 제너레이터를 통해 복잡한 객체를 마치 배열과 같은 반복 가능한 객체로써 다룰 수 있게 된다.

예)

```javascript
function Study() {
    this.members = {
        name: 'AAA',
        friend: {
            name: 'BBB',
            friend: {
                name: 'CCC',
                friend: {
                    name: 'DDD',
                },
            },
        },
    };
    this[Symbol.iterator] = function* () {
        let node = this.members;
        while (node) {
            yield node.name;
            node = node.friend;
        }
    };
}

const myStudy = new Study();
for (let m of myStudy) {
    console.log(m);
}

```

또한 이렇게 제너레이터를 만들면 내가 만든 객체의 복잡도를 숨길 수 있다. 그럼 다른 개발자가 내가 만든 데이터 구조를 사용할 수 있다. 

---

**8.5 Well-Known Symbol의 특성**

**잘 알려진 심볼들은 엔진에 의해 모든 영역에서 공유된다**. 앞서 살펴본 전역 심볼 레지스트리의 심볼과 같이 작동하는 것이다. 하지만 모든 영역에서 공유된다는 특성만 같을 뿐 잘 알려진 **심볼들을 전역 심볼 레지스트리에서 찾을 수는 없다.**
