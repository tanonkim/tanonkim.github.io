---
title: "structuredClone()으로 깊은 복사하기"
date: 2025-04-05 16:46:00 +0900
categories: [javascript, object]
tags: [javascript, deep-copy]
---

## 1. 얕은 복사

얕은 복사는 원시(Primitive) 자료형이면 값을 그대로 복사하지만, 객체라면 참조(Reference)를 하므로 공유 참조가 되기에 원본 또는 참조되는 객체 중 하나라도 수정되면 모두 영향을 미치게 된다. 개발자의 실수로 의도하지 않은 변경이 일어날 여지가 존재한다.

### 1.1 Speread Operator

흔히 사용되는 방법으로, 전개 연산자로 얕은 복사를 만드는 방법이다. 중첩된 프로퍼티를 추가 및 수정하면 원본/복사본 모두가 변경이 된다.

```javascript
const originalObject = {
    prop: 'value',
    otherProp: {
        innerOtherProp: true,
    },
};

const shallowCopy = { ...originalObject };

shallowCopy.newProp = 'new value';
shallowCopy.otherProp.newProp = 'new value';

console.log(originalObject);
// {
//   prop: 'value',
//   otherProp: { innerOtherProp: true, newProp: 'new value' }
// }
console.log(shallowCopy);
//{
//  prop: 'value',
//  otherProp: { innerOtherProp: true, newProp: 'new value' },
//  newProp: 'new value' // 원시값은 영향X
//}

```

### 1.2 Object.assign()

> Object.assign()은 프로퍼티의 값을 복사하므로, 깊은 복사를 수행하려면 다른 방법을 사용해야 합니다.<br>만약 값이 객체에 대한 참조라면 참조 값만 복사합니다. - MDN

`Object.assign(target, ...sources);`

- target : 목표 객체, sources를 복사해서 반영 후 반환한다.
- sources : 출처 객체

Object.assign()의 경우에도 마찬가지이다.

```javascript
const originalObject = {
    prop: 'value',
    otherProp: {
        innerOtherProp: true,
    },
};

const shallowCopy = Object.assign({}, originalObject);

shallowCopy.prop = 'changed value';
shallowCopy.newProp = 'new value';
shallowCopy.otherProp.newProp = 'new value';

console.log(originalObject);
// {
//   prop: 'value',
//   otherProp: { innerOtherProp: true, newProp: 'new value' }
// }
console.log(shallowCopy);
//{
//  prop: 'value',
//  otherProp: { innerOtherProp: true, newProp: 'new value' },
//  newProp: 'new value' // 원시값은 영향X
//}
```

## 2. 깊은 복사

깊은 복사는 얕은 복사와는 달리, 복사본이 원본의 참조와 동일한 참조를 공유하지 않는 별개의 독립적인 복사본을 생성한다. 객체의 참조가 아닌, 참조가 되는 모든 객체에 대해 완전히 새로운 사본을 생성한다.

### 2.1 JSON.parse()

JSON.stringify()로 객체를 만들어 JSON string으로 변환한 다음 JSON.parse()로 변환하여 새로운 객체를 만드는 것이다. 일종의 트릭이다.

```javascript
const originalObject = {
    prop: 'value',
    otherProp: {
        innerOtherProp: true,
    },
};

const shallowCopy = JSON.parse(JSON.stringify(originalObject));

shallowCopy.prop = 'changed value';
shallowCopy.newProp = 'new value';
shallowCopy.otherProp.newProp = 'new value';

console.log(originalObject);
// { prop: 'value', otherProp: { innerOtherProp: true } }
console.log(shallowCopy);
// {
//   prop: 'changed value',
//   otherProp: { innerOtherProp: true, newProp: 'new value' },
//   newProp: 'new value'
// }
```

JSON 문법구조가 JS에 비해 훨씬 단순하기 때문에 높은 효율로 파싱이 가능하다는 점에서 종종 언급되는 트릭이다. 단 성능적으로 대단한 이점이 있는 것은 아니다.

