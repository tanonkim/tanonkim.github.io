---
title: "React Fiber with React Profiler"
date: 2025-05-25 20:22:00 +0900
categories: [react, react-study]
tags: [react, profiler]
---

## React Fiber 흐름

React는 렌더링할 컴포넌트를 모두 Fiber 노드로 변환해서 관리한다. 각 Fiber는 일종의 Task Unit이고, 이 노드들을 통해 렌더링 과정을 분할하고 스케쥴링을 할 수 있다.

| 필드명 | 설명 |
| --- | --- |
| `type` | 컴포넌트 타입 (예: 함수, 클래스, DOM 노드) |
| `stateNode` | 해당 Fiber가 담당하는 실제 인스턴스 (DOM 노드나 클래스 인스턴스) |
| `child` | 첫 번째 자식 Fiber |
| `sibling` | 다음 형제 Fiber |
| `return` | 부모 Fiber |
| `alternate` | 이전 렌더에서의 동일 Fiber (→ 변화 비교용) |
| `pendingProps` | 변경 예정인 props |
| `memoizedProps` | 이전 렌더에서 사용된 props |
| `memoizedState` | 이전 렌더에서 사용된 state |
| `flags` | 작업 마킹 (예: 업데이트 필요, 삭제 등) |

## Profiler Tab

프로파일러 탭을 활용하면 컴포넌트의 성능을 테스트할 수 있고, 이 결과를 활용하면 어느 컴포넌트에 개선이 이루어져야 할 지 짐작할 수 있다.

### 1) 프로파일링 시작하기

처음 프로파일러 탭을 열면, "🔵" 버튼을 눌러 기록을 진행하기 전까지는 아무 내용도 확인할 수 없다. 기록을 시작하면 애플리케이션이 렌더링될 때 마다 성능 정보가 자동으로 수집된다. 기록 중에는 평소처럼 앱을 사용하면 된다. 수집을 중단하려면, "🔴" 버튼을 누른다.

### 2) Commit Chart 보는 법

리액트는 아래의 단계를 거쳐 작동한다.

1. Render 단계 : JSX 선언 또는 React.createElement()를 통해 일반 객체인 React 엘리먼트를 생성한다.
2. Reconcile 단계 : 기존의 렌더링된 실제 DOM트리와 새로 렌더링할 React 엘리먼트와 비교하여 변경점을 적용한다.
3. Commit 단계 : 새로운 DOM 엘리먼트를 브라우저 뷰에 `커밋`한다.
4. Update 단계 : props, state 변경 시 해당 컴포넌트와 하위 컴포넌트에 대해 위 과정을 반복한다.

Commit과 Update 단계를 거치면서 React 수명 주기 메서드가 호출된다.

커밋은 프로파일러 상단에 막대 차트로 표시된다. 각 막대는 커밋 별로 그룹화 된 성능 정보를 의미한다.

막대의 **색상과 높이**를 통해 **렌더링에 걸린 시간**을 비교할 수 있다. 길이가 긴 노란색 막대(🟨)가 짧은 초록색 막대(🟩)보다 오랜 시간이 걸렸음을 의미한다.

마우스로 각 막대를 클릭하거나 좌우 화살표 버튼을 클릭해 커밋 별 상세 정보를 확인할 수 있다. 선택된 커밋은 파란색(🟦) 막대로 표시된다.

![](/assets/img/2025-05-25/react-fiber-profiler-1.png)

### 3) Flame Chart 보는 법

Flame은 특정 커밋에 대한 애플리케이션 상태를 보여준다. 여기서 볼 수 있는 차트의 각 막대는 React 컴포넌트를 나타낸다.

여기서도 막대의 크기와 색상은 컴포넌트와 자식 요소까지 모두 렌더링되는데 걸린 시간을 의미한다. 너비는 컴포넌트가 최종 렌더링될 때 소요된 시간을, 색상은 현재 커밋의 일부로써 소요된 시간을 보여준다.

