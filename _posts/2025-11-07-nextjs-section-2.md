---
title: "Next.js 섹션2. Page Router"
date: 2025-11-07 21:40:00 +0900
categories: [next.js, nextjs-lecture]
tags: [next.js, react]
---

{% raw %}
### 2.1 Page Router

#### Page Router

- 가장 안정적인 라우터
- 리액트처럼 라우팅 기능 제공

```
|pages
|——index.js. —> ~/
|——about.js —> ~/about
|——item(폴더)
|————index.js
|————[id].js —> ~/item/1, ~/item/2 (동적경로)
```

명령어

`npx create-next-app@14 page-router`

#### 폴더

public : 정적 파일 저장

pages : 페이지를 담당할 컴포넌트

styles : 스타일 관련

페이지의 역할 X,

공통적인 레이아웃, 데이터 등을 다루기 위함

`_app.tsx`

→ 모든 페이지들의 부모 컴포넌트 역할

![](/assets/img/2025-11-07/nextjs-section-2-1.png)

```typescript
export default function App({ Component, pageProps }: AppProps) {
  return <Component {...pageProps} />;
}

// Component : 페이지 역할을 하는 컴포넌트
// pageProps : 페이지 역할을 하는 컴포넌트에 Props를 객체로 저장
// 예) Global Header
```

`_document.tsx`

→ 모든 페이지에 적용되어야 하는 공통 HTML 코드 설정하는 Component

리액트 index.html과 비슷

메타, 구글애널리틱스, 폰트 등등

language

`next.config.mjs`

→ Next 앱의 설정

reactStrictMode : 개발모드 실행 시 컴포넌트 두번 실행(콘솔에 두번 찍히는 것 등) → False

### 2.2 페이지 라우팅 설정하기

#### 폴더구조

```
|pages
|——index.js. —> ~/
|——about.js —> ~/about
|——search
|————index.tsx —> ~/search
|————setting.tsx —> ~/search/setting
```

`or`

```
|pages
|——index.js. —> ~/
|——about.js —> ~/about
|——search
|————index.tsx —> ~/search
|————setting
|——————index.tsx —> ~/search/setting
```

참고

page라우터에선 useRouter : next/router 만 사용

app라우터에선 useRouter : next/navigation 만 사용

#### 쿼리스트링 콘솔 찍기

```typescript
import { useRouter } from 'next/router';

export default function Page() {
    const router = useRouter();
    console.log(router);

    return <>Search</>;
}

// console이 두번 찍힘
// 컴포넌트를 다시 불러오기 때문
	// 1. query 객체 없는 router 객체
	// 2. query 객체 있는 router 객체
```

#### 동적경로 만들기

```typescript
// src/pages/book/[id].tsx
// /book/13, book/1 동적 대응 가능

import { useRouter } from 'next/router';

export default function Page() {
    const router = useRouter();

    const { id } = router.query;

    return <h1>book {id}</h1>;
}

```

- 만약 book/1/23/14 등 모든 경로 catch
  - `Catch All Segment` 라고 부른다
  - book 의 경로는 404,

```typescript
// src/pages/book/[...id].tsx
// 이런 구조를 Catch All Segment

import { useRouter } from 'next/router';

export default function Page() {
    const router = useRouter();

		// 배열 형태로 전달됨
    const { id } = router.query;

    return <h1>book {id}</h1>;
}

```

- / 뒤에 나오든가 안나오든가 모든 범용페이지
  - `[[...id]].tsx`
    - Optional Catch All Segment

#### 404 페이지

pages 밑에 404 페이지 생성

### 2.3 Navigating

#### Link

`a 태그` : client side 방식이 아님 → 느리다

매번 페이지를 새로고침으로 요청

⇒ `Link` 컴포넌트 사용하기

속도가 빠름

#### Programmatic Navigation

특정 버튼이 클릭 특정 조건이 만족되었을 때 함수 내부에서 페이지 이동하는 방식

onClickButton : `ClientSide` 방식으로 이동시킴

```typescript

export default function App({ Component, pageProps }: AppProps) {
    const router = useRouter();

    const onClickButton = () => {
        router.push('/test'); 
        //replace : 뒤로가기 방지하며 페이지 이동
    };

    return (
        <>
            <header>
                <Link href={'/'}>index</Link>
                &nbsp;
                <Link href={'/search'}>search</Link>
                &nbsp;
                <Link href={'/book/1'}>book/1</Link>
                <div>
                    <button onClick={onClickButton}>/test 페이지로 이동</button>
                </div>
            </header>
            <Component {...pageProps} />
        </>
    );
}
```

