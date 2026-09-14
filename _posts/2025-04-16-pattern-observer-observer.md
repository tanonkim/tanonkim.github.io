---
title: "1. 옵서버 패턴"
date: 2025-04-16 09:55:00 +0900
categories: [design-pattern, observer]
tags: [design-pattern, observer]
---

## 옵서버 패턴이란

여러 객체에 자신이 관찰중인 객체에 발생하는 모든 이벤트에 대해 알리는 구독 매커니즘을 정의할 수 있는 행동 디자인 패턴이다.

- 사용사례 : 다른 객체들과 클래스들과 결합하지 않고 해당 객체들에서 발생하는 이벤트들에 반응하는 방법을 제공한다.

### 비유상황

**문제상황**

`손님`/`가게` 두가지 유형의 객체가 있다. 손님은 가게에 곧 출시될 상품에 매우 관심이 있다. 매일 방문해서 재고를 확인할 수 있으나, 가게에 운송되기 전까지 이런 방문은 무의미하다.

반면 가게에서는 상품이 출시될 때마다 모든 고객에게 스팸으로 간주될 수도 있는 수많은 메일을 보낼 수 있다. 이 메일은 일부 손님들을 잦은 가게 방문을 부터 해결할 수 있으나, 동시에 신상품에 관심이 없는 고객들은 불만이 가득찰 것이다.

여기서 충돌이 발생한다. 손님들이 신상품의 출시를 확인하기 위해 시간을 낭비하든, 가게가 알림을 원치 않는 손님들까지 신상품 출시를 알리며 자원을 낭비해야 한다.

**해결책**

시간이 지나면 변경될 수 있는 중요한 상태를 가진 객체가 있다고 가정하자. 이 객체는 `주제`(subject)라고 부른다.

`옵서버 패턴`은 가게 클래스에 개별 객체들이 그 가게로 오는 이벤트들의 알림을 구독 또는 구독 취소할 수 있는 **`구독 매커니즘`**을 추가할 것을 제안한다.

구독 매커니즘은 1. 구독자 객체들에 대한 참조 리스트를 저장하기 위한 배열 필드와 2. 구독자들을 추가하거나 제거할 수 있는 공개된(public) 메서들로 구성된다.

가게에서 중요한 이벤트가 발생할 때마다 구독자 리스트를 참조한 후 그들의 객체에 있는 특정 알림 메서드를 호출한다.

![](/assets/img/2025-04-16/pattern-observer-observer-1.png)

실제 앱에서는 같은 가게 클래스의 이벤트를 추적하는데 관심이 있는 수십개의 다른 구독자 클래스가 존재할 수 있다. 그치만 가게를 모든 클래스에 연결하고 싶지도 않고, 다른 사람에 의해 사용되어야 한다면, 손님 중 일부는 구독자 클래스 중 일부를 미리 알지 못할 수도 있다.

⇒ 모든 구독자(손님)이 같은 인터페이스를 구현하고, 가게(출판사)가 오직 그 인터페이스만을 통해 구독자들과 통신하는 것이 매우 중요하다.

이 인터페이스는 가게(출판사)가 알림과 데이터를 전달하는데 사용할 수 있는 매개변수의 집합과 알림 메서드를 선언해야 한다.

![](/assets/img/2025-04-16/pattern-observer-observer-2.png)

**결론**

신문/잡지를 구독을 하면 다음 호가 있는지 확인하러 가게에 갈 필요가 없다. 대신 출판사가 발행 후에 구독자의 우편으로 보낸다.

출판사는 구독자 리스트를 관리하고, 어떤 신문/잡지에 관심이 있는지 알고 있고, 출판사가 새로운 잡지를 보내는 것을 중단하고 싶다면 구독자는 언제든지 떠날 수 있다.

## 구조

![](/assets/img/2025-04-16/pattern-observer-observer-3.png)

1. 출판사
   1. 다른 객체들에 관심 이벤트를 발행한다.
   2. 구독 인프라가 포함되어있고, 이 인프라는 구독취소/ 구독할 수 있도록 한다.
2. 새 이벤트가 발행하면 출판사는 구독자 리스트를 살핀 후 각 구독자 객체의 구독자 인퍼테이스에 선언된 메서드를 호출한다.
3. 구독자 인터페이스는 알림 인터페이스를 선언하며 단일 update 메서드로 구성된다.
4. 콘크리트 구독자들은 출판사가 보낸 알림에 대해 응답으로 작접을 수행한다.
   1. 이런 모든 클래스는 출판사가 구상 클래스들과 결합하지 않도록 같은 인터페이스를 구현해야 한다.
5. 구독자들은 올바르게 업데이트를 처리하기 위해 컨텍스트 정보가 필요하다.
   1. 출판사 자신을 인수로 전달 가능하며, 구독자가 필요한 데이터를 직접 가져오도록 한다.
6. 클라이언트는 출판사 및 구독자 객체를 별도로 생성한 후 구독자들을 출판사 업데이트에 등록한다.

### 간단한 용어

- **Subject (관찰 대상)**: 상태를 가지고 있고, 변경되면 관찰자에게 알림을 줌.
- **Observer (관찰자)**: 알림을 받는 쪽. Subject가 알림 주면 거기에 반응함.
- **subscribe / unsubscribe**: 관찰을 시작하거나 중단함.
- **notify**: 알림을 보내는 행동.

