---
title: "Five Lines of Code 4장. 타입 코드 처리하기"
date: 2025-04-27 19:08:00 +0900
categories: [refactoring, five-lines-of-code]
tags: [refactoring, five-lines-of-code]
---

4장 주제

- if문에서 else문을 사용하지 말 것과 switch를 사용하지 말 것으로 이른 바인딩 제거
- 클래스로 타입 코드 대체와 클래스로의 코드 이관으로 if문 제거
- 메서드 전문화로 문제가 있는 일반성 제거하기
- 인터페이스에서만 상속받을 것으로 커플링 방지하기
- 메서드의 인라인화 및 삭제 후 컴파일을 통한 불필요한 메서드 제거

## 1. 규칙 : if문에서 else를 사용하지 말것

### 정의

프로그래밍에서 타입 검사를 할 때만 if-else를 사용할 것

### 설명

if-else 코드에서 결정이 내려지는 지점을 고정한다

→ if-else 이외의 위치에서 다른 변형작업을 할 수 없기에 유연성이 떨어진다.

if-else문은 데이터의 입출력에 직접 연결되어야 하며, 애플리케이션이 나머지 부분과는 분리되어야 한다.

if문은 검사로 간주하고, if-else문은 의사결정으로 간주한다

if절은 early-return을 시키므로 이 규칙은 if-else문을 대상으로 한다.

### 스멜

`early binding` : if-else는 의사결정 동작이 컴파일 시 처리되어 애플리케이션에 고정된다 ↔ `late binding`

early binding은 if문을 수정해야만 변경할 수 있기에 추가/변경이 어렵다

### 의도

제어 흐름을 if절이 아닌 객체(클래스)가 담당하도록 한다

인터페이스를 사용해 인스턴스화 하는 클래스에 따라 실행할 코드를 결정할 수 있다.

### 규칙적용

1. 임시 명칭으로 Input2라는 새로운 인터페이스 도입하기

   ![](/assets/img/2025-04-27/five-lines-of-code-4-1.png)

2. 4개의 열거값에 해당하는 클래스 4개 생성, 클래스에 해당하는 메서드를 제외한 모든 메서드는 false 반환

   ![](/assets/img/2025-04-27/five-lines-of-code-4-2.png)

3. 열거형의 이름을 RawInput으로 변경 → 컴파일 에러 발생시키는 의도

   ![](/assets/img/2025-04-27/five-lines-of-code-4-3.png)

4. 매개변수 타입을 Input → Input2로 변경 + 새로운 메서드로 교체

```typescript
function handleInput(input: Input) {
  // if (input === Input.RIGHT)
  if (input.isRIght)
    moveHorizontal(1);
  // else if (input === Input.LEFT)
  else if (input.isLeft)
    moveHorizontal(-1);
  // else if (input === Input.DOWN)
  else if (input.isDown)
    moveVertical(1);
  // else if (input === Input.UP)
  else if (input.isUp)
    moveVertical(-1);
}
```

## 1.1 [리팩터링 패턴] 열거형 → 클래스 타입으로 코드 대체

### 설명

- 열거형을 인터페이스로 변환하고, 열거형의 값들을 클래스로 만든다.
- 각 열거형 값에 속성 추가, 해당 특정 값과 관련된 기능을 특성에 맞게 생성
  - 타입별로 기능과 데이터를 그룹화할 수 있다.
- 열거형에 새 값을 추가하는 것은 파일 해당 열거형과 관련된 **모든 로직을 확인**해야 함. → 인터페이스를 사용
  - **인터페이스를 사용하면 새로운 구현체만 추가하면 됨.**
  - 해당 클래스를 사용하기 전까지 다른 코드를 사용하지 않아도 됨
- 타입 코드는 추적이 까다로워 **열거형**으로 변환하기
  - 정수타입, 일치 비교 연산자 `===`를 지원하는 모든 타입은 타입 코드

    ![](/assets/img/2025-04-27/five-lines-of-code-4-4.png)

### 절차

1. 임시 이름의 인터페이스 도입, 인터페이스에는 열거형의 각 값에 대한 메서드가 있어야 함
2. 열거형의 각 값에 해당하는 클래스 생성, 클래스에 해당하는 메서드를 제외한 인터페이스의 모든 메서드는 `false`
3. 열거형의 이름을 다른 이름으로 변경(컴파일러가 해당 열거형에 대해 모두 오류 발생)
4. 타입을 이전 이름에서 임시 이름으로 변경하고, 일치성 검사를 새로운 메서드로 대체
5. 남아있는 열거형 값에 대한 참조 대신 새로운 클래스를 인스턴스화하여 교체
6. 오류가 더 없으면 인터페이스의 이름을 사용할 값으로 교체

예) 기존 코드

```typescript
enum TrafficLight {
    RED,
    GREEN,
    YELLOW,
}

const CYCLE = [TrafficLight.RED, TrafficLight.GREEN, TrafficLight.YELLOW];
function updateCarForLight(current: TrafficLight) {
    if (current === TrafficLight.RED) car.stop();
    else car.drive();
}
```

