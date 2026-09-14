---
title: "Rendering Patterns 3 : CSR, SSR, SSG(Static), ISR"
date: 2025-06-08 17:20:00 +0900
categories: [design-pattern, rendering]
tags: [design-pattern, ssr]
---

{% raw %}
## 1. CSR_Client Side Rendering

클라이언트 사이드 렌더링에서 서버는 뼈대가 되는 최소한의HTML만 렌더링한다. 로직, 데이터, 템플릿, 라우팅을 통한 페이지 전환은 모두 브라우저에서 실행되는 JavaScript에 의해 처리된다. CSR은 SPA 개발에 거의 필수적으로 사용된다.

### 클라이언트 사이드 렌더링 - 기본구조

HTML:

```javascript
<div id="root"></div>
```

JS:

```javascript
function tick() {
  const element = (
    <div>
      <h1>Hello, world!</h1>
      <h2>It is {new Date().toLocaleTimeString()}.</h2>
    </div>
  )
  ReactDOM.render(element, document.getElementById('root'))
}
setInterval(tick, 1000)
```

서버를 통해 처리하는 것은 따로 없고, 렌더링된 HTML은 그대로 그 자리에서 업데이트 된다.

### JavaScript 번들과 성능

클라이언트 사이드 렌더링에서는 JavaScript 번들의 크기가 클 수록 FCP, TTI가 커진다.

![](/assets/img/2025-06-08/pattern-rendering-csr-ssr-ssg-isr-1.png)

번들의 크기가 증가하면 브라우저에서 빈 html을 보여주기에 사용자는 처름에 FCP까지의 모든 구간에서 빈 화면을 보게 된다.

### CSR React의 장점과 단점

React의 모든 앱의 로직들은 클라이언트 사이드에서 실행되고, 서버와는 API 호출을 통해 데이터를 조회하거나 저장하게 된다. UI 또한 클라이언트에서 만들어진다.

전체 웹은 첫 번째 요청에서 다운되며, 페이지 렌더를 위해 서버에 별도의 처리를 하지 않는다. 화면 또는 데이터를 변화시키기 위한 코드들은 모두 클라이언트에서 실행된다.

클라이언트 사이드 렌더링은 SPA를 만들어 사용자가 페이지를 새로고침하지 않아도 페이지를 탐색할 수 있어 좋은 사용자 경험을 줄 수 있다.

![](/assets/img/2025-06-08/pattern-rendering-csr-ssr-ssg-isr-2.png)

뷰를 변경하기 위한 데이터 처리량이 제한되어있기 때문에, 그리고 JS를 실행한 컨텐츠가 렌더링되기 때문에 페이지간 라우팅 속도가 빨라져 페이지가 더 빨리 반응하는 것 처럼 보인다.

**주의점**

1. SEO 고려사항.

   대부분 웹 크롤러들은 웹사이트의 서버 렌더링을 직접적으로 읽는다. 클라이언트 사이드 렌더링 과정이 복잡해져 API 요청 등으로 네트워크 요청 단계가 늘어날 경우, 의미있는 렌더링이 늦어져 크롤러가 인덱싱할 수 없게 된다. →SEO 최적화를 위한 추가적인 작업을 해야 한다.

2. 성능

   클라이언트 사이드 렌더링을 하면 서버를 여러번 거치는 게 아니기 때문에 인터렉션 중에 응답 시간이 향상될 수 있다. 하지만 브라우저가 처음 렌더링 될 때 JavaScript를 로드하고 처리하기까지 기다려야 한다. JavaScript 번들 크기가 증가할수록, 클라이언트의 성능이 부족할 수록 사용자 경험에 악영향을 줄 수 있다.

3. 유지보수성

   클라이언트 측과 서버 측 양쪽에서 다른 언어로 각각 구현해야 할 수 있다. → 비즈니스 로직을 깔끔히 처리하는게 어렵다.