## 의사코드

![](/assets/img/2025-04-16/pattern-observer-observer-4.png)

## 적용

- 옵서버 패턴은 한 객체의 상태가 변경되어 다른 객체들을 변경해야 할 필요가 있을 때, 실제 객체 집합들을 미리 알 수 없거나 동적으로 변경될 때 사용해야 한다.
- 옵서버 패턴은 앱의 일부 객체들이 제한된 시간 동안, 또는 특정 경우에만 다른 객체들을 관찰해야 할 때 사용해야 한다.

## 장단점

- 개방폐쇄 원칙이 가능하다

  출판사의 코드를 변경하지 않고도 새 구독자 클래스를 도입할 수 있다.

- 런타임에 객체 간의 관계들을 형

## 예시코드1

피자집 주인(Subject)이고, 손님들(Observers)이 오면 “새 피자가 나오면 알려줘”라고 알림 신청함, 피자가 나오면 알림을 신청한 사람들에게 피자 나왔어요라고 알림

(어떤 객체(Subject)의 상태 변화가 있을 때, 이를 구독(Subscribe)한 다른 객체들에게 자동으로 알려주는 구조)

```typescript
interface Observer {
    update:(message:string) => void;
}

class PizzaShop {
    private observers: Observer[] = [];

    subscribe(observer : Observer) {
        this.observers.push(observer)
    }
    
    unsubscribe(observer : Observer) {
        this.observers = this.observers.filter(obs => obs !== observer);
    }

    notify(message: string) {
        for (const observer of this.observers) {
            observer.update(message);
        }
    }

    newPizza() {
        this.notify('New Pizza Made!')
    }
}

class Customer implements Observer {
    constructor(private name: string){}

    public update(message: string) {
        console.log(`${this.name}님에게 알림: ${message}`);
    }
}

const shop = new PizzaShop();
const alice = new Customer('Tanon');
const bob = new Customer('David');

shop.subscribe(alice);
shop.subscribe(bob);

shop.newPizza();
shop.unsubscribe(alice)
console.log('unsubscribe alice');

shop.newPizza();
```

- 결과

```
Tanon님에게 알림: New Pizza Made!
David님에게 알림: New Pizza Made!
unsubscribe alice
David님에게 알림: New Pizza Made!
```

## 예시 코드2

```typescript
// Subject는 구독자를 관리하는 일련의 방법
interface Subject {
    attach(observer : Observer) : void;
    detach(observer : Observer) : void;
    notify() : void; // 모든 observer에게 이벤트 알림
}

class ConcreteSubject implements Subject {
    public state: number;
    private observers: Observer[] = [];
    
    public attach(observer: Observer): void {
        const isExist = this.observers.includes(observer);
        if (isExist) {
            return console.log('Subject: Observer has been attached already.');
        }

        console.log('Subject : Attached an observer');
        this.observers.push(observer);
    }

    public detach(observer: Observer): void {
        const observerIndex = this.observers.indexOf(observer);
        if (observerIndex === -1) {
            return console.log('Subject : NonExist observer');
        }
        this.observers.splice(observerIndex, 1); // splice is a -> b changes
        console.log('Subject : Detached an observer')
    }
    
    public notify(): void {
        console.log('Subject : Notifying observers..')
        for (const observer of this.observers) {
            observer.update(this);
        }
    }

    public specificBusinessLogic() : void {
        console.log('Subject : Doing something special');
        this.state = Math.floor(Math.random() * (10 + 1));

        console.log(`Subject : My State has just changed to ${this.state}`);
        this.notify();
    }
}

interface Observer {
    update(subject : Subject): void;
}

class ConcreateObserverA implements Observer {
    public update(subject: Subject): void {
        if (subject instanceof ConcreteSubject && subject.state < 3) {
            console.log('ConcreateObserverA : Reacted to the event');
        }
    }
}

class ConcreateObserverB implements Observer {
    public update(subject: Subject): void {
        if (subject instanceof ConcreteSubject && (subject.state === 0 || subject.state >= 2)) {
            console.log('ConcreteObserverB: Reacted to the event.');
        }
    }
}

const subject = new ConcreteSubject();

const observer1 = new ConcreateObserverA();
subject.attach(observer1);

const observer2 = new ConcreateObserverB();
subject.attach(observer2);

subject.specificBusinessLogic();
subject.specificBusinessLogic();

subject.detach(observer2)
subject.specificBusinessLogic();
```

- 결과

```javascript
Subject : Attached an observer
Subject : Attached an observer

Subject : Doing something special
Subject : My State has just changed to 6
Subject : Notifying observers..
ConcreteObserverB: Reacted to the event.

Subject : Doing something special
Subject : My State has just changed to 6
Subject : Notifying observers..
ConcreteObserverB: Reacted to the event.

Subject : Detached an observer
Subject : Doing something special
Subject : My State has just changed to 1
Subject : Notifying observers..
ConcreateObserverA : Reacted to the event
```

## 어디서 사용되나?

리액트 상태관리 자체가 Observer 패턴 기반이다. `useState, useEffect, Recoil, Zustand, Redux` 와 같은 도구들은 내부적으로 Observer 패턴을 사용한다.
