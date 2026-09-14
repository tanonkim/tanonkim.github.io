---
title: "Performance Patterns 5 : Tree Shaking"
date: 2025-06-01 13:17:00 +0900
categories: [design-pattern, performance]
tags: [design-pattern, tree-shaking]
---

## Tree Shaking

앱의 어느 곳에서도 사용되지 않는 코드들을 코드베이스에 추가하는 경우가 있다. 이런 dead code들을 제거하여 최종 번들 크기를 줄이고, 불필요한 로딩 타임을 줄일 수 있는데, dead code들을 최종 번들중에 제거하는 과정을 `tree-shaking`이라 한다.

### 컨셉

tree-shaking은 최종 자바스크립트 번들에서 실제로 사용되지 않는 코드들을 제거하기 위해 사용된다. 이 과정이 올바르게 수행된다면 클라이언트 측에서 다운로드해야 하는 번들 크기를 줄여 로딩 속도, 구분 분석, 실행 시간을 줄일 수 있다. 대부분의 모던 앱들은 webpack, rollup, turboPack과 같은 모듈 번들러를 통해 빌드되기 때문에 tree-shaking이 자동으로 적용된다.

개발하는 앱과 내부의 의존 관계를 Abstract Syntax Tree로 볼 수 있다. 트리의 각 노드는 우리의 앱에 기능을 제공하는 의존성이라고 볼 수 있다. tree-shaking에서는 입력된 파일 모두 그래프로 처리하는데, tree-shaking은 그래프를 앱의 진입점부터 모든 실제 순회 경로를 표시하는 탐색 과정이라고 볼 수 있다. 각 노드에서 part는 각 노드에서 최상의 구문이 된다.

**사이드이펙트**

각 컴포넌트들은 심볼(자바스크립트 내 선언하고 사용가능한 모든 것, ES6 심볼 X)들을 선언할 수 있고, 참조할 수 있고, 다른 파일에 의존할 수 있다. 각각의 part(파일)은 사이드 이펙트(다른 코드들을 참조하는 것)가 있을수도, 없을수도 있다. 예를 들어 `let firstName = 'KIM'` 구문의 경우 사이드 이펙트가 없기 때문에 firstName을 참조하는 코드만 확인하면 된다. 하지만  `let firstName = getName()` 의 경우 firstName을 사용하는 코드가 없다 하더라도, getName()의 호출로 이어지는 내용 때문에 제거할 수 없다.

### Imports

ES2015의 모듈 구문(import, export)으로 정의된 모듈들만 tree-shaking이 가능하다. 모듈을 import하는 방법이 tree-shaking 가능 여부를 결정하게 된다.

tree-shaking은 앱이 시작되는 파일부터 새로운 섹션들에게 닿게 될 때까지 사이드 이펙트(다른 구문을 참조하는 것)로 관련된 트리의 각 노드 끝 부분까지 탐색한다. 이 과정이 끝나면 자바스크립트 번들에는 탐색된 부분들만 포함된다.

### Side Effects

ES6 모듈을 import하는 경우 해당 모듈은 즉시 실행된다. 이 때 참조하는 코드가 export하는 것을 참조하지 않더라도 그 코드 내부에서 전역에 무언가 영향을 줄 수 있다. 예 : 폴리필 추가, 전역 스타일시트 추가 등, 그리고 이를 side effect라고 한다.  모듈이 export하는 것을 참조하지 않고 있더라도, 위와 같은 특이 동작들로 인해 tree-shaking이 불가해지는 경우가 있다.

웹팩의 side effect 문제

[https://webpack.js.org/guides/tree-shaking/#clarifying-tree-shaking-and-sideeffects](https://webpack.js.org/guides/tree-shaking/#clarifying-tree-shaking-and-sideeffects)
