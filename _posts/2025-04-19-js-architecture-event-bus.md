---
title: "EventBus"
date: 2025-04-19 15:02:00 +0900
categories: [javascript, architecture]
tags: [javascript, event-bus]
---

## EventBus 아키텍쳐 패턴

소프트웨어 개발에서 디자인패턴은 재사용 가능한 솔루션을 제공하고, 가장 중요한 점은 코드 유지관리를 더 쉽게 만들어준다.

EventBus 아키텍쳐는 일반적으로 Pub/Sub 패턴과 연관된다. 이벤트 버스는 애플리케이션 내의 어느 곳에서나 엑세스 할 수 있도록 메시지나 이벤트를 전송하는 글로벌한 방법으로 볼 수 있다.

## 이벤트 버스의 영감

이벤트버스의 개념은 버스 토폴로지/버스 네트워크에서 유래된다. 버스 토폴로지에서는 노드가 “버스”에 직접 연결된다. 아래 이미지처럼 모든 스테이션은 모든 네트워크 트래픽을 수신한다.

![](/assets/img/2025-04-19/js-architecture-event-bus-1.png)

소프트웨어의 컨텍스트에선 컴퓨터 대신 객체로 생각할 수 있다. 그러면 메세지는 이벤트버스를 통해 이벤트를 트리거할 수 있고, 이벤트를 통해 일부 데이터를 포함하여 데이터를 전송할 수 있다.

## TypeScript로 구현

### 1. 인터페이스 정의

scrc/event-bus/event-bus.types.ts

```typescript
export interface Registry {
    unregister: () => void;
}

export interface Callable {
    [key: string] : Function;
}

export interface Subscriber {
    [key: string] : Callable;
}

export interface IEventBus {
    dispatch<T>(event: string, arg?: T) : void;
    register(event:string, callback : Function) : Registry;
}
```

- Callable

  key : 문자열, 값 : function

  이벤트이름 → 콜백함수 구조로 활용

- Subscriber

  이벤트 이름을 string키로, 그 키 안에 여러개의 Callable을 가짐

  중첩된 딕셔너리 구조, 이벤트 이름별로 여러 리스너를 저장하고 관리함

- IEventBus

  `dispatch<T>(event: string, arg?: T) : void;`

  이벤트를 발행(emit)하는 메서드

  `register(event:string, callback : Function) : Registry;`

  이벤트를 구독하는 메서드

  리턴값은 Registry 객체 → unregister()메서드로 구독해제를 하도록 할 수 있음

### 2. 이벤트버스 구현

예1)

```typescript
import { IEventBus, Registry, Subscriber } from "./event-bus.types";

export class EventBus implements IEventBus {
    private subscribers : Subscriber;
    private static nextId = 0;
    public static instance?: EventBus = undefined;

    private constructor() {
        this.subscribers = {};
    }

    public static getInstance() : EventBus {
        if (this.instance === undefined) {
            this.instance = new EventBus();
        }

        return this.instance;
    }
	 // 예2, 3)에서 진행 : dispatch(), register() 구현
}
```

- 필드

  subscribers : 구독자 콜백을 저장, 딕셔너리 구조이며 내부 구조는 Subscriber 타입

  nextId : 각 콜백에 부여할 ID, 등록 시 하나씩 증가

  instance : 싱글톤 인스턴스를 담는 static 변수

- 생성자

  생성자 메서드는 비공개, 어느 곳에서도 객체를 생성하는 것을 불허

- `public static getInstance() : EventBus` : 싱글턴 패턴 적용

  싱글턴 패턴 적용 이유 → 이벤트 버스는 애플리케이션 어느 곳에서 접근가능해야 하므로 고유한 인스턴스를 가지는게 중요함. (객체를 한 번만 인스턴스화 함)

예2)

```javascript
    public dispatch<T>(event: string, arg?: T): void {
        const subscriber = this.subscribers[event];

        if (subscriber === undefined) return;
        
        Object.keys(subscriber).forEach((key) => subscriber[key](arg));
    }
```

- `public dispatch<T>(event: string, arg?: T) {}`
  1. subscribers는 딕셔너리 형태, 키에 해당하는 값이 없다면 return

     → EventBus.getInstance().register(**Event.HelloWorld**, (name:string) ⇒ {}

     즉 사전에 등록되지 않은 이벤트명이 존재한다면 return

  2. `Object.keys(subscriber).forEach((key) => subscriber[key](arg));`

     해당 이벤트에 등록된(존재하는 키에 대한) 콜백함수를 실행

     subscriber는 아래와 같은 객체 형태

     ```javascript
     export interface Callable {
         [key: string] : Function;
     }
     
     export interface Subscriber {
         [key: string] : Callable; // 하나의 객체에 여러개의 키-쌍 값이 가능함
     }
     
     {
     	"hello-world" : (arg) => {...},
     	"good-bye" : (arg) => {...},
     	...
     }
     ```

예3)

```javascript
    public register(event: string, callback: Function): Registry {
        const id = this.getNextId();
        if (!this.subscribers[event]) this.subscribers[event] = {};
        
        this.subscribers[event][id] = callback;
        
        return {
            unregister: () => {
                delete this.subscribers[event][id];
                if (Object.keys(this.subscribers[event]).length === 0) {
                    delete this.subscribers[event];
                }
            }
        }
    }

    private getNextId() : number {
        return EventBus.nextId++;
    }
```

- `public register(event: string, callback: Function): Registry {}`

  id : 각 콜백 함수마다 고유 ID 부여

  `if (!this.subscribers[event]) this.subscribers[event] = {};` : 이벤트 키가 등록되지 않은 경우 비어있는 객체를 넣음

  `this.subscribers[event][id] = callback;` : 해당 이벤트에 고유 ID로 콜백 등록

  `return {unregister: () ⇒ {}}` : unregister 함수를 포함한 객체 반환

  delete this.subscribers[event][id]; id에 해당하는 이벤트 콜백을 삭제하고, 만약 이벤트에 콜백이 없다면 이벤트 삭제

### 3. 이벤트버스 사용

```typescript
import { EventBus } from "./event-bus/event-bus";

enum Event {
    HelloWorld = 'hello-world'
}

const registry = EventBus.getInstance().register(Event.HelloWorld, (name:string) => {
    if (name) {
        console.log('Hello ' + name);
    } else {
        console.log('Good Morning');
    }
});

EventBus.getInstance().dispatch<string>(Event.HelloWorld, 'Tanon'); // Hello Tanon
EventBus.getInstance().dispatch<string>(Event.HelloWorld); // Good Morning
registry.unregister();
EventBus.getInstance().dispatch<string>(Event.HelloWorld); // X
```
