---
title: "Next.js 섹션3. App Router"
date: 2025-11-07 21:41:00 +0900
categories: [next.js, nextjs-lecture]
tags: [next.js, react]
---

### 3.1 App Router 시작하기

앱라우터 추가 사항

- React 18 신규 기능
  - 리액트 서버 컴포넌트
  - 스트리밍
- 데이터 페칭 방식 변경
- 레이아웃 설정 방식 변경
- 페이지 라우팅 설정 방식 변경

크게 변경되지 않는 사항

- 네비게이팅
- 프리페칭
- 사전렌더링

npx create-next-app@rc unlock-event-next-improved

APP 라우터 → `src/app`이라는 폴더가  존재

layout.tsx 존재

### 3.2 페이지 라우팅 설정하기

```
|-app
|——page.tsx —> /
|——search
|—————page.tsx —> /search
|—————[id]
|————————page.tsx —> /search/1
|—————[…id]
|————————page.tsx —> /search/1/12/123145/123123/123123/1111
|—————[[…id]] : Optional Catch All Segment
|————————page.tsx —> /search/1/12/123145/123123/123123/1111
```

—> /search, /search/1, /search/12/3/123/123/123/14/3 모두 가능

#### QueryString 사용

그냥 Props에서 꺼내 쓰면 됨

```typescript
// localhost:3000/book?q=200
export default function Page({searchParams} : {searchParams : {q? : string}}) {
	// props를 console로 찍어봐
	reutrn <div>Hello</div>
}

// localhost:3000/book/200
export default function Page({params} : {params : {id : string | string[]}}) {
	// props를 console로 찍어봐
	reutrn <div>Hello</div>
}
```

![](/assets/img/2025-11-07/nextjs-section-3-1.png)

### 3.3 레이아웃 설정하기

#### 1. 레이아웃 설정하기

```
|-app
|——search
|————page.tsx
|————layout.tsx
```

**layout 컴포넌트가 먼저 렌더링**, → 그 다음 children으로 page가 렌더링됨

![](/assets/img/2025-11-07/nextjs-section-3-2.png)

- /search 경로에 모두 적용됨
  - /search
  - /search/setting

- 중첩 레이아웃
  - search
    - layout.tsx
    - /search/setting
      - layout.tsx

  ![](/assets/img/2025-11-07/nextjs-section-3-3.png)

```
|-app
|——search
|————page.tsx
|————layout.tsx
|————setting
|——————page.tsx
|——————layout.tsx → layout 중첩 적용
```

src/app/layout : RootLayout

```typescript
export default function RootLayout({
  children,
}: Readonly<{
  children: React.ReactNode;
}>) {
  return (
    <html lang="en">
      <body className={`${geistSans.variable} ${geistMono.variable}`}>
        {children}
      </body>
    </html>
  );
}
```

src/app/search/layout

```typescript
export default function Layout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
			<div>
				임시 서치바
        {children}
			</div>
  );
}
```

#### 2. 특정 페이지에서만 Layout 적용하기

search, index처럼 자식 페이지가 아닌 페이지 Layout 적용하기

**라우트 그룹** : `(name)` 안에 있는 그룹

경로상에는 영향을 미치지 않음

하나의 그룹으로 묶음

![](/assets/img/2025-11-07/nextjs-section-3-4.png)

### 3.4 리액트 서버 컴포넌트 이해하기

#### 리액트 서버 컴포넌트

`React Server Component` - 리액트 18부터 추가된 새로운 유형의 컴포넌트

- 오직 서버 측에서만 실행되는 컴포넌트(브라우저 실행 X)
- Next.js page Router에서는 서버 컴포넌트가 지원되지 않았음

#### 서버 컴포넌트 등장 배경

하이드레이션을 위해 JS 번들로 묶어서 후속 전달

- 전달된 리액트 컴포넌트들은 Hydration을 위해 브라우저에서 한번더 실행됨

  ![](/assets/img/2025-11-07/nextjs-section-3-5.png)

사전 렌더링 시 불필요한 번들은 제외해야 함

- 상호작용이 필요없는 컴포넌트들
- Hydration이 줄어들어 TTI가 줄어들게 됨

- → 브라우저에서 Hydration이 필요가 없어짐

  ![](/assets/img/2025-11-07/nextjs-section-3-6.png)

  초록 : 하이드레이션, 상호작용이 필요함

  - 클라이언트 컴포넌트 : `사전 렌더링 시` 한번, `Hydration 진행할 때` 한번 → **총 2번** 실행

    일반 컴포넌트들, 꼭 필요한 경우에만 사용 권장

  - 서버 컴포넌트 : 서버 측에서 사전 렌더링을 진행할 때 **딱 한번만** 실행됨

    페이지의 대부분을 서버 컴포넌트로 구성할 것을 권장

    `App router`에는 기본적으로 모든 컴포넌트가 `서버 컴포넌트`로 작동함

    ![](/assets/img/2025-11-07/nextjs-section-3-7.png)

    - 서버클라이언트 → 보안/server fetch 관련 작업을 함

#### 클라이언트 컴포넌트

useEffect : 브라우저에서 실행됨

→ 최상단에 `“use client”`

#### 클라이언트 VS 서버 컴포넌트

**상호작용**이 있어야 하면 클라이언트, 그 이외는 전체 서버 컴포넌트로 작성

상호작용 : 입력, 클릭 등…

(\*링크는 HTML 고유 기능)

초록 : 클라이언트 컴포넌트

레드 : 서버 컴포넌트

