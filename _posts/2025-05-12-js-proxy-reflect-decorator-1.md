---
title: "Proxy, Reflect, Decorator (1)"
date: 2025-05-12 19:33:00 +0900
categories: [javascript, es6]
tags: [javascript, proxy]
---

### 메타 프로그래밍

Proxy를 알아보기 전 메타 프로그래밍의 개념을 알아보자. 메타 프로그래밍이란 하나의 **`컴퓨터 프로그램이 다른 프로그램을 데이터로 취급`**하는 것을 말한다. 즉 어떤 프로그램이 다른 프로그램을 읽고, 분석하거나 변환하도록 설계된 것이다. 

메타 프로그래밍에 이용되는 언어를 메타 언어라고 하고, 조작 대상이 되는 언어를 대상 언어라고 한다. (같을수도, 다를수도 있다.)

**메타 언어와 대상 언어가 다른 경우**

예) JSP에서 HTML을 만들어내는 코드

```java
out.println("<html><body>");

out.println("<h1>Hello World</h1>");
out.println("The current time is : " + new java.util.Date());
out.println("</body></html>");
```

메타 언어는 Java이고, 대상 언어는 HTML이다.

**메타 언어와 대상 언어가 같은 경우**

예) eval()

```javascript
> eval('1 + 1')
2
```

메타 언어와 대상 언어는 둘다 JavaScript이다. `eval()` 은 런타임 중에 매개변수를 평가한다. 이렇게 한 프로그래밍 언어가 자기 자신의 메타 언어가 되는 것을 반영(`Reflection`)이라고 하며, 이는 런타임 시점에서 자신의 구조와 행위를 관리하고 수정하는 것을 말한다.

### **반사형 프로그래밍**

Proxy와 Reflect는 반영을 구현한 것이다. 그리고 여기에는 또 세가지 종류가 있다.

- Type introspection(유형 검사)
  - 런타임에서 프로그램이 자신의 구조에 접근하여 타입이나 속성을 알아내는 능력을 말한다. 
  - 예 : Object.keys()
- Self-modification(구조를 스스로 변경)
  - 구조를 스스로 변경할 수 있는 의미이다. 
  - 예) 어떤 속성에 접근하기 위해 `[]` 표기법을 사용하고, `delete` 연산자로 제거한다.
- Intercession (개입, 알선)
  - 어떤 것을 대신하여 개입하는 행위를 뜻하고, 언어가 수행되는 일부 의미를 재정의하는 것을 의미한다.
  - 예) Proxy _ ES2015에 만들어짐

Proxy는 프로그래밍 언어의 어떤 기능, 개념을 지원하기 위해 만들어진 문법이다.

### Proxy

`Proxy`는 특정 객체를 감싸 프로퍼티 읽기, 쓰기와 같은 객체에 가해지는 작업을 중간에서 가로채는 객체이다. 가로챈 작업은 Proxy에서 처리되기도 하고, 원래 객체가 처리하도록 그대로 전달되기도 한다.

예) 문법

```javascript
let proxy = new Proxy(target, handler);
```

- target : 감싸게 될 객체로 함수를 포함한 모든 객체가 가능하다.
- handler : 동작을 가로채는 메서드인 `트랩` 이 담긴 객체로 여기서 프록시를 설정한다.

proxy에 작업이 가해지고, handler에 작업과 대응되는 `트랩`이 있으면, 트랩이 실행되어 프록시가 이 작업을 처리할 기회를 얻게 된다. 트랩이 없으면 target에 작업이 직접 수행된다.

### 트랩이 없는 Proxy

handler가 비어있으면 Proxy에 가해지는 작업은 target에 바로 전달이 된다.

예) trap이 없는 Proxy

```javascript
let target = {}
let proxy = new Proxy(target, {}); // 빈 핸들러

proxy.test = 5; 프록시에 값을 할당한다.
alert(target.test); // 5

alert(target.test); // 5
for (let key in proxy) alert(key) // test
```

해당 예시는 Proxy에 트랩이 없기에 proxy에 가해지는 모든 작업은 target에 전달된다.

