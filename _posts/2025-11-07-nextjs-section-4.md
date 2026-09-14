---
title: "Next.js 섹션4. 데이터 페칭"
date: 2025-11-07 21:42:00 +0900
categories: [next.js, nextjs-lecture]
tags: [next.js, react]
---

{% raw %}
### 4.1 데이터 페칭

기존 Page Router 데이터 요청

1. SSR

   getServerSideProps() {} ⇒ 최상단에 데이터 페칭이므로 Props로 전달해야 함

2. SSG

   getStaticProps() {}

3. Dynamic SSG

   getStaticPaths() {paths:[…], fallback:…}

```typescript

// 서버 컴포넌트는 비동기로 사용해도 됨!!
export async function Page(props){
	// 기존의 getServerSideProps, getStaticProps를 대체
	const data = await fetch('...') 
	
	return <div></div>
}
```

`Fetching Data where it’s needed`

데이터가 필요한 곳에서 직접 불러와라

![](/assets/img/2025-11-07/nextjs-section-4-1.png)

```typescript
import BookItem from '@/components/book-item';
import style from './page.module.css';
import { BookData } from '@/types';

async function AllBooks() {
    const response = await fetch(`${process.env.NEXT_PUBLIC_SERVER_URL}/book`);
    if (!response.ok) {
        return <div>오류가 발생했습니다.</div>;
    }

    const allBooks: BookData[] = await response.json();

    return (
        <div>
            {allBooks.map((book) => (
                <BookItem key={book.id} {...book} />
            ))}
        </div>
    );
}
async function RecoBooks() {
    const response = await fetch(`${process.env.NEXT_PUBLIC_SERVER_URL}/book/random`);
    if (!response.ok) {
        return <div>오류가 발생했습니다.</div>;
    }

    const randomBooks: BookData[] = await response.json();

    return (
        <div>
            {randomBooks.map((book) => (
                <BookItem key={book.id} {...book} />
            ))}
        </div>
    );
}

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

```typescript
import style from './page.module.css';

export default async function Page({ params }: { params: { id: string | string[] } }) {
    const response = await fetch(`${process.env.NEXT_PUBLIC_SERVER_URL}/book/${params.id}`);

    if (!response.ok) {
        return <div>오류가 발생했습니다.</div>;
    }

    const book = await response.json();

    const { id, title, subTitle, description, author, publisher, coverImgUrl } = book;

    return (
        <div className={style.container}>
            <div
                className={style.cover_img_container}
                style={{ backgroundImage: `url('${coverImgUrl}')` }}
            >
                <img src={coverImgUrl} />
            </div>
            <div className={style.title}>{title}</div>
            <div className={style.subTitle}>{subTitle}</div>
            <div className={style.author}>
                {author} | {publisher}
            </div>
            <div className={style.description}>{description}</div>
        </div>
    );
}

```

### 4.2 데이터 캐시

fetch 메서드를 활용해 불러온 데이터를 Next 서버에서 보관하는 기능

→ 불필요한 데이터 요청 수를 줄여 웹서비스 성능을 크게 개선

```typescript
const response = await fetch('url', { cache : "force-cache"})
```

cache는 오직 `fetch` 메서드에서만 활용이 가능

#### 옵션

- { cache : “no-store” }

  데이터 페칭 결과를 저장X

  캐싱을 아예 하지 않도록 설정

  ```typescript
      const response = await fetch(`${process.env.NEXT_PUBLIC_SERVER_URL}/book`, {
          cache: 'no-store',
      });
      if (!response.ok) {
          return <div>오류가 발생했습니다.</div>;
      }

      const allBooks: BookData[] = await response.json();
  ```

  ![](/assets/img/2025-11-07/nextjs-section-4-2.png)

  ![](/assets/img/2025-11-07/nextjs-section-4-3.png)

  ~14 : 기본은 무조건 캐싱

  15 : 기본은 no-store

  ```javascript
  /** @type {import('next').NextConfig} */
  const nextConfig = {
      // Url Fetch 시 log 발생
      logging: {
          fetches: {
              fullUrl: true,
          },
      },
  };

  export default nextConfig;

  ```

- { cache : “force-cache” }

  데이터 페칭 결과를 무조건 저장

  캐싱을 하도록 설정

  ```typescript
      const response = await fetch(`${process.env.NEXT_PUBLIC_SERVER_URL}/book`, {
           cache: 'force-cache',
      });
      if (!response.ok) {
          return <div>오류가 발생했습니다.</div>;
      }

      const allBooks: BookData[] = await response.json();
  ```

  Miss → Set → (Cache) Hit → Cache Hit → Cache Hit …..

  ![](/assets/img/2025-11-07/nextjs-section-4-4.png)

  ![](/assets/img/2025-11-07/nextjs-section-4-5.png)

  .next > cache 폴더에 캐시 저장함

- { next: {   revalidate: 3  }}

  특정 시간을 주기로 캐시 업데이트

  Page Router의 ISR 방식과 유사

  ```typescript
      const response = await fetch(`${process.env.NEXT_PUBLIC_SERVER_URL}/book/random`, {
          next: {
              revalidate: 3,
          },
      });
      if (!response.ok) {
          return <div>오류가 발생했습니다.</div>;
      }

      const randomBooks: BookData[] = await response.json();

  ```

  ![](/assets/img/2025-11-07/nextjs-section-4-6.png)

  stale 순간에는 캐시 데이터 전달함

- { next : { tags : [’a’] } }

  요청이 왔을 때만 데이터 최신화함

  Page Router에서 On-Demand ISR 과 같은 방식

### 4.3 리퀘스트 메모이제이션

\*`Request Memoization`과 `Data Cache`는 전혀 다름

**`Request Memoization : 요청을 기억한다`**

⇒ 하나의 페이지에서 렌더링 할 때

중복된 요청을 다시는 발생하지 않도록 데이터 Fetching을 `최적화`

\*\* 페이지 렌더링 종료 시 자동 소멸 → 새로고침 시(새로운 접속 요청 시) API 요청함

![](/assets/img/2025-11-07/nextjs-section-4-7.png)

![](/assets/img/2025-11-07/nextjs-section-4-8.png)

Miss → Skip(데이터가 없기 때문) → Data Set → Cache Hit

데이터 캐시 : 백엔드 서버로 부터 불러온 데이터를 영구적으로 보관하기 위해 사용

- 서버 가동중에는 영구적으로 보관

Footer 컴포넌트에서 book api는 book 에서 동일한 API를 중복 호출함

→ Request Memoization이 사용됨

```typescript
async function Footer() {
    const response = await fetch(`${process.env.NEXT_PUBLIC_SERVER_URL}/book`);
    if (!response.ok) {
        return <footer>제작 @winterlood</footer>;
    }

    const books: BookData[] = await response.json();
    const bookCount = books.length;

    return (
        <footer>
            <div>제작 @winterlood</div>
            <div>{bookCount}개의 도서가 등록되어 있습니다</div>
        </footer>
    );
}
```

![](/assets/img/2025-11-07/nextjs-section-4-9.png)

페칭 로깅

```javascript
// next.config.js
  module.exports = {
    logging: {
      fetches: {
        fullUrl: true,
          },
        },
      };
```
{% endraw %}