4. 데이터 페칭

   클라이언트사이드 렌더링에서 데이터 페칭은 주로 이벤트에서 시작된다. 처음에는 데이터 없이 로드될 수 있지만, 데이터는 페이지 로드나 API 호출을 통해 받게 된다. → TTI까지의 시간을 증가시킨다.

### 클라이언트 사이드 렌더링의 성능

클라이언트 사이드 렌더링 성능은 JavaScript 번들 사이즈와 반비례하기 때문에 성능을 고려하여 구조화해야 한다.

- 번들 크기 관리

  초기 페이지 로드 속도를 위해 합리적인 JavaScript 번들 크기를 결정해야 한다. minified되고, Gzipped되었을 때 최소 100-170KB정도로 잡는 것이 좋은 시작점이다. ⇒ 코드가 이보다 더 늘어나면 필요에 따라 추가로 받도록 구성해야 한다.

- 프리로딩

  페이지 로드에 필수적인 리소스들에 이 기법을 적용할 수 있다. `<head>` 태그 안에 지시자를 사용하여 처리할 수 있다.

  ```javascript
  <link rel="preload" as="script" href="critical.js">
  ```

  브라우저가 페이지 렌더링 매커니즘을 시작하기 전에 critical.js를 로드하도록 할 수 있다. 따라서 이 스크립트는 더 일찍 시작될 수 있고, 페이지 렌더링 매커니즘을 방해하지 않으므로, 성능 향상에 도움을 줄 수 있다.

- 지연로딩

  지연로딩을 사용하여 리소스들이 나중에 필요해질 때 다운로드받도록 할 수 있다. 이 방법을 사용하면 초기에 받아야 하는 번들량이 줄어들어 최초 페이지 로드 타임이 감소한다.

- 코드 스플리팅

  번들 크기가 증가하는 것을 예방하기 위해 코드를 나눌 수 있다. 코드 스플리팅은 Webpack과 같은 모듈 번들러에 의해 지원되며, **동적**으로 로드 가능한 여러 번들을 만들 수 있다.

- **서비스 워커를 이용한 앱 쉘 캐싱**

  이 기법은 앱내 최소한의 UI를 구성하는 HTML, CSS, JavaScript를 캐싱하는 것이다. 이 서비스 워커를 이용하면 오프라인에서도 이 캐시를 사용할 수 있다. 껍데기 UI는 미리 보이고, 나머지 필요 데이터는 점진적으로 호출이 가능해, 네이티브 앱을 이용하는 것과 비슷한 사용자 경험을 제공할 수 있다.

---

## 2. SSR_Server Side Rendering

서버사이드 렌더링은 웹 컨텐츠의 렌더링에서 꽤 오랫동안 사용된 방법이다. 서버사이드 렌더링에서는 사용자의 페이지 요청에 대해 전체 HTML을 만들어서 응답한다. 렌더링된 컨텐츠에는 데이터 저장소나, 외부 API로부터 받은 데이터가 포함되어있다.

데이터 조회, 컨텐츠를 표현하기 위한 HTML을 만드는 것 모두 서버에서 처리한다. 따라서 서버사이드 렌더링을 사용하면 데이터를 조회하고 템플릿 렌더링을 하이 위해 서버와 별도로 통신하지 않아도 된다. → 클라이언트 측에 렌더링 코드는 필요없어서 따로 코드를 전송하지 않아도 된다.

### 클래식 서버사이드 렌더링 구현

예) 현재 시간을 보여주는 페이지

```html
<!DOCTYPE html>
<html>
   <head>
       <title>Time</title>
   </head>
   <body>
       <div>
       <h1>Hello, world!</h1>
       <b>It is <div id=currentTime></div></b>
       </div>
   </body>
</html>
```

```javascript
function tick() {
  var d = new Date()
  var n = d.toLocaleTimeString()
  document.getElementById('currentTime').innerHTML = n
}
setInterval(tick, 1000)
```