또한 `함수`, `Symbol`, `DOM`, `Map`, `Set`, `Date`, `ArrayBuffer`, `RegExp` , `File`등은 직렬화 과정에서 실패하거나 손실되는 한계도 존재한다.

### 2.2 lodash 라이브러리 _.cloneDeep()

그래서 lodash 라이드러리의 cloneDeep()을 사용하는 경우가 많았다. JSON 트릭보다 더 의도에 부합하게 재귀적으로 깊은 복사를 수행하는게 가능하다.

```javascript
const originalObject = {
    prop: 'value',
    otherProp: {
        innerOtherProp: true,
    },
};

const shallowCopy = _.cloneDeep(originalObject);
```

### 2.3 structuredClone()

이런 요구 때문에 네이티브로 깊은 복사를 수행할 수 있는 방법이 추가되었다. MDN에 따르면 구조화된 복제 알고리즘을 사용하여 깊은 복사를 수행할 수 있다. JSON 트릭에 비해 더 제한 사항도 적고, 성능 상 좋다.

```javascript
const originalObject = {
    prop: 'value',
    otherProp: {
        innerOtherProp: true,
    },
};

const shallowCopy = structuredClone(originalObject);

shallowCopy.prop = 'changed value';
shallowCopy.newProp = 'new value';
shallowCopy.otherProp.newProp = 'new value';

console.log(originalObject);
// { prop: 'value', otherProp: { innerOtherProp: true } }
console.log(shallowCopy);
// {
//  prop: 'changed value',
//  otherProp: { innerOtherProp: true, newProp: 'new value' },
//  newProp: 'new value'
// }
```

하지만 `structuredClone()` 역시 여전히 일부 제한사항은 존재한다.

- 클래스 인스턴스인 경우 프로토타입이 폐기되고 일반 객체가 반환된다.
- DOM, 함수의 경우 복사에 실패하고 에러가 발생한다.

브라우저 호환성은 전체 호환이 가능하다. 

호환성은 다음과 같다.

![](/assets/img/2025-04-05/js-structured-clone-1.png)

### 2.3.1 `structuredClone` 의 제한사항

1. 클래스 인스턴스인 경우 프로토타입이 폐기되고 일반 객체가 반환된다.

```javascript
class OriginalClass {
    constructor() {
        this.prop = 'value';
        this.otherProp = {
            innerOtherProp: true,
            // innerfunc: function () { // 주석처리 함 : 왜냐면 함수는 structuredClone()이 불가함
            //     return 'Hello';
            // },
        };
    }
}

const originalInstance = new OriginalClass();

console.log(originalInstance);
console.log(
    `originalInstance instanceof OriginalClass : ${originalInstance instanceof OriginalClass}`
); // ✅ true

const deepCopy = structuredClone(originalInstance);

deepCopy.prop = 'changed value';
deepCopy.newProp = 'new value';
deepCopy.otherProp.newProp = 'new value';

console.log(deepCopy);
console.log(`originalInstance instanceof OriginalClass : ${deepCopy instanceof OriginalClass}`); // ❌ false
console.log(Object.getPrototypeOf(deepCopy)); // Object.prototype

```

1. DOM, 함수의 경우 복사에 실패하고 에러가 발생한다.

예) 함수 깊은 복사 실패

```javascript
// [DataCloneError]: function () { return 'Hello'; } could not be cloned.
class OriginalClass {
    constructor() {
        this.prop = 'value';
        this.otherProp = {
            innerOtherProp: true,
            innerfunc: function () { // Fail
                return 'Hello';
            },
        };
    }
}

const originalInstance = new OriginalClass();
console.log(originalInstance);
```

예) DOM 깊은 복사 실패

```javascript
const div = document.createElement('div');
div.textContent = 'Hello DOM';

const cloned = structuredClone(div); // TypeError
```

DOM은 브라우저 환경에 의존적이고, 내부 상태를 가지고 있어서 `structured clone algorithm`의 벗어나는 객체
