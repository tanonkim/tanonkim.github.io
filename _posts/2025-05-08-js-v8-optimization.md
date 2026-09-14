---
title: "V8엔진의 최적화"
date: 2025-05-08 20:27:00 +0900
categories: [javascript, v8]
tags: [javascript, v8]
---

## V8 엔진

구글에서 개발한 오픈 소스 자바스크립트 엔진이며, 자바스크립트 코드를 실행하는데 사용되고 C++로 작성되었다. chrome, safari를 포함한 여러 브라우저가 V8 엔진을 사용중이다.

V8은 속도 항샹을 위해 **`JIT`(Just In Time) `컴파일러`**를 사용해 코드를 실행하기 전에 기계어로 번역하여 실행한다. 기계어 코드는 컴퓨터의 하드웨어에서 직접 실행이 되기에 저수준이며, 속도 측면에서 이점이 있다.

JIT는 실행중에 최적화가 가능한 코드를 `TurboFan`을 이용해 최적화한다.

## JIT 동작원리

프로그램을 실행할 때 **인터프리터를 사용해 일부 코드를 실행**하면서, **반복된는 코드를 컴파일**하여 기계어로 변환하는 방식이다.

과정은 아래와 같다.

1. 낱말 분석 과정을 통해 코드를 토큰화한다. parser에서 분해된 토큰을 바탕으로 `tree`를 생성하는데, 추상구문 트리(Abstract Syntax Tree)라고 한다.
2. 인터프리터를 통해 AST를 기반으로 코드를 실행한다.
3. 인터프리터는 코드 실행과정에서 `프로파일링` 정보를 수집한다.(함수 호출 빈도, 변수 타입 등)
4. 프로파일링 정보를 기반으로 JIT 컴파일러가 중간표현(Intermediate Representation _ IR)을 생성한다.
5. `터보팬`에서는 IR 그래프를 분석해 최적화하는 시도를 한다.
   1. 최적화 기술 중 인라인 캐싱, 히든 클래스 등이 사용된다.
6. 최적화된 IR 코드가 기계코드로 컴파일되고, 메모리에 로드된다.
7. 기계코드를 실행한다.

## 인라인캐싱, 히든클래스

터보팬에서 IR 그래프를 분석해 최적화하는 시도를 하고 인라인캐싱, 히든 클래스와 같은 기술을 사용한다.

### **인라인캐싱 : 객체 타입의 캐시를 유지**

호출 지점에서 이전의 메서드 검색의 결과를 기억해서 런타임 메서드 바인딩의 성능을 높이는 것이다. 함수를 호출할 때 같은 call site에서는 같은 함수를 불러올 가능성이 크다는 관찰을 기반으로 한다.

V8 엔진은 메서드 호출에 따라 파라미터로 전달된 객체 타입의 캐시를 유지하고, 이 정보를 이용해 앞으로 파라미터로 넘어올 객체의 타입에 대해 가정을 한다. (일치된 객체 구조로 사용하기)

인라인 캐시는 최적화 과정에서 객체의 속성에 접근하는 부분에 실제 메모리 주소를 할당하여 lookup 과정을 생략한다. 따라서 반복적으로 접근하는 객체의 구조가 동일해야 객체를 구분하는 과정을 생략하고 offset을 적용해서 최적화가 용이하게 된다.

예1) 일치되지 않은 객체 구조

```javascript
(() => {
    const tanon = { firstname: 'Tanon', lastname: 'Kim', job: 'developer' };
    const sujin = { firstname: 'Sujin', lastname: 'Yoon', age: 50 };
    const sangmin = { firstname: 'Sangmin', lastname: 'Park', gender: 'female' };
    const jimin = { firstname: 'Jimin', lastname: 'Ko', hobby: 'fishing' };
    const minjoon = { firstname: 'MinJoon', lastname: 'Lee', nickname: 'Coco' };

    const getFullName = (user) => `${user.lastname} ${user.firstname}`;

    const people = [tanon, sujin, sangmin, jimin, minjoon];

    console.time('최적화 되지 않은 코드');
    for (let i = 0; i < 1_000_000_000; i++) {
        getFullName(people[i % 5]);
    }
    console.timeEnd('최적화 되지 않은 코드');
})();
// 최적화 되지 않은 코드: 13.278s (평균)

```

예2) 일치된 객체 구조

```javascript
(() => {
    const tanon = { firstname: 'Tanon', lastname: 'Kim' };
    const sujin = { firstname: 'Sujin', lastname: 'Yoon' };
    const sangmin = { firstname: 'Sangmin', lastname: 'Park' };
    const jimin = { firstname: 'Jimin', lastname: 'Ko' };
    const minjoon = { firstname: 'MinJoon', lastname: 'Lee' };

    const getFullName = (user) => `${user.lastname} ${user.firstname}`;

    const people = [tanon, sujin, sangmin, jimin, minjoon];

    console.time('최적화 된 코드');
    for (let i = 0; i < 1_000_000_000; i++) {
        getFullName(people[i % 5]);
    }
    console.timeEnd('최적화 된 코드');
})();
// 최적화 된 코드: 9.607s (평균)
```

