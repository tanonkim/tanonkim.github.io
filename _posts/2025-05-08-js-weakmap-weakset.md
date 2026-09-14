---
title: "WeakMap ↔ Map, WeakSet"
date: 2025-05-08 21:20:00 +0900
categories: [javascript, es6]
tags: [javascript, weakmap]
---

### 1. Map

자바스크립트 엔진은 도달 가능하고, 추후 사용될 가능성이 있는 값을 메모리에 유지한다. 

예1.1) 객체 선언, null로 값 할당

```javascript
let tanon = { name: 'tanon' };

tanon = null; // 객체가 메모리에서 삭제됨
```

객체는 tanon이라는 참조를 통해 접근이 가능하다. 그런데 참조를 null로 할당해버리면 해당 객체에 더 이상 도달 가능하지 않게 된다. 자료구조를 구성하는 요소도 자신이 속한 자료구조가 메모리에 남아있는 동안 대게 도달가능한 값으로 취급되기에 메모리에서 삭제되지 않는다. 객체의 프로퍼티(키+값의 쌍), 배열의 요소, 맵/셋을 구성하는 요소들이 이에 해당된다.

예1.2) 배열에 객체 추가

```javascript
let tanon = { name: 'tanon' };

let array = [tanon];

tanon = null;
console.log(JSON.stringify(array[0])); // {"name":"tanon"}
```

객체를 참조하는게 아무것도 없어도, 배열이 메모리에 남아있는 한(참조되고 있는 한) 배열의 요소인 이 객체도 메모리에 남아있게 된다.

**동작방식**

`{ name: 'tanon' };` 객체가 메모리 생성

tanon변수는 이 객체를 참조함

배열을 할당하고 tanon객체를 삽입, array에 할당

동시에 `array[0] = tanon`에서 array[0]도 같은 객체를 참조함

이후 `tanon = null;`을 하면 tanon변수는 null을 가리키지만, 배열 내부의 array[0]은 여전히 원래의 객체를 참조하고 있다.

따라서 JSON.stringify(array[0])을 실행하면 원래의 객체 문자열로 변환이 된다.

Map에서 객체를 키로 사용한 경우에도 Map이 메모리에 있는 한 객체도 메모리에 남는다. 즉 **가비지 컬렉터**의 대상이 되지 않는다.

예1.3) 객체를 키로 사용한 경우

```javascript
let tanon = { name: 'tanon' };

let map = new Map();
map.set(tanon, '...'); // 객체를 키로 사용한 예시

tanon = null;

for (let obj of map.keys()) {
    console.log(JSON.stringify(obj));
}

console.log(map.size);
```

### 2. WeakMap

**차이점1**

Map과 WeakMap의 첫번째 차이는 WeakMap의 키가 반드시 `객체`여야 한다. 원시값은 WeakMap의 객체가 될 수 없다.

예2.1)

```javascript
let weakMap = new WeakMap();

let obj = {};

weakMap.set(obj, 'ok'); // 객체 : 가능함

weakMap.set('test', 'Whoops'); // TypeError: Invalid value used as weak map key
```

WeakMap은 일반 Map과 전혀 다른 양상을 보인다. WeakMap을 사용하면 키로 쓰인 객체가 GC의 대상이 된다. 

예 2.2) tanon을 나타내는 객체는 이제 메모리에서 사라짐

```javascript
let tanon = { name: 'tanon' };

let weakMap = new WeakMap();
weakMap.set(tanon, '...');

tanon = null;
```

tanon을 나타내는 객체는 오로지 WeakMap의 키로만 사용되고 있으므로 null로 참조를 덮어쓰면 이 객체는 WeakMap과 메모리에서 자동으로 삭제된다.

**차이점2**

WeakMap은 `keys()`, `values()`, `entries()` 메서드를 지원하지 않는다. 따라서 위크맵에서는 키나 값 전체를 얻는게 불가능하다.

