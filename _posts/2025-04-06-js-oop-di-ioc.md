---
title: "DI, IoC"
date: 2025-04-06 00:04:00 +0900
categories: [javascript, oop]
tags: [javascript, dependency-injection]
---

## DI(Dependency Injection)

### Dependency

의존성 : 어떤 객체가 다른 객체와 직접 상호작용 하거나, 어떤 객체가 다른 객체를 참조한다면 그 객체는 다른 객체에 `의존성`을 갖는다.

### DI

⇒ 이 의존성을 직접 갖도록 하는게 아닌, 제 3자로부터 주입받는 것을 의존성 주입이라고 한다.

비유 : 의존성 주입이란 어떤 객체가 다른 객체를 필요로 할 때 그 객체를 가져오는게 아닌, 필요하다고 선언만 하면 프로그램이 대신 가져다주는 역할. (다른 제 3자가 대신 주입해준다.)

일반적으로 프로그램에게 어떤 객체를 사용하겠다는 요청을 하면 프로그램이 그 객체를 대신 생성해서 주입해주기 때문에 객체 간의 의존성을 낮출 수 있다.

예) 강한 결합(Dog - PetOwner)

```javascript
class Dog {
    walk() {
        console.log('산책');
    }
}

class PetOwner {
    constructor() {
        this.animal = new Dog();
        this.animal.walk();
    }
}
```

PetOwner라는 객체는 Dog라는 객체를 new 키워드로 인스턴스를 생성하여 사용하게 된다. 강한 의존관계가 성립되고, Dog 구현체가 변경이 되는 경우 PetOwner는 Dog를 직접 참조하기 때문에 Dog가 변경이 될 때마다 PetOwner를 바꿔주어야 하는 문제가 생긴다.

결론 : 강한 의존관계가 일어나서는 안된다.

그러면 어떻게 요소들 사이의 결합을 약하게 할 수 있을까? ⇒ 객체지향 프로그래밍에서는 인터페이스를 선언하여 해당 인터페이스를 implements하는 클래스를 가져오게 하면 된다.