HTML이 서버에서 렌더링되어 내려온 뒤, 시간은 tick()에 의해 노출된다. → 이런 데이터를 서버 시간과 같은 서버 데이터를 이용하려면 HTML이 렌더링되기 전에 코드를 서버측에서 실행해야 한다. 따라서 페이지를 완전히 새로고침하지 않으면 최신 데이터를 볼 수 없게 된다.

### 서버사이드 렌더링의 장점과 단점

렌더링 코드를 서버측에서 실행하여 자바스크립트 크기를 줄이면 FCP, TTI가 빨라진다.

![](/assets/img/2025-06-08/pattern-rendering-csr-ssr-ssg-isr-3.png)

페이지 내 다양한 UI 요소들과 비즈니스 로직들이 포함되었을 때, 서버사이드 렌더링은 클라이언트 사이드 렌더링에 비해 상당히 **작은 자바스크립트 번들**을 사용한다. → FCP, TTI가 줄어들고 FCP와 TTI가 거의 같아진다.

서버사이드 렌더링을 사용하면 화면에 요소가 보여지고, 인터렉트 가능해지기까지 더 짧은 시간만 기다려도 된다.

**클라이언트 사이드 자바스크립트의 여유**

개발팀은 원하는 성능을 이끌어내기 위해 페이지 내의 자바스크립트 번들의 크기를 관리해야 한다. 서버사이드 렌더링을 사용하면 페이지 렌더에 쓰이는 자바스크립트 코드들을 직접 제거할 수 있어, 서드파티 라이브러리 등 앱에 필요한 다른 자바스크립트 코드를 위한 공간이 확보된다.

**검색엔진 최적화**

검색엔진 크롤러는 서버사이드 렌더링 앱의 컨텐츠에 대해 쉽게 크롤링할 수 있고, 페이지의 검색 엔진 최적화 수준이 향상된다.

모든 상황에 완벽하게 처리하긴 어려우므로 최적화 작업을 따로 해주어야 한다.

**느린 TTFB**

모든 프로세스가 서버에 의해 처리되므로 아래와 같은 상황 등으로 인해 서버의 응답이 지연될 수 있다.

- **동시에 아주 많은 사용자들이 데이터 로딩을 시작한다. (웹/서버 모두)**
- 네트워크가 느리다.
- 서버 코드가 최적화되어있지 않다.

**특정 인터렉션의 새로고침 필요**

모든 코드를 클라이언트에서 사용할 수 없어 주요 동작들에 대해 서버 데이터가 필요한 경우, 전체 페이지를 새로고침해야 할 수 있다. 이렇게 되면 사용자가 인터렉션 중간에 더 오래 기다려야 하기 때문에, 결과적으로 인터렉션 시간이 더 늘어날 수 있다.

### Next.js의 서버사이드 렌더링

각 요청에 대한 pre-render를 수행한다.

```typescript
export const getServerSideProps = async (context: GetServerSidePropsContext) => {
    const { q } = context.query;
    const books = await fetchBooks(q as string);

    return { props: { books } }; // will be passed to the page component as props
};

export default function Page({ books }: InferGetServerSidePropsType<typeof getServerSideProps>) {
    console.log(books);

    return (
        <>
            {/* SEO 설정 */}
            <Head>
                <title>검색결과</title>
                <meta property="og:image" content="/thumbnail.png" />
                <meta property="og:title" content="검색결과" />
                <meta property="og:description" content="등록된 도서들을 만나보세요" />
            </Head>
            <div>
                {books.map((book) => (
                    <BookItem key={book.id} {...book} />
                ))}
            </div>
        </>
    );
}

Page.getLayout = (page: ReactNode) => {
    return <SearchableLayout>{page}</SearchableLayout>;
};
```

### 서버를 위한 React

React는 동형적으로 될 수 있다. → 함수가 브라우저 뿐만이 아닌 서버와 같은 기타 플랫폼에서도 실행될 수 있다는 의미이다. 따라서 React를 활용하여 서버에서도 렌더링이 가능하다.

```javascript
ReactDOMServer.renderToString(element)
```