![](/assets/img/2025-11-07/nextjs-section-3-8.png)

```typescript
// src/app/(with-searchbar)/searchbar.tsx (컴포넌트 파일)

'use client';

import { useState } from 'react';

export default function SearchBar() {
    const [search, setSearch] = useState('');

    const onChangeSearch = (e: React.ChangeEvent<HTMLInputElement>) => {
        setSearch(e.target.value);
    };

    return (
        <div>
            <input value={search} onChange={onChangeSearch} />
            <button>검색</button>
        </div>
    );
}

// layout.tsx
import SearchBar from './searchbar';

export default function Layout({ children }: { children: React.ReactNode }) {
    return (
        <div>
            <SearchBar />
            <div>{children}</div>
        </div>
    );
}

```

### 3.5 리액트 서버 컴포넌트 주의사항

1. 서버 컴포넌트에는 브라우저에서 실행될 코드가 포함되면 안됨

   예) `useXXX`, `onClick` 등, 브라우저에서 실행되는 기능의 라이브러리

2. 클라이언트 컴포넌트는 클라이언트에서만 실행되지 않음

   클라이언트 컴포넌트

   사전 렌더링을 위해 서버에서 1번

   하이드레이션을 위해 브라우저에서 1번

3. 클라이언트 컴포넌트에서 서버 컴포넌트를 import할 수 없음

   ![](/assets/img/2025-11-07/nextjs-section-3-9.png)

   서버 컴포넌트는 서버에서만 딱 한번 실행

   서버에서는 두 코드 모두 존재

   브라우저 Hydration : 서버 컴포넌트는 존재하지 않음(서버에서만 실행됨) → 에러

   참고, client 컴포넌트로 하면 JS 번들이 커짐

   **예**

   ```typescript
   // Home -> Client Component -> Server Component

   => 서버 컴포넌트는 자동으로 Client 컴포넌트로 실행됨
   서버 컴포넌트가 Client 컴포넌트로 실행됨
   -> Next.js가 자동으로 Client 컴포넌트로 변환함

   import ClientComponent from './client-component';
   import styles from './page.module.css';

   // page.tsx
   export default function Home() {
       return (
           <div className={styles.page}>
               인덱스 페이지
               <ClientComponent />
           </div>
       );
   }

   // client-component.tsx
   'use client';

   import ServerComponent from './server-component';

   export default function ClientComponent() {
       console.log('클라이언트 컴포넌트 실행됨');

       return <ServerComponent />;
   }

   // server-component.tsx
   export default function ServerComponent() {
       console.log('서버 컴포넌트 실행됨');

       return <div></div>;
   }

   ```

   **해결방안**

   ```typescript
   // children으로 서버 컴포넌트를 전달함!!
   "use client"

   // page.tsx
   import ClientComponent from './client-component';
   import styles from './page.module.css';
   import ServerComponent from './server-component';

   // 기본적으로 Server Component
   export default function Home() {
       // console.log('Home 컴포넌트 실행');

       return (
           <div className={styles.page}>
               인덱스 페이지
               <ClientComponent>
                   <ServerComponent />
               </ClientComponent>
           </div>
       );
   }

   // client-component.tsx
   'use client';

   import { ReactNode } from 'react';

   export default function ClientComponent({ children }: { children: ReactNode }) {
       console.log('클라이언트 컴포넌트 실행됨');

       return <div>{children}</div>;
   }

   ```

4. 서버 컴포넌트에서 클라이언트에게 **직렬화**되지 않는 props는 전달이 안됨

   직렬화 : Serialization

   객체 등 복잡한 구조의 데이터를 네트워크 상으로 전송하기 위해 아주 단순한 형태로 변환

   객체 배열 → 문자열, Byte로 변환

   ![](/assets/img/2025-11-07/nextjs-section-3-10.png)

   \*JS 함수의 직렬화는 불가능함

   서버 컴포넌트 → 클라이언트 컴포넌트로 Props로 전달 안됨

   페이지 구성하는 모든 컴포넌트 → 서버 컴포넌트만 따로 실행(RSC Payload) → 완성된 HTML 페이지

### 3.6 네비게이팅

페이지 이동 : 페이지 이동은 Client Side Rendering 방식으로 처리됨 (Page Router과 동일)

![](/assets/img/2025-11-07/nextjs-section-3-11.png)

페이지 이동 전 미리 Pre-Fetch 함

Server Component 추가됨

`RSC Payload`(서버 컴포넌트의 결과물_정보)도 함께 전달함 (React Server Component 순수한 데이터)

![](/assets/img/2025-11-07/nextjs-section-3-12.png)

RSC 값

index → Search(서버 컴포넌트로만 되어있음)

![](/assets/img/2025-11-07/nextjs-section-3-13.png)

클라이언트 컴포넌트 : 자바스크립트 번들 + RSC payload

페이지에 클라이언트 렌더링 하는 예시 보여줌

서버 컴포넌트는 RSC payload 전달

#### Prefetch

개발모드에서는 동작하지 않음 ⇒ build 후 start

index에서 search, book/1이 미리 prefetch됨

**index 페이지는 정적(static)이라서 JS 번들 + RSC까지 모두 호출**

**book/1은 동적 페이지 → JS 번들은 생략, RSC만 호출**

동적인 페이지는 SSR처럼 동작함(브라우저의 요청을 받을 때마다 생성됨)

App router는 Static or Dynamic 둘 중 하나

Static : RSC + JS

Dynamic : RSC
