---
title: "Next.js 섹션5. 페이징 캐싱"
date: 2025-11-07 21:43:00 +0900
categories: [next.js, nextjs-lecture]
tags: [next.js, react]
---

### 5.1 풀 라우트 캐시

빌드 타임에 특정 페이지의 렌더링 결과를 캐싱하는 기능

페이지 /a의 결과를 풀라우트 캐시(서버에)로 저장함

![](/assets/img/2025-11-07/nextjs-section-5-1.png)

#### 페이지

**Static Page(정적인 페이지)**

⇒ `풀 라우트 캐시`가 적용됨

**ISR처럼 revalidate해야 하는 fetch도 적용이 가능함**

![](/assets/img/2025-11-07/nextjs-section-5-2.png)

**데이터 캐시, 페이지 캐시 다 모두 이루어짐**

설정 기준

Dynamic Page가 아니면 Static Page

**Dynamic Page(동적 페이지)**

특정 페이지가 접속 요청을 받을 때마다 매번 변화가 생기거나, 데이터가 달라질 경우

설정 기준

1. 캐시되지 않는 Data Fetching을 사용할 경우

   ```typescript
   async function AllBooks() {
       const response = await fetch(`${process.env.NEXT_PUBLIC_SERVER_URL}/book`, {
           cache: 'no-store',
       });
       if (!response.ok) {
           return <div>오류가 발생했습니다.</div>;
       }
   ```

2. 동적 함수(쿠키, 헤더, 쿼리 스트링)을 사용하는 컴포넌트가 있을 때

   ```typescript
   async function AllBooks() {
   		const cookie = cookies()
   		const hearerList = headers()
   		const q = searchParams.q
   ```

(동적 함수, 데이터캐시가 사용되지 않을 때도 Dynamic Page 사용됨)

![](/assets/img/2025-11-07/nextjs-section-5-3.png)

리퀘스트 메모이제이션, 데이터 캐시는 사용이 가능함!

풀 라우트 캐시만 사용이 안됨!

\*풀 라우트 캐시도 revalidate가 가능함(n초마다)

### 5.2 풀 라우트 캐시 2

\*\* 가능한 페이지는 Static Page로 만들어야 함

```typescript
export default function Searchbar() {
  const router = useRouter();
  const searchParams = useSearchParams(); // client 
  const [search, setSearch] = useState("");
```

useSearchParams : 쿼리 스트링을 받아옴 → 빌드 타임에는 존재할 수 없음

⇒ build 시 문제가 발생함 : `Suspense`

- Suspense

  미완성의 의미

  ```typescript
  export default function Layout({ children }: { children: ReactNode }) {
      return (
          <div>
              <Suspense fallback={<div>Loading...</div>}>
                  <Searchbar />
              </Suspense>
              {children}
          </div>
      );
  }

  ```

  사전 렌더링 진행 시 렌더링 바로 안하고, fallback 즉 대체 UI로 대체

  SearchBar의 비동기 작업이 완료될 때까지 fallback

  - useSearchParams : `비동기`, 브라우저에서 쿼리 스트링 받을 때 비동기가 종료됨 (클라이언트 사이드에서 쿼리 스트링을 불러왔을 때)

![](/assets/img/2025-11-07/nextjs-section-5-4.png)

---

#### Static으로 변환하기

![](/assets/img/2025-11-07/nextjs-section-5-5.png)

- RootLayout

  ```typescript
  async function Footer() {
      const response = await fetch(`${process.env.NEXT_PUBLIC_SERVER_URL}/book`, {
          cache: 'force-cache', // Dynamic Page -> Static Page
      });
  ```

- (with-searchbar)/page.tsx

  ```typescript
  async function AllBooks() {
      const response = await fetch(`${process.env.NEXT_PUBLIC_SERVER_URL}/book`, {
          // cache: 'no-store', // Dynamic Page
  	         cache: 'force-cache', // Static Page
      });
      .
      .
      .

  export default async function Home() {
      return (
          <div className={style.container}>
              <section>
                  <h3>지금 추천하는 도서</h3>
                  <RecoBooks />
              </section>
              <section>
                  <h3>등록된 모든 도서</h3>
                  <AllBooks />
              </section>
          </div>
      );
  }

  ```

⇒ .next/server/app/index.html이 존재

