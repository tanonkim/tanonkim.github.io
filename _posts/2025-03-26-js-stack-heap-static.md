---
title: "Stack/Heap/Static Area"
date: 2025-03-26 15:07:00 +0900
categories: [javascript, engine]
tags: [javascript, memory]
---

### 1. 콜스택/메모리 힙

![](/assets/img/2025-03-26/js-stack-heap-static-1.png)

자바스크립트 엔진은 Memory Heap과 Call Stack으로 구성되어있다. 자바스크립트는 싱글 쓰레드 프로그래밍 언어인데, 이 의미는 Call Stack이 하나라는 의미이다. (멀티가 아닌 하나씩 처리한다.)

### Call Stack

원시 타입 데이터가 저장된다. 실행 컨텍스트를 통해 식별자 저장, 스코프/체인 및 this 관리, 코드 실행 순서 관리 등을 수행한다.

### Memory Heap

참조 타입 데이터가 저장된다. 메모리 할당이 일어난다.

![](/assets/img/2025-03-26/js-stack-heap-static-2.png)

- 원시타입 데이터

  24, ‘abcd’의 값 자체는 원시타입이므로 콜스택에 저장된다.

  myNumber 변수에는 24가 저장된 콜스택 메모리의 주소값이 저장된다.

  \*변수 식별자 myNumber 자체는 콜스택 상의 실행 컨텍스트의 LexicalEnvironment에 저장된다.

- 참조타입 데이터

  배열, 객체, 함수 등은 참조타입이므로 메모리 힙에 저장된다.

  참조타입 데이터가 저장된 메모리 주소값은 **콜스택**에 저장된다.

  즉 메모리 힙의 주소가 저장된 콜스택의 주소는 myArray에 저장된다.

  \*마찬가지로 변수 myArray 이름 자체는 콜스택 상의 실행 컨텍스트의 LexicalEnvironment에 저장된다.

### 2. 원시 타입의 변수 데이터

1. 원시타입의 데이터 값은 콜스택에 저장되고, 데이터 값이 저장된 주소값은 변수 a,b에 저장된다.

```javascript
let a = 10;
let b = 20;
```

1. 변수 a에 20을 재할당하면 a의 주소값을 b의 주소값으로 변경한다.

```javascript
a = 20;
```

1. 변수 a에 30, 즉 새로운 값으로 재할당하면, 새로운 메모리를 확보하여 30을 저장하고, 여기에 해당하는 주소값을 a의 주소값으로 교체한다.

```javascript
a = 30
```

1. 가비지 컬렉터

더이상 참조되지 않는 10은 (주소/값) 메모리에서 해제된다.

### 자바스크립트의 참조타입

```javascript
let myArray = []
```

1. 변수의 고유 식별자를 생성한다.
2. 콜스택의 메모리에 주소를 할당한다. (런타임)
3. 힙에 할당된 메모리 주소를 콜스택의 값으로 저장한다. (런타임)
4. 힙의 메모리의 값에 빈 배열을 저장한다.

```javascript
myArray.push('first');
myArray.push('second');
myArray.push('thrid');
myArray.push('fourth');
myArray.pop();
```

![](/assets/img/2025-03-26/js-stack-heap-static-3.png)

### 3. let, const

일반적으로 변수가 변경될 사실을 알 때 let을 사용하고, 그 외에는 가능한 const를 사용해야 한다.

### 4. Static

**4.1 인스턴스 멤버의 메모리 할당 방식**

```javascript
class Rectangle {
	constructor (width, height) {
		this.width = width;
		this.height = height;
	}

	getArea() {
		return this.width * this.height;
	}
}

const rect1 = new Rectangle(3,4);
const rect2 = new Rectangle(5,6);
```

- 메모리의 흐름

  rect1과 rect2가 생성될 때, width와 height는 각 인스턴스마다 개별적으로 메모리가 할당됨 (중복 발생)

  getArea()는 `Rectangle.prototype`에 저장되므로 모든 인스턴스가 공유하여 사용(메모리 절약 가능)

  ⇒ 인스턴스가 많아질수록 각 인스턴스의 속성들이 메모리를 차지하게 됨

**4.2 스태틱 멤버의 메모리 할당 방식**

```javascript
class Rectangle {
	constructor (width, height) {
		this.width = width;
		this.height = height;
	}

	static isRectangle(instance) {
		return instance instanceof Rectangle && 
			instance.width > 0 && instance.height > 0;
	}
}

console.log(Rectangle.isRectangle(new Rectangle(3,4))); // new Constructor는 instance임!!
```

**Rectangle.isRectangle()**은 `클래스 자체`에 저장되며, 인스턴스가 몇 개 생성되든 오직 **하나의 메모리 공간만 차지**함

모든 인스턴스가 공유하는 정보라면 스태틱 멤버를 사용함으로써 메모리 절약 가능

isRectangle()을 개별 인스턴스가 가지고 있지 않으므로, rect1.isRectangle()처럼 인스턴스로 접근 불가능

**Nest.js의 싱글톤(Single Ton) 패턴**

Nest.js에선 기본적으로 싱글톤 패턴을 따르므로, **하나의 인스턴스**만 생성되어 재사용된다. (비슷하면서도 다르다.)

```typescript
import { Injectable } from '@nestjs/common';
import { TradePrismaService } from 'apps/event/prisma/trade-prisma.service';
import { CreateTradeLogDto } from './dto/create-trade.dto';

@Injectable()
export class TradeService {
  constructor(private readonly prismaService: TradePrismaService) {}
  async createTradeLog(userId: string, createTradeLog: CreateTradeLogDto) {
    const enrichedData = createTradeLog.data.map((item) => ({
      ...item,
      old_user_id: userId,
      new_user_id: createTradeLog.new_user_id,
      ...
      중략
```

- 인스턴스를 한 번만 생성하고 앱 종료 전까지 유지함
- 같은 의존성을 필요로 하는 모든 클래스에서 같은 인스턴스를 공유

⇒ 메모리 낭비 없이 효율적으로 관리할 수 있음

**메모리 사용 방식 비교**

- 스태틱 멤버
  - 클래스 로드 시 메모리 할당 됨
  - 앱 종료 전까지 메모리에 유지됨
  - 변경이 어렵고 **`GC가 관리하지 않음`**
- Nest.js DI 싱글턴 인스턴스
  - Nest.js가 필요할 때 한 번만 생성(지연 생성 가능)
  - 필요없으면 GC에 의해 제거될 수 있음
  - useClass, useFactory처럼 더 유연하게 교체 가능
