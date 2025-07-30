---
title: FSD 아키텍쳐로 변경하면서 느낀점들
date: 2025-04-30 12:30:00 +0900
categories: [fsd, next.js]
tags: [fsd]
---

최근 사내 프로젝트의 폴더 구조를 **FSD(Feature-Sliced Design)** 아키텍처로 전환하면서 경험했던 리팩토링 과정과 고민들, 그리고 얻은 인사이트를 정리해보았다.

## 1. 폴더 구조 리팩토링

기존 구조는 다음과 같다.

```
app/
components/
hooks/
lib/
store/
types/
```

기능, UI, 상태관리 코드가 뒤섞여 있어 **역할 기반의 구조로의 전환**이 필요했다.

### 임시 네이밍 전략

리팩토링이 완료되기 전까지는 새로운 구조를 기존 코드와 구분하기 위해, 그리고 한 묶음으로 한눈에 파악하기 위해 아래와 같이 `new-` 접두어를 붙였다.

```
new-entities/
new-features/
new-shared/
new-views/
new-widgets/
```

또한 기존의 `page.tsx`는 `old-page.tsx`로 변경하고, 해당 페이지에서 사용하던 컴포넌트들은 `views/`로 이동시켜 기능별로 더 명확한 위치를 갖도록 했다.

### CSR/SSR API

**프록시 역할**을 통해 CORS 문제를 해결하기 위해서 Next.js에서 rewrites를 사용하는 중이다. 그래서 CSR과 SSR에서 사용되는 API의 형식이 다르다.

- CSR, SSR API 호출 차이 예

  ```tsx
  // 예) CSR
  import { API_HOST } from "@/shared/api-commons";

  export async function fetchUserData(token: string) {
    const userResponse = await fetch(`${API_HOST["auth"]}users/me`, {
      method: "GET",
      headers: {
        // eslint-disable-next-line @typescript-eslint/naming-convention
        "Content-Type": "application/json",
        Authorization: "Bearer " + token,
      },
    });

    if (!userResponse.ok) {
      throw new Error(userResponse.statusText);
    }

    return userResponse.json();
  }
  ```

  ```tsx
  // 예) SSR
  export async function fetchUserDataByServer(token: string) {
    const userResponse = await fetch(
      `${process.env.NEXT_PUBLIC_USER_API_URL}/users/me`,
      {
        method: "GET",
        headers: {
          // eslint-disable-next-line @typescript-eslint/naming-convention
          "Content-Type": "application/json",
          Authorization: "Bearer " + token,
        },
        cache: "no-store",
        next: { tags: ["user-update"] },
      },
    );
    return await userResponse.json();
  }
  ```

- 참고) CORS 에러가 발생하는 이유
  ### CORS 에러가 발생하는 이유
  MSA 백엔드와 Next.js 프론트엔드가 서로 다른 도메인/포트에서 실행될 때:
  - 프론트엔드: `http://localhost:3000`
  - 백엔드 API: `http://localhost:3123`, `http://api.example.com` 등
    브라우저의 **`Same-Origin Policy`**에 의해 클라이언트사이드에서 직접 다른 도메인으로 요청을 보내면 CORS 에러가 발생한다.
  ### 서버사이드에서는 왜 에러가 없을까?
  서버사이드 렌더링(SSR)에서는:
  - Node.js 서버에서 API를 호출
  - 브라우저가 아닌 **서버 환경**에서 실행
  - Same-Origin Policy가 적용되지 않음
    → 따라서 CORS 제한이 없음

## 2.1 고민1 - ui는 어디까지 순수해야 할까?

`ui`는 FSD에서 **순수 UI 요소**를 담당한다. 그런데 다음과 같은 코드처럼 `router` 같은 비즈니스 로직이 섞이기 시작하면 경계가 애매해질 수 있다. 그렇다고 router 하나 때문에 model 계층으로 분리하면 오버엔지니어링이 아닐까는 생각이 들었다.

