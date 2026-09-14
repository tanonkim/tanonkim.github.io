---
title: "Five Lines of Code 5장. 유사한 코드 융합하기"
date: 2025-04-27 19:09:00 +0900
categories: [refactoring, five-lines-of-code]
tags: [refactoring, five-lines-of-code]
---

5장 주제

- 유사 클래스 통합하기
- 조건부 산술로 구조 노출하기
- 간단한 UML클래스 다이어그램 이해하기
- 전략 패턴의 도입으로 유사 코드 통합하기
- 구현체가 하나뿐인 인터페이스 만들지 말 것으로 어지러움 정리하기

## 1. 유사한 클래스 통합하기

`||` 에 대해 각기 하나의 함수를 도입하기

예) (map[y][x].isStone() \|\| map[y][x].isFallingStone)

```typescript

// 기존 코드
function updateTile(x: number, y: number) {
  // if ((map[y][x].isStone() || map[y][x].isFallingStone())
  if (map[y][x].isStony())
    && map[y + 1][x].isAir()) {
    map[y + 1][x] = new FallingStone();
    map[y][x] = new Air();
  } 
  // else if ((map[y][x].isBox() || map[y][x].isFallingBox())
  else if (map[y][x].isBoxy())
    && map[y + 1][x].isAir()) {
    map[y + 1][x] = new FallingBox();
    map[y][x] = new Air();
  } else if (map[y][x].isFallingStone()) {
    map[y][x] = new Stone();
  } else if (map[y][x].isFallingBox()) {
    map[y][x] = new Box();
  }
}
```

메서드가 상수를 반환할 때 상수 메서드라고 한다.

경우에 따라 다른 값을 반환하는 상수 메서드를 공유하기 때문에 이 두 클래스를 합칠 수 있다.

**과정**

1. 두 개의 moveHorizontal을 동일하게 만들기
   1. if (true) {} 추가

   ![](/assets/img/2025-04-27/five-lines-of-code-5-1.png)

   b. true를 isFallingStone() === true, isFallingStone() === false로 변경

   ![](/assets/img/2025-04-27/five-lines-of-code-5-2.png)

   c. 각 moveHorizontal 본문을 복사하여 else와 함께 다른 moveHorizontal에 붙여넣기

   ![](/assets/img/2025-04-27/five-lines-of-code-5-3.png)

2. 상수 메서드 isFallingStone만 달라서 falling 필드 도입하고, 생성자에서 그 값을 할당

   ![](/assets/img/2025-04-27/five-lines-of-code-5-4.png)

3. isFallingStone을 변경해서 새로운 falling 필드를 반환

   ![](/assets/img/2025-04-27/five-lines-of-code-5-5.png)

4. 컴파일로 문제 유무 확인
5. 각 클래스에 대해 falling의 기본값을 매개변수로 받도록 하기

   ![](/assets/img/2025-04-27/five-lines-of-code-5-6.png)

   이 때 컴파일러 오류 보고 인자값 전달

   ![](/assets/img/2025-04-27/five-lines-of-code-5-7.png)

6. 통합 중인 클래스 중 하나를 제외한 모든 클래스 삭제 + 컴파일 오류를 남아있는 클래스로 바꿔서 수정

   ![](/assets/img/2025-04-27/five-lines-of-code-5-8.png)

7. 여전히 else문이 존재하는게 문제

   Stone 클래스에 있던 bool 인자를 enum으로 추출 (가독성을 위함)

   ![](/assets/img/2025-04-27/five-lines-of-code-5-9.png)

   ![](/assets/img/2025-04-27/five-lines-of-code-5-10.png)

8. 클래스 타입 코드로 대체

```typescript
interface FallingState {
  isFalling(): boolean;
	isResting(): boolean;
}
class Falling implements FallingState {
  isFalling() { return true; }
  isResting() {return false;}
  }
}
class Resting implements FallingState {
  isFalling() { return false; }
  isResting() {return true;}
  }
}
```

![](/assets/img/2025-04-27/five-lines-of-code-5-11.png)

![](/assets/img/2025-04-27/five-lines-of-code-5-12.png)

## 1.1 [리팩터링 패턴] 유사 클래스 통합

### 설명

- 상수 메서드를 공통으로 가진 두 개 이상의 클래스에서 이 상수 메서드가 클래스에 따라 다른 값을 반환할 때마다 사용
- 일련의 상수 메서드 집합을 기준(basis)이라고 한다.
- 상수 메서드가 두개일 때 두개의 접접을 가진 기준이라고 한다.

### 절차

1. 첫 번째 단계는 모든 비기준 메서드를 동일하게 만든다. 이런 메서드에 각각 다음 수행
   1. 각 메서드 버전 본문의 기존 코드 주위에 if(true) {} 추가
   2. true 를 모든 기본 메서드를 호출하여 그 결과를 상수 값과 비교하는 표현식으로 바꾼다.
   3. 각 버전의 본문을 복사하고 else와 함께 다른 모든 버전에 붙여 넣습니다.~
2. 기준만 다르므로 두 번째 단계에서는 기준 메서드에 각 메서드에 대한 필드를 도입, 생성자에서 상수를 할당하는 것으로 시작
3. 상수 대신 도입한 필드를 반환하도록 메서드 변경
4. 문제가 없는지 확인하기 위해 컴파일
5. 각 클래스에 대해 한 번에 하나의 필드씩 수행
   1. 필드의 기본값을 매개변수로 지정
   2. 컴파일러 오류를 살펴보고 기본값을 인자로 전달
6. 모든 클래스가 동일하면 하나를 제외한 모두를 삭제, 컴파일러 오류 수정