1. `proxy.test = 5`로 값을 할당하면 target에 새로운 값이 설정된다.
2. proxy.test를 이용해서 값을 읽으면 target에서 값을 읽어온다.
3. proxy를 대상으로 반복 작업이 가능하고, target에 저장된 값이 반환된다.

트랩이 없으면 target을 둘러싸는 투명한 래퍼가 된다.

![](/assets/img/2025-05-12/js-proxy-reflect-decorator-1-1.png)

### 트랩을 사용해 가로챌 수 있는 작업

### get 트랩으로 프로퍼티 기본값 설정하기

가장 흔하게 볼 수 있는 트랩은 프로퍼티를 읽거나 쓸 때 사용되는 트랩이다.

프로퍼티 읽기를 가로채려면 handler에 `get(target, property, receiver)`메서드가 있어야 한다. get 메서드는 프로퍼티를 읽으려고 할 때 작동한다. 

- target : 동작을 전달할 객체로, new Proxy의 첫번째 인자이다.
- property : 프로퍼티 이름
- receiver : 타깃 프로퍼티가 getter라면 receiver는 getter가 호출될 때 this이다. 보통은 proxy 객체 자신이 this가 되며, 프록시 객체를 상속받은 객체가 있다면 해당 객체가 this가 되기도 한다. 

예) 존재하는 요소가 있다면 해당 값 반환, 없으면 기본값 반환

```javascript
let numbers = [0, 1, 2];

numbers = new Proxy(numbers, {
	get(target, prop) {
		if (prop in target) {
			return target[prop];
		} else {
			return 0;
		}
	}
});
```

예시의 코드는 존재하지 않는 요소를 읽으려고 할 때 기본값 0을 반환해주는 배열이다. 존재하지 않는 요소를 읽으려고 하면 배열은 `undefined`를 반환하는데, 예시에선 객체(배열)을 감싸면서 존재하지 않는 요소를 읽으려고 할 때 0을 반환한다.

예) 값이 없는 경우 undefined를 반환하는 proxy 미적용 예시

```javascript
let dictionary = {
	'Hello' : '안녕하세요',
	'Bye' : '안녕히 가세요',
}

alert(dictionary['Hello']); // 안녕하세요
alert(dictionary['World']); // undefined
```

예) 값이 없는 경우 그대로 반환하는 proxy 적용 예시

```javascript
let dictionary = {
	'Hello' : '안녕하세요',
	'Bye' : '안녕히 가세요',
}

dictionary = new Proxy(dictionary, {
	// 프로퍼티 읽기 가로채기
	get(target, phrase) {
		if (phrase in target) {
			return target[phrase];
		} else {
			return phrase;
		}
	}
});

alert(dictionary['Hello']); // 안녕하세요
alert(dictionary['World']); // World

```

`dictionary = new Proxy(dictionary, {});` 타깃 객체의 위치와 상관없이 프록시 객체는 타깃 객체를 덮어써야만 한다. 객체를 프록시로 감싼 이후엔 절대로 타깃 객체를 참조하는 코드는 없어야 한다.

### set 트랩으로 프로퍼티 값 검증하기

가정 : 숫자만 저장할 수 있는 배열을 생성한다.

조건 : 숫자형이 아닌 값을 추가하려고 하면 에러가 발생되도록 해야 한다.

해당 가정과 조건이 주어졌을 때 이를 가로채는 set트랩을 이용해 구현을 해보자. set메서드의 인수는 아래와 같은 역할을 한다.

`set(target, property, value, receiver);` 

- target : 동작을 전달할 객체
- property : 프로퍼티 이름
- value : 프로퍼티 값
- receiver : get 트랩과 유사하게 동작하는 객체, setter 프로퍼티에만 관여한다.

예) set 트랩을 활용한 input의 숫자형 검증

```javascript
let numbers = [];

numbers = new Proxy(numbers, {
	set(target, prop, val) {
		if (typeof val == 'number') {
			target[prop] = val;
			return true; // 성공 시 true를 꼭 반환해야 한다.
		} else {
			return false;
		}
	}
});

numbers.push(1); // success
numbers.push(2); // success
console.log(numbers.length) // 2

numbers.push("1"); // Error : 'set' on proxy
```