### 2.4 Pre-Fetching

Pre-Fetching : 페이지를 미리 불러온다

빠른 페이지 이동을 위해 제공

![](/assets/img/2025-11-07/nextjs-section-2-2.png)

클라이언트 사이드로 처리하는데, 왜 pre-Fetching이 필요하나?

→ JS 번들 : **현재 페이지**에 필요한 JS 번들만 전달된다.

예) /search 접속 요청 → Search JS Bundle만 전송

모든 번들 파일을 전달 시 `Hydration`이 느려짐 ⇒ TTI가 늦어짐

![](/assets/img/2025-11-07/nextjs-section-2-3.png)

**해당 페이지 내에서 이동할 수 있는 모든 페이지를 Pre-Fetch함**

→ 연결된 모든 페이지의 JS 번들을 불러옴

**결과 : 페이지 이동 요청 후 JS 번들 요청이 아닌, 페이지 이동 전 JS 번들 요청 완료**

(현재 페이지에서 이동할 수 있는 모든 페이지의 JS 코드를 부름)

![](/assets/img/2025-11-07/nextjs-section-2-4.png)

참고 : Dev 모드에선 pre-fetching이 실행이 안됨

→ build 후 yarn start로 확인

![](/assets/img/2025-11-07/nextjs-section-2-5.png)

index 페이지인데, book과 search를 미리 요청함

그리고 book으로 이동하면 어떠한 요청도 하지 않음 (JS 번들이 미리 요청되었으니)

예외)

캐시가 만료된다면 다시 요청을 보내긴 함

**/test는 prefetching이 되지 않음**

⇒ Programatic하게 페이지 이동하기 때문 (Link가 아닌, onClick으로 이동)

해결 : mount될 때 prefetch를 같이 써주자(useEffect 안에)

```typescript
import '@/styles/globals.css';
import type { AppProps } from 'next/app';
import Link from 'next/link';
import { useRouter } from 'next/router';
import { useEffect } from 'react';

export default function App({ Component, pageProps }: AppProps) {
    const router = useRouter();

    const onClickButton = () => {
        router.push('/test');
    };

		// prefetch 설정
    useEffect(() => {
        router.prefetch('/test');
    }, []);

    return (
        <>
            <header>
                <Link href={'/'}>index</Link>
                &nbsp;
                <Link href={'/search'}>search</Link>
                &nbsp;
								// prefetch 설정
                <Link href={'/book/1'}>book/1</Link>
                // <Link href={'/book/1'} prefetch={false}>book/1</Link> // -> prefetch 해제

                <div>
                    <button onClick={onClickButton}>/test 페이지로 이동</button>
                </div>
            </header>
            <Component {...pageProps} />
        </>
    );
}

```

### 2.5 API Routes

/api/file-name

페이지가 아닌 api route로 자동 설정됨

```typescript
// Next.js API route support: https://nextjs.org/docs/api-routes/introduction
import type { NextApiRequest, NextApiResponse } from "next";

type Data = {
  name: string;
};

export default function handler(
  req: NextApiRequest,
  res: NextApiResponse<Data>,
) {
  res.status(200).json({ name: "John Doe" });
}

```

### 2.6 스타일링

global css는 App 컴포넌트에서만 호출이 가능함

css module

import style from “./index.module.css”

### 2.7 글로벌 레이아웃

components/global-layout.tsx

### 2.8 페이지별 레이아웃 설정하기

```typescript
// src/components/searchable-layout.tsx
import { ReactNode } from 'react';

export default function SearchableLayout({ children }: { children: ReactNode }) {
    return (
    <div>
	    <div>임시서치바</div>
	    {chlidren}
    </div>;
    )
}

// src/pages/_app.tsx
type NextPageWithLayout = NextPage & { // NextPage 컴포넌트 타입 + getLayout 타입
    getLayout?: (page: ReactNode) => ReactNode;
};

export default function App({
    Component,
    pageProps,
}: AppProps & { // 타입 확장
    Component: NextPageWithLayout;
}) {
    const getLayout = Component.getLayout ?? ((page: ReactNode) => page);

    return <GlobalLayout>{getLayout(<Component {...pageProps} />)}</GlobalLayout>;
}

// src/page/index.tsx

import { ReactNode } from 'react';
import style from './index.module.css';
import SearchableLayout from '@/components/searchable-layout';

export default function Home() {
    return <h1 className={style.h1}>인덱스</h1>;
}

// 함수도 객체이기 때문에, 메서드 추가가 가능함 (사용하고싶은 Page에 적용하기, index, search 등등)
Home.getLayout = (page: ReactNode) => {
    return <SearchableLayout>{page}</SearchableLayout>;
};
```