예) 리팩토링 적용

```typescript
interface TrafficLight {
    isRed(): boolean;
    isGreen(): boolean;
    isYellow(): boolean;
}

class Red implements TrafficLight {
    isRed(): boolean {
        return true;
    }
    isGreen(): boolean {
        return false;
    }
    isYellow(): boolean {
        return false;
    }
}
class Green implements TrafficLight {
    isRed(): boolean {
        return false;
    }
    isGreen(): boolean {
        return true;
    }
    isYellow(): boolean {
        return false;
    }
}
class Yellow implements TrafficLight {
    isRed(): boolean {
        return false;
    }
    isGreen(): boolean {
        return false;
    }
    isYellow(): boolean {
        return true;
    }
}

const CYCLE = [new Red(), new Green(), new Yellow()]; // 인스턴스화
function updateCarForLight(current: TrafficLight) {
    if (current.isRed()) car.stop();
    else car.drive();
}
```

## 1.2 [리팩터링 패턴] 클래스 코드 이관

### 설명

- 기능을 클래스로 옮기기 때문에 클래스로 타입 코드 대체 패턴의 연장선
- 항상 메서드 전체를 클래스로 옮긴다고 가정

### 절차

1. 원래 함수를 복사하여 모든 클래스로 붙여 넣기
   1. 메서드이므로 function을 제거, 컨텍스트를 this로 변경 후 사용되지 않는 매개변수 제거
   2. 메서드에 잘못된 이름이 존제하여 에러가 발생되기 때문
2. 메서드 선언 부분의 메서드 이름과 매개변수 리스트를 인터페이스에 복사하고, 원래 메서드와 다른 이름을 지정
3. 모든 클래스에서 새로운 메서드 점검

   3.1 클래스에 맞게 true/false 결정

   3.2 미리 계산할 수 있는 모든 계산을 수행

   3.3 메서드 처리가 완료되었음을 알릴 수 있는 이름으로 변경 (컴파일 오류 발생되면 안됨)

```typescript
enum Input { // enum -> interface
  UP, DOWN, LEFT, RIGHT 
}

function handleInput(input: Input) {
  if (input === Input.RIGHT) // -> input.isRight() -> this.isRight() -> true
    moveHorizontal(1);
  else if (input === Input.LEFT)
    moveHorizontal(-1);
  else if (input === Input.DOWN)
    moveVertical(1);
  else if (input === Input.UP)
    moveVertical(-1);
}
```

```typescript
interface Input {
  isRight(): boolean;
  isLeft(): boolean;
  isUp(): boolean;
  isDown(): boolean;
  handle(): void;
}

class Right implements Input {
  isRight() { return true; }
  isLeft() { return false; }
  isUp() { return false; }
  isDown() { return false; }
  handle() {
    moveHorizontal(1);
  }
}
```

## 1.3 [리팩터링 패턴] 메서드의 인라인화

### 설명

- **가독성에 도움이 안되는 메서드를 제거**
- 메서드를 인라인화를 수행할 때는 모든 호출하는 쪽에서 원래의 메서드를 제거
- 인라인 불가능한 메서드
  - 메서드로 존재하는게 가독성에 도움되는 경우
  - 낮은 수준의 연산에 의존하는 코드 : 동일한 추상화 수준에 있어야 한다는 규칙을 위함

### 절차

1. 메서드 이름을 임시로 변경 → 컴파일러 오류 발생
2. 메서드의 본문을 복사하고 매개변수를 기억
3. 컴파일러가 오류를 발생시킨 모든 곳의 호출을 복사된 코드로 교체, 인자를 매개변수에 매핑
4. 오류없이 컴파일되면 원래 메서드 삭제

예) 불필요한 메서드의 인라인화

```typescript
function hanldeInputs() {
    while (inputs.length > 0) {
        let current = inputs.pop();
        handleInput(current);
    }
}

function handleInput(input: Input) {
    input.handle();
}
```

```javascript
function hanldeInputs() {
    while (inputs.length > 0) {
        let input = inputs.pop();
        input.handle();
    }
}
```

---

## 2. 긴 if문의 리팩터링

### 일반성 제거

- 너무 일반적인 함수는 변경하기 어렵다
- 일반화를 줄이고 더 특정화한 버전의 함수를 도입하는 과정을 `메서드 전문화`라고 한다.

## 2.1 [리팩터링 패턴] 메서드 전문화

### 설명

- 코드를 일반화하고 재사용하면 할 수록 책임이 흐려지고, 다양한 위치에서 사용되면서 문제가 될 수 있다.
- 전문화된 메서드는 더 적은 위치에서 호출되어 필요성이 없어지면 더 빨리 제거할 수 있다.

### 절차

1. 전문화하려는 메서드 복제하기
2. 메서드 중 하나의 이름을 새로 사용할 이름으로 변경하고, 전문화하려는 매개변수를 제거/교체
3. 매개변수 제거에 따라 메서드를 수정해서 오류가 없도록 하기
4. 이전의 호출을 새로운 것으로 사용하도록 변경