배열 관련 기능들을 여전히 사용할 수 있다. push를 사용해 배열에 새로운 요소를 추가하고, length 프로퍼티는 이를 잘 반영하고 있다. 프록시를 사용해도 기존에 있던 기능은 절대로 손상되지 않는다.

`push`, `unshift`와 같이 배열에 값을 추가하는 메서드들은 내부에서 `[[Set]]`을 사용하고 있기 때문에 메서드 오버라이드를 하지 않아도 프록시가 동작을 가로채고 값을 검증해준다.

set 트랩을 사용할 땐 성공 시 true는 꼭 반환해야 한다. true를 반환하지 않았거나, falsy 값을 반환하게 되면 TypeError가 발생한다.

### ownKeys와 getOwnPropteryDiescriptor로 반복작업 하기

`Object.keys`, `for..in` 반복문을 비롯한 프로퍼티 순환 관련 메서드 대다수는 내부 메서드 `[[OwnPropertyKeys]]` 를 사용해 프로퍼티 목록을 얻는다. 

아래는 메서드의 세부 동작 차이이다. 

- Object.getOwnPropertyNames(obj) : 심볼형이 아닌 키만 반환한다.
- Object.getOwnPropertySymbols(obj) : 심볼형 키만 반환한다.
- Object.keys/values() : enumerable 플래그가 true이면서 심볼형이 아닌 키나 심볼형이 아닌 키에 해당하는 값 전체를 반환한다.
- for..in : enumerable 플래그가 true인 심볼형이 아닌 키, 프로포타입 키를 순회한다.

메서드마다 차이가 있지만, `[[OwnPropertyKeys]]` 를 통해 프로퍼티 목록을 얻는다는 점은 동일하다.

예) _로 시작하는 프로퍼티는 반복문의 순환 대상에서 제외

```javascript
let user = {
	name : "John",
	age : 20,
	_password : "123123"
};

user = new Proxy(user, {
	ownKeys(target) { 
		return Object.keys(target).filter(key => !key.startsWith('_'));
	}
});

for (let key in user) alert(key); // name, age
alert( Object.keys(user) ); // name,age
alert( Object.values(user) ); // John,20
```

예) 빈 문자열을 출력하는 Object.keys()

```javascript
let user = { };

user = new Proxy(user, {
  ownKeys(target) {
    return ['a', 'b', 'c'];
  }
});

alert( Object.keys(user) );
```

객체 내에 존재하지 않는 키를 반환하려고 하면 `Object.keys`는 키를 제대로 보여주지 않는다.

이유는 `Object.keys`는 enumerable 플래그가 있는 프로퍼티만 반환한다.  이를 확인하기 위해 Object.keys는 내부 메서드인 `[[GetOwnProperty]]` 를 호출해 모든 프로퍼티의 설명자를 확인한다. 위 예시의 프로퍼티는 설명자가 없고, enumerable 플래그도 없어서 순환 대상에서 제외된다.

만약 Object.keys 호출 시 프로퍼티를 반환하게 하려면 1. enumerable 플래그를 붙여서 프로퍼티가 객체에 존재하도록 하거나, 2. [[GetOwnProperty]]가 호출될 때 중간에 가로채서 설명자 enumerable: true를 반환하게 하면 된다. getOwnPropertyDescriptor 트랩이 이 때 사용된다.

```javascript
let user = { };

user = new Proxy(user, {
  ownKeys(target) { // 프로퍼티 리스트를 얻을 때 딱 한 번 호출
    return ['a', 'b', 'c'];
  },

  getOwnPropertyDescriptor(target, prop) { // 모든 프로퍼티를 대상으로 호출
    return {
      enumerable: true,
      configurable: true
    };
  }

});

alert( Object.keys(user) ); // a, b, c
```

### deleteProperty와 여러 트랩을 사용한 프로퍼티 보호

