---
title: "Property + Object.defineProperty (2)"
date: 2025-04-15 21:29:00 +0900
categories: [javascript, object]
tags: [javascript, property]
---

## 4. DefineProperty

`Object.defineProperty()` 는 Javascript에서 객체의 property를 추가하거나 property 속성(attribute)를 변경 메서드이다. 이 이 메서드를 사용하면 객체에 새로운 프로퍼티를 추가하거나, 기존 프로퍼티 속성을 변경할 수 있다.

### defineProperty 인수

```javascript
const person = {};

Object.defineProperty(person, 'firstName', {
	value : 'tanon',
	writable : true,
	enumerable : true,
	configurable : true
})
```

- 첫번째 인수 : 객체 참조
- 두번째 인수 : 프로퍼티 키
- 세번째 인수 : Object Descriptor 객체

### Object Descriptor 객체의 기본값

- value : undefined
- get : undefined
- set : undefined
- writable : false
- enumerable : false
- configurable : false

**속성 설명**

**configurable**

속성 값을 변경할 수 있고, 객체에서 삭제가 가능하면 true이다.

**enumerable**

객체의 속성 열거 시 노출되면 true이다.

**value**

프로퍼티 속성에 연관된 값이다.

**writable**

할당 연산자로 속성의 값을 바꿀 수 있으면 true이다.

**get**

속성의 접근자로 사용할 함수이다. 

optional

**set**

속성의 설정자로 사용할 함수이다.

optional

## 5. DefineProperties

defineProperty는 하나의 프로퍼티만 정의할 수 있지만, `defineProperties`는 여러개의 프로퍼티를 정의할 수 있다.

```javascript
const obj = {};

Object.defineProperties(obj, {
  name: {
    value: "John",
    writable: true
  },
  age: {
    value: 30,
    writable: false
  },
  city: {
    value: "New York",
    writable: true
  }
});

console.log(obj.name); // "John"
console.log(obj.age); // 30
console.log(obj.city); // "New York"
```
