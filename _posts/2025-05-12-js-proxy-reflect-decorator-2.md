---
title: "Proxy, Reflect, Decorator (2)"
date: 2025-05-12 20:09:00 +0900
categories: [javascript, es6]
tags: [javascript, reflect]
---

프로그래밍에서 `reflection`이라는 것은 런타임에서 객체의 변수, 속성, 메서드를 다이나믹하게 조작하는 프로세스를 말한다.

## Reflect

Reflect는 Proxy의 생성을 간소화하는 내장 객체이다.  (객체의 기본 동작을 함수 형태로 제공하는 내장 객체)

내부 메서드(`[[Get]]`, `[[Set]]`)는 내부 사양에만 적용되고 직접 호출될 수 없다. Reflect 객체는 어느정도 객체 호출을 가능하게 한다.

Object.keys(), Object.getOwnPropertyDescriptor() 및 prototype과 같은 Object 메서드는 어찌보면 고전적인 리플렉션 기능이라고 할 수 있다. 조금 더 나은 편의를 위해 ES6에서 Reflect라는 새로운 전역 객체를 추가하였다.

Reflect 객체는 생성자가 존재하지 않아 new 인스턴스로 사용하는게 아닌, Math나 JSON 객체처럼 정적 메서드만 지원한다. Proxy의 모든 트랩 종류를 동일한 인터페이스로 지원한다.

### Reflect.get()

객체의 속성을 조회한다.

```javascript
const obj = { a: 1, b: 'zero', c: true };
const arr = [1, 'zero', true];

Reflect.get(obj, 'a'); // 1
Reflect.get(obj, 'd'); // undefined
Reflect.get(arr, 1); // zero
```

### Reflect.set

객체의 속성을 설정한다.

```javascript
const obj = { a: 1, b: 'zero', c: true };

Reflect.set(obj, 'd', ['arg1', 'arg2']);
console.log(obj.d); // [ 'arg1', 'arg2' ]
```

### Reflect.has

```javascript
const obj = { a: 1, b: 'zero', c: true };

Reflect.has(obj, 'b'); // true
```

### Reflect.apply

```javascript
function add(a, b) {
    return a + b;
}

Reflect.apply(add, null, [3, 5]); // 8
```

`Reflect.apply(function, thisArg, argument)` 

- 첫 번째 인자는 실행할 함수
- 두 번째 인자는 this
- 세 번째 인자는 함수에 넣을 인자이다.

### Reflect.constructor

```javascript
class Person {
    constructor(name, age) {
        this.name = name;
        this.age = age;
    }
}

const p1 = Reflect.construct(Person, ['tanon', 20]);
console.log(p1); // Person { name: 'tanon', age: 20 }

const p2 = new Person('tanon2', 40);
console.log(p2); // Person { name: 'tanon2', age: 40 }

```

### Reflect.ownKeys

`객체`의 속성명들을 배열로 반환한다.

단 상속받은 속성과, enumerable하지 않는 프로퍼티는 제외된다.

```javascript
const duck = {
    name: 'Maurice',
    color: 'white',
    greeting: function () {
        console.log(`Quaaaack! My name is ${this.name}`);
    },
};

Reflect.ownKeys(duck); // [ "name", "color", "greeting" ]
```

## Proxy + Reflect 조합하기

자바스크립트 Proxy 객체를 이용해 `[[Get]]` 트랩 핸들러를 설정했다. 만약 Target Object에서 `[[Get]]` 연산이 발생하면, Proxy Handler에서 정의한 get함수가 먼저 호출하게 된다. 따라서 get 트랩 함수에 정의한 내용에 따라 Target Object의 `[[Get]]` 연산과 별개로 동작이 가능하다.

예) set 트랩에서 Reflect 사용

```javascript
let numbers = [];

numbers = new Proxy(numbers, {
    set(target, prop, val) {
        if (typeof val == 'number') {
            // target[prop] = val;
            Reflect.set(target, prop, val);
            return true; // 성공 시 true를 꼭 반환해야 한다.
        } else {
            return false;
        }
    },
});

numbers.push(1); // success
numbers.push(2); // success
console.log(numbers.length); // 2

numbers.push('1'); // Error : 'set' on proxy
```

만약 Target Object의 `[[GET]]` 연산을 이어나가고 싶을 때 매개변수 `target[prop] = val`로 처리하였다. 하지만 일반적인 Proxy의 트랙 파라미터를 이용한 방법은 현재 일어나는 탐색의 주체가 무엇인지 알 수 없기에 사이드 이펙트 문제가 발생하게 된다. 그래서 Reflect 메소드로 조합해서 사용한다.