`_` 이 붙은 프로퍼티나 메서드는 내부용으로만 쓰도록 하는 컨벤션은 널리 사용되고 있는 컨벤션 중 하나이다. `_` 이 앞에 붙으면 객체 바깥에선 이 프로퍼티에 접근해선 안된다.

```javascript
let user = {
  name: "John",
  _password: "비밀"
};

alert(user._password); // 비밀
```

프록시를 이용해 `_` 로 시작하는 프로퍼티에 접근하지 못하도록 막아야 한다.

기능 구현을 위해선 아래의 트랩이 필요한다.

- get : 프로퍼티를 읽으려고 하면 에러 던지기
- set : 프로퍼티에 쓰려고 하면 에러 던지기
- deleteProperty : 프로퍼티를 지우려고 하면 에러를 던지기
- ownKeys : `for..in`이나 `Object.keys`와 같은 순환 메서드를 사용할 때 _로 시작하는 메서드는 제외

```javascript
let user = {
    name: 'John',
    _password: '***',
};

user = new Proxy(user, {
    get(target, prop) {
        // console.log('target : ' + target); // user._password로 호출 시 user객체
        // console.log('prop : ' + prop); // user._password로 호출 시 _password

        if (prop.startsWith('_')) {
            throw new Error('접근이 제한되어있습니다.');
        }
        let value = target[prop];
        return typeof value === 'function' ? value.bind(target) : value;
    },
    set(target, prop, val) {
        if (prop.startsWith('_')) {
            throw new Error('접근이 제한되어있습니다');
        } else {
            target[prop] = val;
            return true;
        }
    },
    deleteProperty(target, prop) {
        if (prop.startsWith('_')) {
            throw new Error('접근이 제한되어있습니다');
        } else {
            delete target[prop];
            return true;
        }
    },
    ownKeys(target) {
        return Object.keys(target).filter((key) => !key.startsWith('_'));
    },
});

// ------------------------------------------------- //

// "get" 트랩이 _password 읽기를 막는다.
try {
    console.log(user._password); // Error: 접근이 제한되어있습니다.
} catch (e) {
    console.log(e.message);
}

// "set" 트랩이 _password에 값을 쓰는 것을 막는다.
try {
    user._password = 'test'; // Error: 접근이 제한되어있습니다.
} catch (e) {
    console.log(e.message);
}

// "deleteProperty" 트랩이 _password 삭제를 막는다.
try {
    delete user._password; // Error: 접근이 제한되어있습니다.
} catch (e) {
    console.log(e.message);
}

// "ownKeys" 트랩이 순회 대상에서 _password를 제외시킨다.
for (let key in user) console.log(key); // name
```

여기서 get트랩을 살펴보자. value의 형식이 function인 경우 `value.bind(target)`을 호출하고 있다. 왜냐하면 `user.checkPassword()`같은 객체가 _password에 접근할 수 있도록 해야 하기 때문이다.

예) get트랩에서 function일 때 `bind`를 한 이유

```javascript
user = new Proxy() {
    get(target, prop) {
        if (prop.startsWith('_')) {
            throw new Error('접근이 제한되어있습니다.');
        }
        let value = target[prop];
        return typeof value === 'function' ? value.bind(target) : value;
    }, 
    // ... 중략
    
    // checkPassword는 _password를 읽을 수 있어야 한다.
    checkPassword(value) { 
	    return value === this._password;
    }
}
```

user.checkPassword()를 호출하면 this는 프록시로 감싼 user에 접근하게 되는데, this._password는 get트랩을 활성화하기에 에러가 발생한다.

❗️객체 메서드의 컨텍스트를 원본 객체인 target에 바인딩 한 이유는 function으로써 호출이 필요할 때 **target이 this**가 되게 하기 위함이다.❗️

유의점 1 : 메서드가 프록시로 감싸지 않은 객체를 넘기는 상황이 존재 등 이상적인 방법은 아니다. 기존 객체와 프록시로 가마싼 객체가 어디있는지 파악하기 어렵기 때문이다.