renderToString은 `ReactDom.hydrate()` 와 함께 사용된다. 이 함수는 서버에서 렌더링된 HTML을 그대로 클라이언트에서 쓸 수 있게 해주므로, 클라이언트에서는 이벤트 핸들러만 등록하면 된다.

- 공식문서

`renderToString`은 문자열을 즉시 반환하므로, 로딩 중인 콘텐츠를 스트리밍하는 것을 지원하지 않는다. 가능하면 다음과 같은 완전한 기능을 갖춘 대안을 사용하는 것을 권장합니다.

- Node.js를 사용하는 경우 [`renderToPipeableStream`](https://ko.react.dev/reference/react-dom/server/renderToPipeableStream)을 사용

## 3. SSG_Static Rendering

서버사이드 렌더링에서 서버 처리 시간이 늘어나는 만큼 TTFB(Time to First Byte)에 부정적인 영향을 주는 것을 알 수 있다. 또 클라이언트 사이드 렌더링에서 자바스크립트 번들 크기가 증가하면 스크립트를 다운받아 처리하는 시간이 증가하여, FCP, LCP, TTI에 안좋은 영향을 준다.

SSG _ Static 렌더링은 사이트를 빌드할 때 미리 렌더링해 둔 HTML 자체를 서빙하여 이런 문제를 해결한다. 사용자가 접속할 수 있는 각 라우팅 경로에 대응하는 HTML 파일들이 미리 생성된다.

정적 HTML파일들은 캐시가 가능하므로 유연성을 제공한다. HTML파일들을 미리 만들어두고 사용하기 때문에, 요청을 처리받았을 때 처리 시간은 거의 무시할 수 있는 수준이며, 결과적으로 빠른 TTFB와 FCP/TTI를 달성할 수 있다.

### SSG 기본 구조

소개, 연락처, 온라인 커머스 앱  상품 페이지들은 정적 페이지가 될 수 있다.

```javascript
// about/page.tsx
export default function About() {
  return (
    <div>
      <h1>About Us</h1>
      {/* ... */}
    </div>
  )
}
```

사이트가 next build를 통해 빌드될 때 about.html로 프리렌더되어 /about 경로에 서빙된다.

### 데이터를 포함한 SSG

만들어야 하는 HTML파일의 갯수는 작성한 블로그 포스트의 갯수나 상품의 갯수에 따라 다르다. 이런 페이지들이 얼마나 있는지 알기 위해 HTML로 만들어진 페이지에 해당하는 리스트 데이터가 필요하다.

```typescript
// book/[id].tsx
import { GetStaticPropsContext, InferGetStaticPropsType } from 'next';
import style from './[id].module.css';
import fetchOneBook from '@/lib/fetch-one-book';
import { useRouter } from 'next/router';
import Head from 'next/head';

export const getStaticPaths = () => {
    return {
        paths: [
	          // map함수 사용으로 리팩토링 가능 
            { params: { id: '1' } }, // 파라미터의 값은 항상 문자열로!
            { params: { id: '2' } },
            { params: { id: '3' } },
            { params: { id: '4' } },
        ],
        fallback: false, // id : 5 -> 404 Page
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

    if (router.isFallback) {
        return (
            <>
                <Head>
                    <title>북스</title>
                    <meta property="og:image" content="/thumbnail.png" />
                    <meta property="og:title" content="북스" />
                    <meta
                        property="og:description"
                        content="북스에 등록된 도서들을 만나보세요"
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
        </>
    );
}
```

**`getStaticProps()`**

페이지를 만들기 위한 데이터가 서버 외부에 있는 경우 데이터는 미리 받아두어야 한다. 불러온 데이터는 pre-render될 페이지에 props로 제공된다.

**`getStaticPaths()`**

위의 예시에서 나열된 각 페이지들에 대해 상세 페이지를 만들어야 한다. 이런 상세 페이지들은 리스트에서 각각 대응하는 항목의 링크를 클릭하여 라우팅하게 된다.

상품번호 101, 102, 103에 해당하는 상품이 있다고 가정해 보자. 각각의 상품 상세는 `/book/101`, `/book/102`, `/book/103` 으로 접근할 수 있다. 이를 위해 Next.js의 `getStaticPath()` 를 동적 라우팅과 조합하여 사용할 수 있다.

- fallback: true

  해당 옵션은 빌드 시 존재하지 않는 id인 5번으로 요청한다면 on-demand로 생성한 뒤 캐시한다. 다음번의 요청은 캐시에 저장되어 static하게 서빙된다. SSR처럼 동작

- fallback: false

  404page를 보여준다.

- fallback : ‘blokcking’

  요청 시 페이지 생성 + 로딩 UI 없이 완전히 렌더링 후 반환

## 4. ISG_ Incremental Static Generation

정적 사이트 렌더링(이하 SSG)은 서버 사이드 렌더링과 클라이언트 사이드 렌더링이 가진 대부분의 문제를 해결하지만 주로 정적인 컨텐츠를 렌더링하는 데 적합하다. 렌더링할 컨텐츠가 동적이거나 자주 변경되는 경우 제한되는 부분이 많다.

→ 하나의 페이지 때문에 전체 페이지를 다시 빌드하는 것은 비효율적이다. 따라서 큰 사이트나 앱을 SSG로 빌드하는 것은 애매하다.

ISR_증분정적 재생성, 점진적 정적 사이트 렌더링 패턴은 SSG의 동적 데이터에 관련된 문제와 자주 변경되는 대량의 데이터에 대한 문제를 해결할 수 있는 패턴으로 도입되었다. ISG를 사용하면 페이지에 대한 요청이 들어오는 중간에도 백그라운드에서 pre-rendering하여 기존 페이지를 업데이트하거나, 새로운 페이지를 추가할 수 있다.

**새로운 페이지 추가**

```javascript
export const getStaticPaths = () => {
    return {
        paths: [
	          // map함수 사용으로 리팩토링 가능 
            { params: { id: '1' } }, // 파라미터의 값은 항상 문자열로!
            { params: { id: '2' } },
            { params: { id: '3' } },
            { params: { id: '4' } },
        ],
        fallback: true, // id : 5 -> SSR 방식처럼 동작 후 캐시
    };
};
```

fallback : true가 된다면 fallback 컴포넌트가 노출되고, 페이지가 만들어지고 나면 캐시되어 페이지가 보여지게 된다.

**존재하는 페이지 업데이트**

기존에 존재하는 페이지를 리렌더링하기 위해 적합한 타임아웃을 설정할 수 있다. 정의한 타임이 지나면 캐시를 만료하고 패이지를 재생성(revalidate)하게 한다. 사용자는 `stale-while-revalidate` 전략을 사용하여 재생성이 이루어지는 와중에도 캐시된 버전이나 새로운 버전을 받아볼 수 있게 해준다.

```javascript
export async function getStaticProps() {
  return {
    props: {
      products: await getProductsFromDatabase(),
      revalidate: 60, // 60 second
    },
  }
}
```

### ISG의 장점

ISG는 SSG의 장점 뿐만 아니라 아래의 이점을 제공한다.

1. 동적 데이터 : 이는 ISG가 탄생한 근본적인 이유로, 전체 사이트를 다시 빌드하지 않아도 동적 데이터를 지원할 수 있다.
2. 속도 : 데이터를 받는 것과 렌더링을 백그라운드에서 하기 때문에 SSG보다는 빠르다.
3. 가용성 : 그나마 최신 버전의 페이지들은 계속 서빙가능하다. 재생성이 실패하더라도, 예전 버전은 언제든지 서빙이 가능하다.
4. 지속성 : 페이지 재생성은 서버에서 한번에 하나씩 이루어지기 때문에 DB와 백엔드 서버의 부하가 적다.
5. 쉬운 배포 : ISG처럼 pre-render된 페이지 서빙을 위해 CDN을 사용할 수 있다.
{% endraw %}