예) apply에서 Reflect로 변경

```javascript
// decorator 함수
function delay(f, ms) {
    return new Proxy(f, {
        apply(target, thisArgs, args) {
            setTimeout(() => {
                // target.apply(thisArgs, args);
                Reflect.apply(target, thisArgs, args);
            }, ms);
        },
    });
}

function sayHi(user) {
    console.log(`Hello, ${user}`);
}

console.log(sayHi.length); // 1

// 래퍼 함수
sayHi = delay(sayHi, 3000);

console.log(sayHi.length); // 1, 프록시는 get length 연산까지 타깃 객체에 전달해준다.
sayHi('Tanon');
```

Reflect에는 Proxy에서 트랩할 수 있는 모든 내부 메서드와 동일한 내장 메서드를 가지고 있다. 따라서 Reflect를 사용하면 원래 객체에 그대로 작업을 전달할 때 사이드 이펙트 없이 전달 가능하다는 장점이 있다.

## Proxy, Reflect 응용 예제

### 1. 값 할당 시 문자열 필터링 하기

만약 Person 객체의 이름을 지정하는데 있어 10자리 이상 긴 문자열이 올 경우 필터링하려 한다. Person 객체의 멤버를 수정하지 않고 프록시를 이용해 추가 로직을 생성할 수 있다.

```javascript
class Person {
    name;
    setName(name) {
        this.name = name;
    }
}

let me = new Person();

me.setName(123123); // Throw Error
me.setName('this_is_a_too_long_user_name_that_has_to_be_prevented'); // Throw Error

console.log(me);
```

setName()이라는 메서드 실행 로직을 가로채야 하기에, apply 트랩 핸들러를 이용하여 아래와 같이 구현할 수 있다.

```javascript
class Person {
    name;
    setName(name) {
        this.name = name;
    }
}

let me = new Person();
me.setName = new Proxy(me.setName, {
    apply(target, thisArgs, args) {
        const value = args[0];

        if (typeof value !== 'string') {
            throw new TypeError('Name should be String');
        } else if (value.length > 10) {
            throw new RangeError('Length should be under 10');
        }

        return Reflect.apply(target, thisArgs, args);
    },
});

try {
    me.setName(123123); // Error
} catch (error) {
    console.error(error.message);
}
try {
    me.setName('this_is_a_too_long_user_name_that_has_to_be_prevented'); // // Error
} catch (error) {
    console.error(error.message);
}
me.setName('tanon');
console.log(me);
```

### 2. 존재하지 않은 프로퍼티는 에러 발생 시키기

자바스크립트는 객체에 등록되지 않은 프로퍼티라도 접근하지 않으면 그냥 undefined를 반환한다. 

요구사항 : 이를 타이트하게 관리하기 위해 존재하지 않은 프로퍼티를 접근하는 경우 에러를 던지게 하고 싶다.

```javascript
let user = {
    name: 'tanon',
};

user = new Proxy(user, {
    get(target, prop, receiver) {
        if (prop in target) {
            return Reflect.get(target, prop, receiver);
        } else {
            throw new Error(`ReferenceError: Property doesn't exist "${prop}"`);
        }
    },
});

console.log(user.name); // tanon
console.log(user.age); // Error: ReferenceError: Property doesn't exist "age"
```

### 3. 음수 인덱스 구현하기

자바스크립트의 배열 인덱스는 음수 인덱스를 지원하지 않는다. 이를 프록시를 이용해 구현할 수 있다.

```javascript
let array = ['홍길동', '임꺽정', '박혁거세'];

// 마지막 요소 : 박혁거세
console.log('array[-1]: ', array[-1]); // undefined

// 뒤에서 두 번째 요소 : 임꺽정
console.log('array[-2]: ', array[-2]); // undefined

// 뒤에서 세 번째 요소 : 홍길동
console.log('array[-3]: ', array[-3]); // undefined
```

Proxy의 get 트랩에 prop 매개변수에 담기는 인덱스가 0보다 작은 경우 계산하여 변환 후 실행하면 된다.

```javascript
let array = ['홍길동', '임꺽정', '박혁거세'];

array = new Proxy(array, {
    get(target, prop, receiver) {
        let propNum = +prop;

        if (propNum < 0) {
            propNum += target.length;
        }
        return Reflect.get(target, propNum, receiver); // propNum을 리턴
    },
});

// 마지막 요소 : 박혁거세
console.log('array[-1]: ', array[-1]);

// 뒤에서 두 번째 요소 : 임꺽정
console.log('array[-2]: ', array[-2]);

// 뒤에서 세 번째 요소 : 홍길동
console.log('array[-3]: ', array[-3]);
```