Home.getLayout이 Home에 적용되어있고, 렌더링될 때 SearchableLayout으로 감싸짐.

#### getLayout 메서드

- Next.js에서 레이아웃을 유연하게 지정하기 위한 커스텀 함수
- 특정 페이지에만 적용할 수 있는 레이아웃을 쉽게 관리하기 위한 패턴

⇒ 만약 자식 페이지에서 getLayout이 정의되어있으면 해당 페이지를 특정 레이아웃으로 감싸는 역할을 함

- OnkeyDown

```typescript
    // enter를 눌러도 submit되게 함
    const onKeyDown = (e: React.KeyboardEvent<HTMLInputElement>) => {
        if (e.key === 'Enter') {
            onSubmit();
        }
    };
    
		// jsx
		<input
		      value={search}
		      onKeyDown={onKeyDown}
		      onChange={onChangeSearch}
		      placeholder="검색어를 입력하세요 ..."
		  />
```

### 2.9 한입북스 UI 구현하기

tsconfig.json

```typescript
"paths" : {
		"@/* : ["./src/*"] 
}
```

### 2.10 사전 렌더링과 데이터 페칭

React DataFetching의 문제점 : 초기 접속부터 데이터 로딩까지 오랜 시간이 걸림

```typescript
// 1. 불러온 데이터 보관할 state 선언
// 2. 데이터 fetch 함수 작성
// 3. 컴포넌트 마운트 시점에 fetchData 호출 // --> 요청 자체가 늦기에 데이터 요청도 늦음
// 4. 데이터 로딩중일 때 예외처리

import { useEffect, useState } from 'react';

export default function Page() {
    const [state, setState] = useState();

    const fetchData = async () => {
        const response = await fetch('https://api.example.com/data');
        const data = await response.json();
        setState(data);
    };

    useEffect(() => {
        fetchData();
    }, []);

    if (!state) {
        return <div>Loading...</div>;
    }

    return <div>...</div>;
}
```

![](/assets/img/2025-11-07/nextjs-section-2-6.png)

React :FCP까지 느린데, 데이터 로딩도 느림

FCP 이후에 API Request함

Next.js :  JS 렌더링 (`API 요청/응답`도 포함)

![](/assets/img/2025-11-07/nextjs-section-2-7.png)

데이터 로딩이 완료된 페이지도 받음

**만약 사전렌더링이(API 요청/응답이) 느리면 ?**

예) → **`빌드타임`** 사전 렌더링 사용 등 여러 사전 렌더링을 사용함 SSG

- `SSR`(서버사이드 렌더링)
  - 가장 기본적인 사전 렌더링
  - 요청이 들어올 때마다 사전 렌더링 진행
- `SSG`(정적 사이트 생성)
  - 빌드 타임에 미리 페이지를 사전 렌더링함
- `ISR`(증분 정적 재생성)

### 2.11, 2.12 SSR

SSR : Server Side Rendering

브라우저 요청이 들어올 때마다 사전 렌더링을 진행함

![](/assets/img/2025-11-07/nextjs-section-2-8.png)

```typescript
// 1. 요청이 들어옴
// 2. getServerSideProps : Page 페이지 요청보다 먼저 요청됨
// 3. Page 컴포넌트가 그려짐

export const getServerSideProps = () => { 

		// window.location, alert // 불가함, 브라우저가 아닌 서버에서 실행됨
    const data = 'Hello';

    return {
        props: { // props는 하나만 존재해야 함
            data,
        },
    };
};

// 두번 호출됨 (서버, 브라우저)
export default function Home(data: InferGetServerSidePropsType<typeof getServerSideProps>) {
    console.log(data); // 서버에서 호출, 브라우저에서 호출
    // window.location // 서버에서 호출 : 불가, 브라우저에서 호출 : 가능
    
    
    useEffect(() => { // 브라우저에서 호출, 마운트 이후_즉 화면 나타난 이후에 실행
        console.log(window);
    }, []);
    

    return (
        <div className={style.container}>
            <section>
                <h3>지금 추천하는 도서</h3>
                {books.map((book) => (
                    <BookItem key={book.id} {...book} />
                ))}
            </section>
            <section>
                <h3>등록된 모든 도서</h3>
                {books.map((book) => (
                    <BookItem key={book.id} {...book} />
                ))}
            </section>
        </div>
    );
}

Home.getLayout = (page: ReactNode) => {
    return <SearchableLayout>{page}</SearchableLayout>;
};
```