### 5.3 풀 라우트 캐시 3

```typescript
// (with-searchbar)/search/page.tsx

export default async function Page({
    searchParams, // Dynamic Page, 실시간으로 사용자와 소통
}: {
    searchParams: {
        q?: string;
    };
}) {
    const response = await fetch(
        `${process.env.NEXT_PUBLIC_SERVER_URL}/book/search?q=${searchParams.q}`
          { cache: 'force-cache' } // 데이터만 캐시 함, 페이지는 다시 생성
    );
```

Dynamic → Static

```typescript
// book/[id]/page.tsx

// export const dynamicParams = false; // 명시된 값, 즉 1,2,3 이외에는 404 페이지로 보냄

// 서버 측에 풀라우트 캐시로서 잘 저장됨
export function generateStaticParams() {
    return [{ id: '1' }, { id: '2' }, { id: '3' }]; // parameter에 해당하는 값을 빌드 타임에 정적으로 생성
}

export default async function Page({ params }: { params: { id: string | string[] } }) {
    const response = await fetch(`${process.env.NEXT_PUBLIC_SERVER_URL}/book/${params.id}`);

    if (!response.ok) {
        if (response.status === 404) {
            notFound();
        }
        return <div>오류가 발생했습니다.</div>;
    }

```

![](/assets/img/2025-11-07/nextjs-section-5-6.png)

id : 4번 페이지 들어가면 실시간으로 생성됨 (SSG)

풀라우트 캐시로 저장 → 다시 들어가면 빠르게 실행됨

`generateStaticParams` : 강제로 static 페이지로 고정됨

page router : getStaticPath랑 비슷

### 5.4 라우트 세그먼트 옵션

라우트 세그먼트 옵션 :

강제로 특정 페이지를 Static, Dynamic, Revalidate 등을 설정

- 별로 권장되지는 않음

특정 페이지의 유형을 강제로 Static, Dynamic 페이지로 설정

auto : 기본값, 아무것도 강제하지 않음

force-dynamic : 페이지를 강제로 Dynamic 페이지로 설정

force-static : 페이지를 강제로 Static 페이지로 설정 ()

error : 페이지를 강제로 Static 페이지로 설정 (설정하면 안되는 이유가 있다면 -> 빌드 오류로 알려줌)

```javascript
export const dynamic = 'force-static'; 
// Dynamic -> Static : 검색 결과가 나오지 않음, 이유 : Dynamic 페이지 -> Static 페이지 강제로 변환
// static으로 되면 searchParams가 빈 값으로 고정됨 -> 검색해도 값이 나오지 않음

export default async function Page({
    searchParams,
}: {
    searchParams: {
        q?: string;
    };
}) {
    const response = await fetch(
        `${process.env.NEXT_PUBLIC_SERVER_URL}/book/search?q=${searchParams.q}`,
        { cache: 'force-cache' } // 데이터 캐시 최대한 활용
    );
```

![](/assets/img/2025-11-07/nextjs-section-5-7.png)

### 5.5 클라이언트 라우터 캐시

**브라우저에 저장되는 캐시**

페이지 이동을 효율적으로 진행하기 위해 페이지 일부 데이터를 보관

![](/assets/img/2025-11-07/nextjs-section-5-8.png)

두 페이지가 공통적으로 사용하는 RootLayout를 index, search 페이지에서 중복으로 요청함

클라이언트는 HTML, JS 번들, RSC Payload를 받음

RSC : `RootLayout`, `SearchBar Layout`, 페이지 기타 등등 ⇒ 중복

⇒ RSC 중 레이아웃 데이터 클라이언트 라우터 캐시로 저장

![](/assets/img/2025-11-07/nextjs-section-5-9.png)

중복된 Layout 데이터는 브라우저에 저장

![](/assets/img/2025-11-07/nextjs-section-5-10.png)

실습) index -> search 이동 시 레이아웃은 캐시됨, 새로고침될 때는 새로 시간 데이터 받아옴

```typescript
export default function Layout({ children }: { children: ReactNode }) {
    return (
        <div>
            <div>{new Date().toLocaleString()}</div> 
            <Suspense fallback={<div>Loading...</div>}>
                <Searchbar />
            </Suspense>
            {children}
        </div>
    );
}

```