예)

```typescript
function canMove(start: Tile, end: Tile, dx: number, dy: number) {
    return (
        dx * abs(start.x - end.x) === dy * abs(start.y - end.y) ||
        dy * abs(start.x - end.x) === dx * abs(start.y - end.y)
    );
}

if (canMove(start, end, 1, 0)) { // 룩
    // logic
}
if (canMove(start, end, 1, 1)) { // 비숍
    // logic
}
if (canMove(start, end, 1, 2)) { // 나이트
    // logic
}
```

```typescript
function rookCanMove(start: Tile, end: Tile) {
    return abs(start.x - end.x) === 0 || 0 === abs(start.y - end.y);
}

if(rookCanMove(start, end)) ...
```

## 2.2 switch를 사용하지 말 것

### 정의

default 케이스가 없고 모든 case에 반환값이 없는 경우가 아니라면 switch를 사용하지 말자

### 설명

- switch는 default로 중복 없이 여러 값을 지정가능
  - **새로운 값이 추가될 때 다른 조건이 여전히 유효한지 컴파일러로 알 수 없다.**
- default가 지원되는 언어의 경우라면 default 키워드를 사용하지 않는 것이 좋다
  - default 생략이 불가능하면 switch를 사용하지 말아야 한다.
- break를 만나기 전까지 fall-through하는 로직인데, break를 누락 시 효율성이 떨어진다.
  - 모든 케이스에 return을 지정해서 fall-through 문제를 해결할 수 있다.

### 스멜

switch는 컨텍스트, 즉 x값을 처리하는 방법에 초점이 있다.

### 의도

switch를 else-if체인으로 변환하고 다시 클래스로 만든다.

---

## 3. 코드 중복 처리

클래스로의 코드 이관이 주제.

예) 기존 코드

```typescript
function drawMap(g: CanvasRenderingContext2D) {
  for (let y = 0; y < map.length; y++) {
    for (let x = 0; x < map[y].length; x++) {
      map[y][x].color(g);
      if (!map[y][x].isAir() && !map[y][x].isPlayer())
        g.fillRect(x * TILE_SIZE, y * TILE_SIZE, TILE_SIZE, TILE_SIZE);
    }
  }
}
```

예) 리팩토링 1. 메서드 추출

```typescript
function drawMap(g: CanvasRenderingContext2D) {
    for (let y = 0; y < map.length; y++) {
        for (let x = 0; x < map[y].length; x++) {
            drawTile(x, y, g);
        }
    }
}

function drawTile(x: number, y: number, g: CanvasRenderingContext2D) {
    map[y][x].color(g);
    if (!map[y][x].isAir() && !map[y][x].isPlayer())
        g.fillRect(x * TILE_SIZE, y * TILE_SIZE, TILE_SIZE, TILE_SIZE);
}
```

예) 리팩토링 2. 클래스로의 코드 이관

Todo : 작성하기

## 3.1 인터페이스에서만 상속 받을 것

### 인터페이스 대신 추상 클래스를 사용할 수는 없을까?

인터페이스 대신 추상 클래스를 사용해서 중복되는 공통 코드를 넣을 수 있다.

But 인터페이스를 사용하면 속성 누락, 잘못된 오버라이드를 방지할 수 있다.

→ 인터페이스에서만 상속 받기

### 정의

상속은 오로지 인터페이스를 통해서만 받는다

### 설명

추상클래스는 중복을 줄일 수 있지만 단점이 더 많다

**단점**

- 코드 공유는 커플링을 유발
- 구현된 메서드 중 하나를 빈 버전으로 재정의
  - 추상 클래스에 methodA, methodB가 구현
  - 한 하위 클래스에서 methodA만, 다른 하위 클래스에선 methodB만 필요하다면
- 재정의 해야 하는 경우 재정의가 필요한지 컴파일러를 통해서 알 수 없다.

### 스멜

상속 보다는 컴포지션(합성)이 더 좋다

### 의도

상속을 받기 위해 객체를 참조함으로써 코드를 공유해야 한다.

### 클래스에 있는 코드 중복은 무엇일까?

- 중복된 코드의 변경이 필요할 때 한 곳만 변경 시 두 가지 다른 기능이 존재하게 됨

  → 분기를 조장하게 됨

## 3.2 복잡한 if 체인 구문 리팩터링

## 3.3 필요없는 코드 제거하기

- IDE가 표시해주는 사용하지 않는 함수 제거
- 인터페이스에서 사용하지 않는 함수 제거하기
  1. 컴파일을 한다. 오류가 없어야 한다.
  2. 인터페이스에서 메서드를 삭제한다.
  3. 컴파일한다.
     1. 컴파일러 오류가 발생하면 실행을 취소하고 계속 진행
     2. 그렇지 않으면 각 클래스를 살펴보고 오류 없이 해당 메서드를 삭제할 수 있는지 확인