```typescript
// GetServerSidePropsContext : 브라우저로 받은 모든 요청
export const getServerSideProps = async (context: GetServerSidePropsContext) => {
    const { id } = context.params!;

    const book = await fetchOneBook(Number(id));

    return {
        props: { book },
    };
};

```

### 2.13 SSG

SSR : 브라우저의 접속 요청이 있을 때마다 매번 백엔드 서버에 request/response를 받음

장점 : 항상 새로운 데이터를 받음

**단점 : 데이터 응답 속도가 느릴 시 모든게 느려짐**

**SSG** : SSR의 단점을 해결하는 사전 렌더링 방식

`빌드 타임`에 페이지를 미리 사전렌더링 해 둠

`yarn build` 할 때 사전 렌더링을 함, 더이상 새롭게 페이지를 생성하지 않음 (dev 모드에선 무의미)

JS 실행(렌더링) 시 백엔드에게 Request, Response를 요청보냄

![](/assets/img/2025-11-07/nextjs-section-2-9.png)

장점 : 사전 렌더링에 많은 시간이 소요되는 페이지라도 사용자의 요청에는 빠른 속도로 응답이 가능

**단점 : 빌드 타임 이후에는 페이지를 새롭게 생성하지 않음 → `최신 데이터 반영은 어렵다`.**

정적인 데이터인 경우 적합하다.

특징 : SSR이 아니라면 기본은 SSG이다.

```typescript

// getServerSideProps -> getStaticProps
export const getStaticProps = async (context: GetServerSidePropsContext) => {
    const { id } = context.params!;

    const book = await fetchOneBook(Number(id));

    return {
        props: { book },
    };
};

export default function Page({ book }: InferGetServerSidePropsType<typeof getStaticProps>) {
```

![](/assets/img/2025-11-07/nextjs-section-2-10.png)

### 2.14 SSG 2

```typescript
// src/pages/search/index.tsx

// GetStaticPropsContext : context에 query 객체가 없음
export const getStaticProps = async (context: GetStaticPropsContext) => {
    const { q } = context.query;
    const books = await fetchBooks(q as string);

    return { props: { books } };
};

export default function Page({ books }: InferGetServerSidePropsType<typeof getStaticProps>) {
    // console.log(books);

```

GetStaticPropsContext : context에 query 객체가 없음

- **빌드 타임에 query를 알 수 없기 때문**

  ⇒ client side에서 진행하기

### 2.15, 2.16 SSG 3 동적 경로에 적용하기,  fallback 옵션

아래의 코드에선 오류가 발생

Error: `getStaticPaths` is required for dynamic SSG pages and is missing for '/book/[id]'.

```typescript
export const getStaticProps = async (context: GetStaticPropsContext) => {
    const { id } = context.params!;

    const book = await fetchOneBook(Number(id));

    return {
        props: { book },
    };
};

export default function Page({ book }: InferGetStaticPropsType<typeof getStaticProps>) {
    if (!book) return '문제가 발생했습니다';

    const { id, title, subTitle, description, author, publisher, coverImgUrl } = book;
    
    .
    .
    .
```

원인 : 1번, 2번, 3번 … 100번 ⇒ 어떠한 경로가 있는지를 모름 : 에러

해결책 : 사전 렌더링이 되기 전에 Page에 존재할 수 있는 모든 경로를 직접 설정하는 작업이 꼭 필요

`getStaticPaths` 로 경로를 설정해야 함

![](/assets/img/2025-11-07/nextjs-section-2-11.png)

```typescript
export const getStaticPaths = () => {
    return {
        paths: [
            { params: { id: '1' } }, // 파라미터의 값은 항상 문자열로!
            { params: { id: '2' } },
            { params: { id: '3' } },
            { params: { id: '4' } },
        ],
        fallback: false, // false ->  해당 Id 이외에는 404를 부름
    };
};
```

![](/assets/img/2025-11-07/nextjs-section-2-12.png)