예1)과 예2) 차이는 사용자 객체의 구조 뿐이다. 그런데 약 4초 가량의 시간 차이를 보인다.

인라인 캐시는 최적화 과정에서 객체의 속성(attribute)에 접근하는 부분에 실제 메모리 주소를 할당하여 lookup 과정을 생략한다. 따라서 반복적으로 접근하는 객체의 구조가 같아야 구분하는 과정을 생략하고 offset을 적용하여서 최적화가 용이하게 한다.

위 코드 예시에선 user.lastname의 attribute 접근 과정에서 해당 object가 lastname, firstname이라는 속성의 위치를 offset을 저장하여 static하게 최적화 코드에 넣어버린다.

### 히든 클래스

자바스크립트는 동적 언어이며, 객체가 생성된 이후에도 속성을 추가하거나 삭제할 수 있다. 히든 클래스는 객체지향 언어에서 사용되는 고정 객체 레이아웃과 유사하게 작동하는데, **런타임**에 생성된다는 차이점이 있다. V8엔진은 히든 클래스를 활용하여 객체의 프로퍼티 구조를 효율적으로 처리한다.

터보팬은 히든 클래스 정보를 활용하여, 객체에 대한 프로퍼티 접근을 최적화한다.

예)

```javascript
function Point(x,y) {
	this.first = x; // 2.
	this.second = y; // 3.
}

const p1 = new Point(1,2); // 1. 
```

1. V8엔진은 C0 히든 클래스를 생성하고, p1 객체는 C0을 참조한다.

![](/assets/img/2025-05-08/js-v8-optimization-1.png)

1. B과정에서 V8엔진은 C0을 기반으로 C1이라는 두번째 히든 클래스를 생성한다.

![](/assets/img/2025-05-08/js-v8-optimization-2.png)

- property table : 프로퍼티가 메모리의 어디를 참조해야 하는지 알려주는 이정표 역할
- transition table : 구조가 변하면 어디로 가야 하는지 알려주는 이정표 역할

1. C과정에서 V8엔진은 C2 히든 클래스를 만들고 클래스 전환을 통해서 업데이트 한다.

![](/assets/img/2025-05-08/js-v8-optimization-3.png)

**주의점**

위와 같은 과정을 거치며 히든 클래스가 생성되는데, 새로운 히든 클래스를 생성할 때마다 기존에 생성된 히든 클래스를 이용할 수 있도록 작성해야 한다.

예) 히든 클래스 관점에서 좋지 않은 코드

```javascript
function Point(x,y) {
	this.first = x;
}

const p1 = new Point(1);
const p2 = new Point(2);

p1.second = 10;
p1.third = 100;

p2.third = 100;
p2.second = 10;
```

이유 : p1과 p2는 동일한 데이터를 가지고 있지만, 서로 다른 히든 클래스를 참조한다. (일치되지 않는 객체 구조)

이와 같은 코드는 offset을 참조하지 못하므로 인라인 캐싱이 되지 않고, 여러개의 히든 클래스를 만드는 문제를 초래한다.

1. p1은 second → third 순서로 속성을 추가한다.
2. p2는 third → second 순서로 추가한다.

⇒ p1과 p2는 서로 다른 히든 클래스를 사용한다.

예) 최적화를 위한 수정 방법

```javascript
function Point(x,y) {
	this.first = x;
	this.second = 0;
	this.thrid = 0;
}

const p1 = new Point(1);
const p2 = new Point(2);

p1.second = 10;
p1.third = 100;

p2.second = 10;
p2.third = 100;
```

p1, p2는 같은 순서와 구조의 히든 클래스를 공유하게 되어 V8 최적화가 된다.

**결론**

히든 클래스 관점에서 **모든 인스턴스가 동일한 프로퍼티를 같은 순서로 가질 수 있게 설계**하기

프로퍼티를 조건문 없이 항상 생성자에서 정의하기

### 코드를 이렇게 작성하자

1. 객체 속성을 항상 같은 순서로 초기화
   1. 히든 클래스 이후에 새성되는 최적화 코드가 공유될 수 있도록 하기
2. 인라인 캐싱을 위해 히든 클래스를 최소화하기
   1. 비슷한 유형의 객체를 사용해 여러개의 히든클래스를 만들지 않기
3. 동일한 메서드를 반복적으로 수행하는 코드가 속도가 더 빠름을 인지
   1. 서로 다른 메서드를 한 번씩 수행하는 코드 X