유의점 2 : 한 객체를 여러번 프록시로 감쌀 경우 프록시마다 객체에 가하는 수정이 다를 수 있다는 점 또한 유의해야 한다. (프록시를 감싸지 않은 객체를 메서드에 넘기는 경우처럼 예측 불가한 결과가 나타날 수 있다.)

⇒ 클래스 안에 private 프로퍼티를 사용하는건 어떨까? (상속이 불가하겠지만)

### has 트랩으로 범위 내 여부 확인하기

```javascript
let range = {
  start: 1,
  end: 10
};

range = new Proxy(range, {
  has(target, prop) {
    return prop >= target.start && prop <= target.end;
  }
});

alert(5 in range); // true
alert(50 in range); // false
```

### apply 트랩으로 함수 감싸기 (데코레이터 함수의 단점을 Proxy로 극복)

함수 역시 프록시로 감쌀 수 있다. 

`apply(target, thisArg, args)`트랩은 프록시를 함수처럼 호출하려고 할 때 동작한다.

- target : 타깃 객체
- thisArg - this의 값
- args - 인수 목록

예) 함수 기반으로 작성된 데코레이터

```javascript
// decorator 함수
function delay(f, ms) {
    // ms 시간이 흐른 다음, f호출을 전달해주는 래퍼 함수를 반환
    return function () {
        setTimeout(() => f.apply(this, arguments), ms);
    };
}

function sayHi(user) {
    console.log(`Hello, ${user}`);
}

console.log(sayHi.length); // 1

// 래퍼 함수
sayHi = delay(sayHi, 3000);

console.log(sayHi.length); // 0
sayHi('Tanon');
```

데코레이터 함수에서 return은 일정 시간이 지난 후 함수를 호출하게 한다. 

- 단점

  그런데 래퍼 함수는 프로퍼티 읽기/쓰기 등의 연산은 전달하지 못한다. 즉 래퍼함수로 감싸고 난 다음, 기존 함수의 프로퍼티인 (name, length 등)의 정보는 사라진다.

- 해결책

  Proxy 객체는 타깃 객체에 모든 것을 전달해주므로 훨씬 강력하다.

예) 래퍼 함수 ⇒ Proxy로 변경

```javascript
// decorator 함수
function delay(f, ms) {
    return new Proxy(f, {
        apply(target, thisArgs, args) {
            setTimeout(() => {
                target.apply(thisArgs, args);
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

결과는 같은데, 프록시에 가하는 모든 연산이 원본 함수에 잘 전달이 되었다. 또한 get length 연산까지 타깃 객체에 전달해준다.

### 취소 가능한 프록시

취소 가능한 프록시는 비활성화할 수 있는 프록시이다. 리소스가 존재하는데, 언제든지 리소스에 대한 접근을 차단하고 싶다는 가정이 있다. 이 때 취소 가능한 프록시로 래핑하면 된다.

예) 문법

```javascript
let {proxy, revoke} = Proxy.revocable(target, handler);
```

```javascript
let object = {
    data: 'Valuable data',
};

let { proxy, revoke } = Proxy.revocable(object, {});

console.log(proxy.data); // Valuable data

revoke();

console.log(proxy.data); // TypeError: Cannot perform 'get' on a proxy that has been revoked
```

revoke()는 프록시에서 대상 객체에 대한 모든 내부 참조를 제거하기에 더이상 연결되지 않는다. 가비지 콜렉터에 의해 객체가 처리될 수 있다.

또한 프록시 객체를 쉽게 찾을 수 있도록 WeakMap에 revoke를 저장할 수 있다.

예) WeakMap에 revoke 객체 담기

```javascript
let revokes = new WeakMap();

let object = {
    data: 'Valuable data',
};

let { proxy, revoke } = Proxy.revocable(object, {});
revokes.set(proxy, revoke);

revoke = revokes.get(proxy);
console.log(proxy.data); // Valuable data

revoke();

console.log(proxy.data); // TypeError: Cannot perform 'get' on a proxy that has been revoked
```

프록시 객체가 더이상 도달할 수 없게 되면(참조하는 변수가 없게 되면) 더이상 필요없는 WeakMap 객체와 함께 메모리에서 지워질 수 있다.