#### Fallback 옵션

가정) Dynamic SSG에서  params로 지정한 ID가 아니다면

- false
  - 404 페이지를 반환

- ‘blocking’

  - **`즉시 생성`** (SSR 처럼 페이지를 실시간 생성)

    ![](/assets/img/2025-11-07/nextjs-section-2-13.png)

  **이미지**

  ![](/assets/img/2025-11-07/nextjs-section-2-14.png)

  내가 방문한 페이지도 .next에 남음 / 한번만 만들면 SSG 페이지처럼

  주의점 : **사전 렌더링 시간이 길어진다면 로딩**이 발생하게 됨

- true

  - 즉시 생성(페이지만 미리 반환 + 데이터는 후속으로 보내줌)

    Props(데이터)가 없는 페이지만 반환(**UI만 먼저 반환**) → 계산 → Props만 따로 반환

    ![](/assets/img/2025-11-07/nextjs-section-2-15.png)

    ```typescript
    import { GetStaticPropsContext, InferGetStaticPropsType } from 'next';
    import style from './[id].module.css';
    import fetchOneBook from '@/lib/fetch-one-book';
    import { useRouter } from 'next/router';
    import { notFound } from 'next/navigation';

    export const getStaticPaths = () => {
        return {
            paths: [
                { params: { id: '1' } }, // 파라미터의 값은 항상 문자열로!
                { params: { id: '2' } },
                { params: { id: '3' } },
                { params: { id: '4' } },
            ],
            fallback: true,
        };
    };

    export const getStaticProps = async (context: GetStaticPropsContext) => {
        const { id } = context.params!;

        const book = await fetchOneBook(Number(id));

        // 객체가 없다면 404페이지로 이동 시키고 싶을 때 ('문제가 발생했습니다'가 아닌!)
        if (!book) {
            return {
                notFound: true,
            };
        }

        return {
            props: { book },
        };
    };

    export default function Page({ book }: InferGetStaticPropsType<typeof getStaticProps>) {
        const router = useRouter();

        if (router.isFallback) return '로딩중입니다...';

        if (!book) return '문제가 발생했습니다';

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

### 2.17 ISR

Incremental Static Regeneration (증분 정적 재생성)

SSG 방식으로 생성된 정적 페이지를 **일정 시간을 주기로 다시 생성**하는 기술

![](/assets/img/2025-11-07/nextjs-section-2-16.png)

![](/assets/img/2025-11-07/nextjs-section-2-17.png)

만약 쿨타임이 60초라면

0~60초 : 매우 빠른 속도로 응답 가능(기존 SSG 방식)

60초     : v1 그대로 반환

60~       : 최신 데이터 반영 가능 (SSR 방식의 장점)

```typescript
export const getStaticProps = async () => {
    console.log('SSG 인덱스 페이지');

    const [allBooks, randomBooks] = await Promise.all([fetchBooks(), fetchRandomBooks()]); // 병렬로 요청

    return {
        props: {
            allBooks,
            randomBooks,
        },
        revalidate: 5, // ISR 전략
    };
};
```

![](/assets/img/2025-11-07/nextjs-section-2-18.png)

### 2.18 On-Demand ISR (주문형 재검증)

ISR : SSG + SSR 방식

ISR을 적용하기 어려운 페이지 :

- **시간과 상관없이** 사용자의 행동에 따라 데이터가 업데이트 되는 페이지

  예) 게시판

  60초 기준으로 revalidate한다면

  40초 때 사용자가 데이터 수정했다면? 24시간 이후 수정했다면?

  불필요함, SSR을 하면 안되나?

  ⇒ 요청을 기준으로 페이지를 다시 생성 : `On-Demand ISR`

  \*요청을 받을 때마다 페이지를 다시 생성

  예) 댓글이 있는 블로그 게시글 페이지

  이커머스 사이트 제품 정보 갱신

### 2.19 SEO 설정하기

#### favicon 변경 (앱 로고 변경)

#### 페이지 별 메타태그 관리

import Head from next/head

import Head from next/document : _document.tsx에서만 사용가능한 Head

**유의사항 : SSG 방식의 경우 첫 요청 시 메타태그가 적용이 안됨**

```typescript
export const getStaticPaths = () => {
    return {
        paths: [
            { params: { id: '1' } }, // 파라미터의 값은 항상 문자열로!
            { params: { id: '2' } },
            { params: { id: '3' } },
            { params: { id: '4' } },
        ],
        fallback: true,
    };
};

