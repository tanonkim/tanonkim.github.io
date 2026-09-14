---
title: "Frontend Test 1. 테스트 패턴"
date: 2025-04-23 19:00:00 +0900
categories: [test, frontend-test]
tags: [test, frontend-test]
---

예제 코드 : [https://github.com/frontend-testing-book-kr/unittest](https://github.com/frontend-testing-book-kr/unittest), [https://github.com/frontend-testing-book-kr/nextjs](https://github.com/frontend-testing-book-kr/nextjs)

## AAA (Arrange, Act, Assert)

해당 패턴은 메서드를 Arrange, Act, Assert 세 부분으로 나누어야 한다고 제안한다.

코드 중심적 사고이며 개발자 친화적이다.

- Arrange

테스트를 설정하는데 필요한 코드만 존재한다. 객체 생성, Mock객체, 기대값 설정이 포함된다.

- Act

테스트 중인 메서드를 호출하는 단계이다.

- Assert

기대값이 충족되었는지 확인하는 단계이다.

```javascript
import { render, screen, fireEvent } from '@testing-library/react';
import Counter from './Counter';

test('should increment counter when button is clicked', () => {
  // Arrange
  render(<Counter />);

  // Act
  fireEvent.click(screen.getByText('Increment'));

  // Assert
  expect(screen.getByText('Count: 1')).toBeInTheDocument();
});
```

## Given, When, Then

시나리오 중심적 사고이며 비즈니스/비개발자 친화적이다.

- Given

테스트를 위해 준비하는 과정(변수, 입력값 정의, Mock객체 정의)

- When

테스트 코드를 실행. (메서드 호출)

- Then

테스트 검증. (리턴값 검증)

```javascript
import { render, screen, fireEvent } from '@testing-library/react';
import Counter from './Counter';

test('Given the counter is at 0, When user clicks the increment button, Then it should show 1', () => {
  // Given
  render(<Counter />);
  expect(screen.getByText('Count: 0')).toBeInTheDocument();

  // When
  fireEvent.click(screen.getByText('Increment'));

  // Then
  expect(screen.getByText('Count: 1')).toBeInTheDocument();
});
```