위크맵이 지원하는 메서드는 아래와 같다.

- weakMap.get(key);
- weakMap.set(key, value); // 여기서 key는 객체
- weakMap.delete(key);
- weakMap.has(key);

1. 적은 메서드만 제공하는 이유는 가비지컬렉션의 동작 방식 때문이다. 예시의 tanon을 나타내는 객체처럼 객체는 모든 참조를 잃게 되면 자동으로 가비지 컬렉션의 대상이 된다.
2. 가비지 컬렉션이 일어나는 시점은 자바스크립트 엔진이 결정하며, 정확한 동작 시점은 알 수 없다. 객체는 모든 참조를 잃었을 때 그 즉시 메모리에서 삭제될 수 있고, 다른 삭제 작업이 있을 때까지 대기하다가 함께 삭제될 수도 있다. 현재 WeakMap의 요소가 몇 개 있는지 정확하게 파악하는것 자체가 불가능하다. 가비지 컬렉터가 메모리를 한 번에 청소할 수 있고, 부분 부분 메모리를 청소할 수 있기에, 위크맵의 요소(key-value) 전체를 대상으로 무언가를 하는 메서드는 동작 자체가 불가능하다.

### 3. Use Case1 : 방문기록

위크맵은 부차적인 데이터를 저장할 곳이 필요할 때 그 진가를 발휘한다. 

- 가정 : 서드파티 라이브러리와 같은 외부 코드에 속한 객체를 가지고 작업한다.

이 객체에 데이터를 추가해야 하는데, 추가해 줄 데이터는 객체가 살아있을 때만 유효하다. 이럴 때 위크맵을 사용할 수 있다.

위크맵에 원하는 데이터를 저장하고, 이때 키는 객체를 사용하면 되고 객체가 가비지 컬렉션의 대상이 될 때 데이터도 같이 사라진다.

---

- 가정 : 사용자의 특정 페이지를 방문하고, 방문하면 카운트 수가 올라간다

예) 사용자의 방문 횟수를 세어주는 코드

```javascript
// countUser.js
let visitsCountMap = new Map(); // 맵에 사용자의 방문 횟수를 저장함

function countUser(user) {
    let count = visitsCountMap.get(user) || 0;
    visitsCountMap.set(user, count + 1);
}


// main.js
let tanon = { name: 'tanon' };
countUser(tanon);

tanon = null;
```

해당 예시는 사용자의 방문 횟수를 세어주는 코드이다. 관련 정보는 맵에 저장하는데, 맵 요소의 키엔 사용자의 정보를 나타내는 객체를, 값에는 사용자의 방문 횟수를 넣고 있다. 어떤 사용자의 정보를 저장할 필요가 없어지면, 즉 가비지 컬렉션의 대상이 되면 해당 사용자의 방문 횟수도 저장할 필요가 없어지게 된다.

그러나 tanon을 나타내는 객체는 가비지 컬렉션의 대상이 되어야 하는데, visitCountMap의 key(visitsCountMap.set(user, count + 1))로 사용되기에 메모리에서 삭제되지 않는다.

특정 사용자를 나타내는 객체가 메모리에서 사라지면 해당 객체에 대한 정보, 즉 방문 횟수도 사라져야 하는데, 손수 지워줘야 한다. 만약 지우지 않으면 `visitsCountMap` 이 차지하는 메모리 공간이 한없이 커진다. 

이런 문제를 위크맵을 통해 해결할 수 있다.

```javascript
let visitsCountMap = new WeakMap(); // 위크맵에 사용자의 방문 횟수를 저장함

function countUser(user) {
    let count = visitsCountMap.get(user) || 0;
    visitsCountMap.set(user, count + 1);
}

let tanon = { name: 'tanon' };
countUser(tanon);

tanon = null;
console.log(visitsCountMap.get(tanon)); // undefined
```

