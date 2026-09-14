---
title: "Node.js Event Driven Architecture"
date: 2025-04-20 20:37:00 +0900
categories: [javascript, architecture]
tags: [javascript, event-driven]
---

## Event-Driven Programming

이벤트 주도 프로그래밍은

1. 프로그램의 실행 흐름이 이벤트에 의해 결정되도록
2. 오브젝트, 서비스와 같은 개체들이 **중간 매체**를 통해 간접적으로 메세지를 주고받으며 의사소통하는 패러다임

이벤트 주도 프로그래밍에서는 두 개체가 직접적인 관계를 가지고 있지 않고, 송신자/수신자 사이를 **메세지가 중개**한다. → 시스템 내 서로간의 결합을 약하게 함으로써 시스템의 복잡도를 낮춘다.

**이벤트 주도 프로그래밍에서는 생산자는 소비자를, 소비자는 생산자를 알 수 없다**

- 이벤트를 발생시키는 개체 : 생산자(producer)
- 이벤트를 소비하여 활용하는 개체 : 소비자(consumer)

그저 메세지 큐를 가운데 두고 상호작용한다.

### 이벤트 루프

하나의 쓰레드 안에 이벤트 핸들러들이 담긴 큐가 존재한다. 주기적으로 쓰레드 안의 핸들러를 꺼내 처리하는 모델이 필요하다.

이 모델의 장점은 I/O 등을 처리할 때 block되지 않는다. 따라서 다른 일들이 처리되도록 기다리는 동안 사용지의 입력을 받는 등 행동들이 가능하다.

### EDA 사용 이유

보통의 객체지향 프로그램에서 하나의 객체 A가 다른 객체 B로 메세지를 보내기 위해선, A가 B를 직접적으로 알 고 있어야 하고, B 인터페이스를 알고 있어야 한다는 요구조건을 가진다.

이벤트 주도 프로그래밍에서는 A가 이벤트 큐로 이벤트를 보낼 수 있다는 것 외에는 다른 요구조건을 가지지 않는다.

### **언제 사용하나요?**

이벤트 주도 프로그래밍은 웹/모바일 환경에서 자주 사용된다.

버튼을 누르거나, 웹브라우저의 윈도우 사이즈를 줄이고 늘리거나, 마우스를 드래그 하는 등의 이벤트에 대응할 때 반응시간 때문에 비동기적으로 태스크를 처리해야 한다.

### 장/단점

장점

- 실행순서, 개체들 간 관계에서 유연
- 그래픽 인터페이스에서 유용하게 사용될 수 있음
- 단순하고 이해하기 쉬움

단점

- 더 복잡할 수 있음
- 프로그램 흐름을 파악하기 어려움
- 절차적인 코드에 비해 디버깅의 어려움
- 성능 저하

## Node.js Event

`node.js` 에는 이벤트 처리를 위한 event 라이브러리를 지원한다.

이벤트 기반 : 이벤트 발생에 의해 프로그램 흐름이 결정되는 프로그램 패러다임이다.

### 이벤트, 콜백 등록

```typescript
import { EventEmitter } from 'events';

const emitter = new EventEmitter();

// addListener
emitter.addListener('event1', () => {
  console.log('text');
});

emitter.emit('event1');
```

- addListener(eventName, callback)

  특정 이벤트가 발생할 시 실행할 함수를 등록

- emit(eventName)

  등록된 이벤트를 실행

### on(), once()

```typescript
import { EventEmitter } from 'events';

const emitter = new EventEmitter();

// on, once
emitter.on('eventOn', () => {
  console.log('callback of eventOn');
});
emitter.once('eventOnce', () => {
  console.log('callback of eventOnce');
});

emitter.emit('eventOn');
emitter.emit('eventOnce');

// remove
emitter.removeAllListeners('eventOn');
emitter.emit('eventOn');
```

- on(eventName, callback)

  이벤트 리스너 등록 함수, addListener()와 동일

- once(eventName, callback)

  최초 실행 후 자동으로 제거

- removeAllListeners(eventName)

  등록된 모든 이벤트 리스너 제거

```typescript
//// 리스터 카운팅 함수
const listener1 = () => console.log('listener1');
const listener2 = () => console.log('listener2');

const showLs = (e: string) => {
  console.log(emitter.listenerCount(e));
};

emitter.on('eventOn1', listener1);
showLs('eventOn1');
emitter.emit('eventOn1'); // 1

emitter.on('eventOn1', listener2);
showLs('eventOn1');
emitter.emit('eventOn1'); // 2

emitter.removeListener('eventOn1', listener1);
emitter.off('eventOn1', listener2);
emitter.emit('eventOn1'); // X
```

- listenerCount(eventName)

  이벤트에 등록된 리스너의 개수를 반환

- removeListener(eventName, listener)

  이벤트에 정의한 리스너 제거

- off(eventName, listener)

  removeListener()와 같은 역할
