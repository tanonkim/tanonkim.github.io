---
title: "Five Lines of Code 2장. 리팩터링 깊게 들여다보기"
date: 2025-04-27 19:06:00 +0900
categories: [refactoring, five-lines-of-code]
tags: [refactoring, five-lines-of-code]
---

### 2.1 가독성 및 유지보수성 향상

1. **더 나은 코드를 만드는 것**
2. **코드가 하는 일을 변경하지 않는다는 것**

**가독성**

의도를 전달하기 위한 코드의 성질

코딩 컨벤션을 지정하고 따르는 것, 주석 달기, 변수, 메서드, 클래스 및 파일 이름 지정, 공백 사용 등

```typescript
// 읽기 힘든 코드
function checkValue(str: boolean) {
    // 값 체크
    if (str !== false) {
        // 반환
        return true;
    } else;
    return str;
}

// 읽기 쉽게 작성된 코드
function isBool(bool: boolean) {
    if (bool) {
        return true;
    } else return false;
}

// 단순화한 코드
function isTrue(bool: boolean) {
    return bool;
}
```

early return : **if문 : 읽는 맥락이 함수, if else else : 함수의 문맥이 아니라 조건문의 문맥으로 변함**

**Else를 붙이면 if의 조건들을 다 기억해야 한다.**

**`유지보수성`**

얼마나 많은 후보를 조사해야 하는지를 나타내는 표현

추천 서비스를 수정했는데, 지불 시스템이 중단된다 : `시스템이 취약하다`

이 취약성의 근원은 일반적으로 전역상태(Global State)이다.

- 전역 : 우리가 고려한 범위를 벗어난 것
- 상태 : 프로그램이 실행되는 동안 변경될 수 있는 모든 것

리팩터링의 세가지 핵심

- 의도를 전달함으로써 가독성 향상
- **불변속성**의 범위제한을 통한 유지보수성 향상
  - 불변속성 예 : 이 숫자는 절대 음수일 수 없습니다. 이 파일은 확실히 존재합니다.
- 범위 밖의 코드에 영향을 주지 않고 1항, 2항을 수행

### 2.2 속도, 유연성 및 안정성 확보

리팩터링 패턴에는 구체적이고 지역적인 것, 예를 들어 변수명 변경부터 추상적이고 전역적인 것 까지 여러 수준이 있다. 변수명이 가독성을 높이거나 줄일 수 있다는 점에서 동의하지만, 코드의 품질에 가장 큰 영향을 미치는 것은 **`아키텍쳐 변경`**이다.

**2.2.1 상속보다는 컴포지션 사용**

**범위가 제한되지 않은 불변속성을 도입**하는 일반적인 방법인 **상속을 권장하지 않음**.

강하게 결합되는 커플링 자체가 코드에 문제가 되니, 약한 관계로 만드는게 낫다. ⇒ 멘토님 피드백

대부분의 리팩터링 패턴과 규칙은 구체적으로 `객체 컴포지션`을 돕기 위한 것이다.

객체 컴포지션 : 클래스의 구성요소로 사용됨 === **객체 내부에 다른 객체의 참조를 가지는 것**

예) 상속 사용

**상속의 타이트한 결합**

상속 관계에서 상위 클래스 또는 슈퍼 클래스를 변경하면 코드 손상의 위험이 존재

→ 상속을 통해서 생성된 class와 objects는 밀접하게 결합되어있다. (부모 클래스 한 줄 변경 시 자식 클래스의 오작동 위험 가능성 존재)

```typescript
interface Bird {
    hasBeak(): boolean;
    canFly(): boolean;
}

class CommonBird implements Bird {
    hasBeak(): boolean {
        return true;
    }
    canFly(): boolean {
        return true;
    }
}

class Penguin extends CommonBird {
    canFly(): boolean {
        return false;
    }
}
```

예) 컴포지션 사용

```typescript
interface Bird {
    hasBeak(): boolean;
    canFly(): boolean;
}

class CommonBird implements Bird {
    hasBeak(): boolean {
        return true;
    }
    canFly(): boolean {
        return true;
    }
}

class Penguin implements Bird {
    private bird = new CommonBird(); // 컴포지션 사용
    hasBeak() {
        return this.bird.hasBeak(); // 호출을 직접 명시적으로 전달
    }
    canFly() {
        return false;
    }
}
```

**2.2.2 수정이 아닌 `추가`로 코드를 변경**

컴포지션의 장점은 추가로 변경이 가능한 것

더 깔끔하게 코드를 결합하고, 재사용이 가능함

- 계방-폐쇄 원칙 (SOLID : close)

  확장에 대해 열려 있어야 함, 수정에 대해 닫혀 있어야 함

  참고 : 프론트엔드와 SOLID

### 2.3 리팩터링과 일상 업무

- OCP를 설명할 수 있는 타입스크립트로 정리
- SOLID
  - Solid : 변해야 한다면 이유는 하나
    - 기능 변경이 일어났을 때