export const getStaticProps = async (context: GetStaticPropsContext) => {
    const { id } = context.params!;

    const book = await fetchOneBook(Number(id));

    // 객체가 없다면 404페이지로 이동 시키고 싶을 때 ('문제가 발생했습니다'가 아닌!)
    if (!book) {
        return {
            notFound: true,
        };
    }

    return {
        props: { book },
    };
};

export default function Page({ book }: InferGetStaticPropsType<typeof getStaticProps>) {
    const router = useRouter();

    // if (router.isFallback) return '로딩중입니다...';
    if (router.isFallback) { // 페이지가 fallback 중이라도 메타태그 임시 설정됨
        return (
            <>
                <Head>
                    <title>한입북스</title>
                    <meta property="og:image" content="/thumbnail.png" /> // 썸네일 이미지
                    <meta property="og:title" content="한입북스" /> 
                    <meta
                        property="og:description"
                        content="한입 북스에 등록된 도서들을 만나보세요"
                    />
                </Head>
                <div>로딩중입니다.</div>
            </>
        );
    }

		if (!book) return '문제가 발생했습니다';
		

    const { id, title, subTitle, description, author, publisher, coverImgUrl } = book;

    return (
        <>
            {/* SEO 설정 */}
            <Head>
                <title>{title}</title>
                <meta property="og:image" content={coverImgUrl} />
                <meta property="og:title" content={title} />
                <meta property="og:description" content={description} />
            </Head>
```

![](/assets/img/2025-11-07/nextjs-section-2-19.png)

### 2.21 페이지 라우터 정리

#### 장점

1. 파일 시스템 기반의 간편한 페이지 라우팅 제공

   ```
   |-Pages
   |——book
   |————[[…id]].tsx    —> book/100/23/12345/1231 (list 형식으로 저장됨)
   ```

1. 다양한 방식의 사전 렌더링 제공

   ![](/assets/img/2025-11-07/nextjs-section-2-20.png)

   리액트의 느린 FCP를 해결함

   사전렌더링 :

   - SSR
     - 매번 새로운 페이지 생성(최신 데이터 생성)
     - 데이터 요청 딜레이. 시  느려짐
   - SSG
     - 빌드 타임에 미리 페이지를 정적으로 생성해둠
     - 빌드 타임 이후에는 매번 같은 페이지를 응답함(최신 데이터 X)
   - ISR : 증분정적생성
     - SSG 방식 + 유통기한
       - 유통기한이 끝나면 새로운 페이지 반환
     - On-Demand 방식
       - revalidate 시 새로운 페이지로 전달

#### 단점

1. 페이지별 레이아웃 설정이 번거로움

   앱 라우터에서는 간단한 설정이 가능

   ```typescript

   // _app.tsx
   export default function App({
       Component,
       pageProps,
   }: AppProps & {
       Component: NextPageWithLayout;
   }) {
       const getLayout = Component.getLayout ?? ((page: ReactNode) => page);

       return <GlobalLayout>
   						    {getLayout(<Component {...pageProps} />)}
   				    </GlobalLayout>;
   }

   Home.getLayout = (page: ReactNode) => {
       return <SearchableLayout>{page}</SearchableLayout>;
   };

   ```

1. 데이터 페칭이 페이지 컴포넌트에 집중됨

   `getStaticProps` , `getServerSideProps` 로 페이지 컴포넌트에게 **`props`**로 데이터 전달해야 함

   ![](/assets/img/2025-11-07/nextjs-section-2-21.png)

   이 경우에는 redux, context API, recoil 사용

2. 불필요한 컴포넌트에도 JS 번들에 포함

   ![](/assets/img/2025-11-07/nextjs-section-2-22.png)

   불필요한 컴포넌트 : Hydration이 될 필요가 없는 컴포넌트 들(빨간색)

   - **서버**측에서 JS렌더링 시 실행

   - 하이드레이션 시 브라우저 측에서 실행

     하이드레이션 : HTML로만 렌더링 된 페이지에 **JS 연결(이벤트 등 상호작용을 하게 해줌) `TTI`** 라고 부름

   ![](/assets/img/2025-11-07/nextjs-section-2-23.png)

   서버 렌더링 때만 한번 실행되면 되나, 그러지 않음

   ⇒ 하이드레이션이 느려짐
{% endraw %}