위큼맵에 사용자 방문 횟수를 저장하면 visitsCountMap을 수동으로 청소할 필요가 없어진다. tanon을 나타내는 **객체가 도달가능하지 않은 상태**가 되면 메모리에서 자동으로 삭제되기 때문이다. 위크맵의 키(tanon)에 해당하는 값도 자동으로 가비지 컬렉션의 대상이 된다.

**궁금한 점 - 언제 객체가 도달가능하지 않은 상태가 되나?**

tanon = null;이 없으면 언제 GC에 의해서 메모리 해제를 당하는가?

```javascript
let visitsCountMap = new WeakMap(); // 위크맵에 사용자의 방문 횟수를 저장함

function countUser(user) {
    let count = visitsCountMap.get(user) || 0;
    visitsCountMap.set(user, count + 1);
}

let tanon = { name: 'tanon' };
countUser(tanon);

console.log(visitsCountMap.get(tanon)); // 1
```

1. tanon을 null로 설정
2. 블록 스코프를 벗어남

### 4. Use Case2 : 캐싱

위크맵은 캐싱이 필요할 때 유용하다. 캐싱은 시간이 오래 걸리는 작업의 결과를 저장해서 연산 시간과 비용을 절약해주는 기법이다. 동일한 함수를 여러번 호출할 때 최초 호출 시 반환된 값을 저장해두었다가 함수를 호출하는 대신 캐싱된 값을 사용한다.

예4.1) 

```javascript
let cache = new Map();

function process(obj) {
    if (!cache.has(obj)) {
        let result = obj; // 연산 수행 로직
        cache.set(obj, result);
    }

    return cache.get(obj);
}

let obj = { obj: 'obj' };
let result1 = process(obj);
let result2 = process(obj);

obj = null;

console.log(cache.size); // 1, 객체에 여전히 캐시가 남아있고 메모리 낭비 중
```

해당 예시는 함수 연산 결과를 맵에 저장하고 있다. process(obj)를 호출하면 맨 처음만 연산이 수행되고, 그 이후엔 연산 결과를 cache에서 가져온다. 그런데 맵을 사용하고 있어서, 객체가 필요없어져도 cache를 수동으로 청소해야 한다.

예4.2)

```javascript
let cache = new WeakMap();

function process(obj) {
    if (!cache.has(obj)) {
        let result = obj; // 연산 수행 로직
        cache.set(obj, result);
    }

    return cache.get(obj);
}

let obj = { obj: 'obj' };
let result1 = process(obj);
let result2 = process(obj);

obj = null;
```

4.2 예시에선 cache.size를 사용할 수 없다. 하지만 obj가 가비지 컬렉션의 대상이 되므로, 캐싱된 데이터 역시 메모리에서 삭제된다. 

### 5. WeakSet

아래는 위크셋의 특징이다.

- 위크셋은 셋과 유사한데, 객체만 저장이 가능하다. 원시값은 저장 불가능하다. 
- 셋 안의 객체는 도달 가능할 때만 메모리에서 유지된다.
- 셋과 마찬가지로 위크셋이 지원하는 메서드는 단출하다. 
  - 사용가능 : add, has, delete
  - 사용불가능 : size, keys() 등 반복 작업 관련 메서드

예) 위크셋의 용도 : 사이트 방문 여부를 추적하는 용도

```javascript
let visitedSet = new WeakSet();

let john = { name: 'John' };
let peter = { name: 'peter' };
let mary = { name: 'Mary' };

visitedSet.add(john);
visitedSet.add(peter);
visitedSet.add(john); // 재방문

console.log(visitedSet.has(john));
console.log(visitedSet.has(mary)); // false

john = null; // visitedSet에서 john의 객체는 자동으로 삭제
```

### 정리

위크맵과 위크셋의 큰 단점은 반복작업이 불가능해 저장된 자료를 한번에 얻는게 불가능하다. 위크맵과 위크셋은 객체와 함께 추가 데이터를 저장하는 용도로 사용가능하다.
