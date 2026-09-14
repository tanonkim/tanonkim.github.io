---
title: "Next.js 섹션6. 스트리밍과 에러 핸들링"
date: 2025-11-07 21:44:00 +0900
categories: [next.js, nextjs-lecture]
tags: [next.js, react]
---

### 6.1 스트리밍이란

넷플릭스, 와챠 → 큰 용량의 동영상을 아주 빠른 속도로 시청하게 함

스트리밍 : 클라이언트 ← 큰 데이터 ← 서버

HTML 데이터를 쪼개서 연속적으로 클라이언트에게 전송하는 방식

클라이언트 : 지금까지 받은 데이터 접근 가능 → 사용자에게 긴 로딩 없이 접근 가능하게 함

1. 스트리밍 : 뭐라도 빠르게 보여줄  수 있음

2. **데이터 느리면 대체 UI 사용**
   1. 빠르게 렌더링되는 컴포넌트 전달 / 임시 페이지
   2. → 비동기 컴포넌트 전달

   ![](/assets/img/2025-11-07/nextjs-section-6-1.png)

   스트리밍 적용 시 즉시 렌더링하는 결과 바로 보여줌, 비동기 부분은 대체 UI 적용됨

**Next.js는 HTML 스트리밍을 자체 제공함**

- `Dynamic Page`에 자주 사용됨
  - 다이나믹 페이지는 풀라우트 캐시에는 적용되지 않음 (Static 페이지만 가능)
  - Next.js에서 자체 제공됨

  ![](/assets/img/2025-11-07/nextjs-section-6-2.png)

1. 빠르게 보여질 수 있는 부분은 먼저 보여줌(대체 UI등)
2. 오래 걸리는 컴포넌트의 렌더링을 사용자가 좀 더 좋은 환경에서 기다릴 수 있도록 함

### 6.2 스트리밍 1. 페이지 스트리밍 적용하기

page.tsx가 로딩중일 때 loading.tsx가 실행됨

\*Static 보다는 Dynamic 페이지에 더 유용함

#### loading.tsx

사용예시

```typescript
// loading.tsx // 대체 UI

export default function Loading() {
    return <div>Loading...</div>;
}
```

주의점

1. 해당 경로 아래의 모든 page에 적용됨

2. `비동기 컴포넌트`에만 적용됨(**async**)

   ```typescript
   export default async function Page({
       searchParams,
   }: {
       searchParams: {
           q?: string;
       };
   }) {
       await delay(3500);
       const response = await fetch(
           `${process.env.NEXT_PUBLIC_SERVER_URL}/book/search?q=${searchParams.q}`,
           { cache: 'force-cache' } // 데이터 캐시 최대한 활용
       );
   ```

3. page 컴포넌트에만 적용이 됨

   layout, 기타 등등에는 적용이 안됨

   → layout, 기타 등등에 적용하고 싶으면 `suspense`를 사용

4. 쿼리 스트링이 변경될 때는 트리거되지 않음

   http://localhost:3000/search?q=정환

   http://localhost:3000/search?q=자바 → loading이 적용안됨 (페이지가 변경된게 아님)

   ⇒ `suspense` 사용

### 6.3 스트리밍 2. 컴포넌트 스트리밍 적용하기

Suspense key : 언제 바뀔때 Suspense를 하겠습니까?

React Suspense : 내부 컴포넌트 로딩 이후에는 새롭게 로딩 상태로 돌아가지 않음 → `Key`값을 변경

```typescript
// 구조
<Suspense key={searchParams.q} fallback={<div>Loading</div>}>
		<AsyncComponent/>
</Suspense>
```

하나의 페이지에서 비동기 컴포넌트를 여러개 사용될 때 진가를 발휘함

- Static인 index를 `Dynamic` page로 변경

  ```typescript
  export default async function Home() {
      return (
          <div className={style.container}>
              <section>
                  <h3>지금 추천하는 도서</h3>
                  <Suspense fallback={<div>도서를 불러오는 중입니다...</div>}>
                      <RecoBooks />
                  </Suspense>
              </section>
              <section>
                  <h3>등록된 모든 도서</h3>
                  <Suspense fallback={<div>도서를 불러오는 중입니다...</div>}>
                      <AllBooks />
                  </Suspense>
              </section>
          </div>
      );
  }
  ```

  완료되는 순서대로 로딩이 됨

장점 : 많은 곳에 범용적으로 사용 가능, loading.tsx보다 많이 사용됨

### 6.4 스켈레톤 UI 적용하기

사용자 : 대략 이렇게 생긴 컨텐츠가 나타나겠구나

[`React Loading Skeleton`](https://www.npmjs.com/package/react-loading-skeleton)

```typescript
import style from './book-item-skeleton.module.css';

export default function BookItemSkeleton() {
    return (
        <div className={style.container}>
            <div className={style.cover_img}></div>
            <div className={style.info_container}>
                <div className={style.title}></div>
                <div className={style.subtitle}></div>
                <br />
                <div className={style.author}></div>
            </div>
        </div>
    );
}

```

```typescript
export default function Page({
    searchParams,
}: {
    searchParams: {
        q?: string;
    };
}) {
    return (
        <Suspense
            key={searchParams.q}
            fallback={
                <>
                    <BookListSkeleton count={3} />
                </>
            }
        >
            {/* streaming 되도록 설정됨 */}
            <SearchResult q={searchParams.q || ''} />
        </Suspense>
    );
}
```

### 6.5 에러 핸들링

서버 종료 시 fetch 요청이 실패됨

#### error.tsx

reset() : 브라우저에서 다시 화면만 렌더링

- 백엔드 서버오류가 발생 시 백엔드를 다시 호출하지는 않음

→ 해결방안

1. 브라우저 강제 refresh

   클라이언트에 저장된 State 저장된 것들이 싹 날라감

2. router.refresh() + reset() + startTransition

   하위 모든 경로까지 영향 미침

   - router.refresh() :

   ```typescript
   // 위치 : page.tsx의 동일 선상 위치
   'use client';

   import { useRouter } from 'next/navigation';
   import { startTransition, useEffect } from 'react';

   export default function Error({ error, reset }: { error: Error; reset: () => void }) {
       const router = useRouter();

       useEffect(() => {
           console.error(error.message);
       }, [error]);

       return (
           <div>
               <h3>오류가 발생했습니다</h3>
               <button
                   onClick={() => {
                       // React 18, update the state without blocking the UI.
                       startTransition(() => {
                           router.refresh(); // 현재 페이지에 필요한 서버 컴포넌트 다시 실행, (비동기적으로 실행됨) === RSC Payload를 다시 달라!
                           reset(); // 에러 상태 초기화, 컴포넌트들을 다시 렌더링
                       });
                   }}
               >
                   다시시도
               </button>
           </div>
       );
   }

   ```

(error 파일과 같은 경로 선상의 layout까지만 렌더링이 됨 ⇒ 이게 싫으면 각 그룹 안으로 엮으면 됨)

만약 에러 발생 시 app폴더의 layout.tsx까지만 렌더링이 됨