즉 막대의 너비가 넓을 수록 렌더링에 오랜 시간이 걸린 것이며, 막대가 노란색(🟨)이면 상대적으로 오랜 시간, 초록색(🟩)이면 짧은 시간이 소요되었다고 볼 수 있다. 만일 **회색**(⬜️)이라면 이 커밋 중에 컴포넌트가 전혀 렌더링되지 않았다는 뜻이다.

> 보안상 이미지를 생략했습니다.

**특정 컴포넌트의 업데이트**

특정 컴포넌트가 몇 번 업데이트 되었는지 알고 싶다면 플레임 차트에서 해당 컴포넌트 명을 가진 막대를 클릭하면 된다. 우측 패널의 Rendered at 영역을 참고하면 알 수 있다.

![](/assets/img/2025-05-25/react-fiber-profiler-3.png)

- **Fiber 내부 동작과의 연결**

  렌더링 시간 : Fiber 트리 순회 시간

  업데이트 원인(props/state) : Fiber 노드의 변경 감지 및 갱신

  Re-render 여부 : React가 Fiber 노드를 다시 렌더링했는지 판단

  Suspense 발생 : Fiber가 일시중단되고 fallback으로 넘어가는 시점 포착 가능

  ![](/assets/img/2025-05-25/react-fiber-profiler-4.png)

- **Router → LayoutRouterContext.Provider → HotReload**

  ![](/assets/img/2025-05-25/react-fiber-profiler-5.png)

  Fiber 연결 포인트 :

  - 각 Provider는 React의 Context 시스템으로, Context를 사용할 수 있는 경우 React는 관련된 컴포넌트 하위 트리에 있는 모든 Fiber들을 추적하여 필요 시 Rerender한다.
  - LayoutRouterContext.Provider는 내부적으로 Fiber.flags에 Update를 설정해 업데이트를 스케쥴링
  - HotReload가 등장하는건 Next.js 개발 환경이 React Fiber 작업을 중간에 개입하여 트리를 재처리한 결과

- **RedirectErrorBoundary → RedirectBoundary**

  ![](/assets/img/2025-05-25/react-fiber-profiler-6.png)

  Fiber 연결 포인트 :

  - ErrorBoundary는 Fiber 플래그로, DidCapture 플래그를 설정하여 에러를 캐치하고 fallback을 렌더
  - 이 과정에서 Fiber 트리의 하위 노드는 render 단계를 다시 거침 : Profiler에서 시간 소모가 다시 찍힘
  - Fiber 트리 내부에서 에러가 발생하면 fiber.return(부모 노드)을 따라ErrorBoundary까지 올라가며 처리됨

- TurboPackInfo, UserPreferences

  ![](/assets/img/2025-05-25/react-fiber-profiler-7.png)

  Fiber 연결 포인트 :

  - 리렌더가 발생한 시점에서 특정 컴포넌트들이 다시 그려졌다는 의미
  - Fiber는 props, state, context 등을 비교해서 shouldComponentUpdate, React.memo, useMemo, useCallback 등을 기준으로 렌더 스킵 여부를 판단
  - profiler는 이 판단이 끝나고, 실제 렌더가 발생한 Fiber 노드만 그려서 보여준다.

### 4) Ranked Chart 보는 법

단일 커밋을 렌더링에 오랜 시간 순으로 보여준다. 각 막대는 React 컴포넌트를 의미한다. 렌더링에 걸린 시간은 해당 컴포넌트와 하위 컴포넌트까지 모두 렌더링하는데 소요된 시간을 의미하기에, 일반적으로 상위를 차지한 항목이 컴포넌트 트리 상단에 위치해있는 확률이 높다.

![](/assets/img/2025-05-25/react-fiber-profiler-8.png)

플레임 차트에서와 마찬가지로, 개별 막대를 클릭하여 렌더링된 횟수와 렌더링된 원인을 파악할 수 있다.

### 의문점

Fiber는 작업을 중단하고 다음 프레임에 이어서 실행이 가능함

- 렌더링이 중간되었다가 재개되는 이유 확인 방법

Fiber는 우선순위를 기반으로 작업 순서를 정함

- 컴포넌트 B보다 C가 먼저 렌더링되는 경우를 어떻게 확인할까