```tsx
// widgets/home/notice/ui

"use client";

export function RouteBackModal() {
  const router = useRouter();

  return (
    <div>
      <CustomModalComponent
        blackMode
        bottomContent={""}
        isOpen={true}
        title={"알림"}
        topContent={"해당 게시글을 찾을 수 없어요! 😐"}
        onRequestClose={() => router.back("/push")}
      />
    </div>
  );
}
```

우선 아래의 코드처럼 model 계층으로 로직을 분리하면서 UI의 순수성을 지킬 수 있도록 하였다.

```tsx
// widgets/home/notice/model/RouterBackModal.ts

"use client";

import { useRouter } from "next/navigation";

export function useHandleRouter() {
  const router = useRouter();

  router.replace("/push");
}

// widgets/home/notice/ui/useHandleRouter.ts

("use client");

import { CustomModalComponent } from "@/new-shared/modal/ModalComponent";
import { useHandleRouter } from "../model/useHandleRouter";

export function RouteBackModal() {
  return (
    <div>
      <CustomModalComponent
        blackMode
        bottomContent={""}
        isOpen={true}
        title={"알림"}
        topContent={"해당 게시글을 찾을 수 없어요! 😐"}
        onRequestClose={useHandleRouter}
      />
    </div>
  );
}
```

## 2.1 고민2 - API위치는 어디에?

`home`에서 `recentOrder` API를 호출하는 상황이 있었는데, 이 API는 `home`에 있어야 할까, 아니면 `product`에 있어야 할까?

> 🤔 결국 이 API가 어떤 `도메인`에 더 밀접한지를 기준으로 결정하는 것이 맞다고 생각했다.

- **`features/`에 위치한 API**
  → 특정 기능(feature)에 **한정된 로직**에 가깝다면 여기에 위치
- **`entities/`에 위치한 API**
  → 특정 도메인(예: product, user 등)에 명확히 속한다면 이쪽으로

하지만 실제로 프로젝트를 진행하면서 `entities`의 의미가 모호해지는 경우가 종종 있었고, 이 부분은 조직 내에서 **명확한 정의와 컨벤션**이 필요하다고 느꼈다.

## 3. 마이그레이션 예시 – 잘한 점

### PR 예시

1. 중복되는 로직이나 컴포넌트들을 **shared 혹은 공통 API 함수로 분리**하면서 코드 재사용성과 유지보수성이 눈에 띄게 좋아졌다. 중복으로 사용된 코드들이 보였는데, shared나 api인 경우 그냥 fetch만 하면 되니 너무 간편해졌다.

```tsx
// 예: 공통 fetch 함수로 재사용
export const fetchGetMissionByCategory = async (category: string) => {
  return await fetch(`/api/missions?category=${category}`);
};
```

1. 마이그레이션을 하면서 관심사의 분리가 되니, 다른 커밋에서는 중복된 로직을 하나의 컴포넌트로 통합한 것도 수월하게 가능해졌다. (예) 결제 시 주소페이지 - 주소 수정/추가/삭제 ) feature/view/entity의 목적에 맞게 관심사가 잘 분리되었다.

## 4. 마무리

이번 리팩토링을 통해 FSD의 진가는 명확하게 드러났다.

FSD는 단순한 폴더 구조 변경이 아니라, **코드의 책임을 명확히 분리하고, 기능 중심으로 사고하게 만드는 개발 방식**이다.

- 기능 단위로 팀 간 협업이 쉬워지고
- 새로운 기능을 도입할 때 변경 범위를 예측할 수 있으며
- `구조적 일관성`을 통해 코드의 유지보수성과 확장성이 높아집니다.

특히, 많은 인원이 협업하거나, 서비스의 기능이 이미 많이 존재하거나, 기능이 점점 복잡해지는 서비스일수록 FSD는 점점 더 강력한 구조적 해답이 되어준다. 단 러닝커브가 존재하므로, 해당 아키텍쳐에 대한 이해도는 모두가 가져야 할 것으로 보였다.
