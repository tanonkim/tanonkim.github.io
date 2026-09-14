---
title: "Rendering Patterns 5 : React Server Components"
date: 2025-06-08 17:40:00 +0900
categories: [design-pattern, rendering]
tags: [design-pattern, react-server-components]
---

2020년 12월, React ServerComponents가 소개된 이후 리액트 서버 컴포넌트는 가장 주목을 받는 React 18기능 중 하나가 되었다.

### 리액트 컴포넌트의 Data fetching

리액트 릴리즈 이후, React 팀은 다방면으로 퍼포먼스를 향상했으나, Data Fetching의 경우 사용자 경험과 개발자 경험을 모두 만족할 수 있는 방법을 찾기는 어려웠다.

한 페이지에서 여러 정보를 보여주는 컴포넌트를 개발한다면 우리는 흔히 아래와 같이 컴포넌트를 구성한다.

```javascript
function Home({ memberId }) {
  return (
    <MemberDetails memberId={memberId}>
      <MoneyBalance memberId={memberId}></MoneyBalance>
      <PaymentHistory memberId={memberId}></PaymentHistory>
    </MemberDetails>
  );
}
```

부모 컴포넌트와 자식 컴포넌트 2개가 모두 유저 정보가 필요하고, 필요한 정보는 서버 API를 호출하여 받아야 한다.

개발자는 필요한 정보를 받아오는 방법으로 둘 중 하나를 선택하게 된다.

1. 모든 정보를 부모의 컴포넌트에서 하나의 거대한 API로 호출하여 자식으로 내려준다.
2. 컴포넌트에 필요한 API를 각 컴포넌트에서 호출한다.

첫 번째 방법으로 클라이언트에서 서버로 요청하는 API의 수는 줄일 수 있지만, 커플링이 강화되고 유지보수가 어려워진다. 만약 컴포넌트의 구성이 바뀌거나 자식 컴포넌트가 다른 컴포넌트로 이동되는 경우 불필요한 정보를 over-fetching하게 되는 셈이다.

두 번째 방법은 각 컴포넌트가 렌더링할 때 필요한 데이터를 보여줄 수 있는 장점이 있지만, high latency를 가진 클라이언트의 서버 요청은 늘어난다. 또한 부모 컴포넌트는 렌더링 이후 데이터를 받아오기 시작하고, 이 과정이 끝나기 전까지 자식 컴포넌트의 렌더링과 API 호출 또한 지연된다.

→ 연속된 CSR API 요청과 중첩된 API 요청으로 인한 waterfall은 사용자 경험을 떨어트릴 수 있다.

Facebook 팀은 이런 고민을 Relay + GraphQL로 waterfall 문제를 해결하지만, waterfall를 줄일 수 있는 해결책으로 React Server Component가 대안이 될 수 있다.

### No Client-Server waterfall

기존 리액트 컴포넌트의 비동기적 data fetching의 가장 큰 **문제점**은 클라이언트와 서버 간 요청의 high latency와 연속된 Client-Server API 요청으로 발생하는 waterfall이었습니다.

아래는 클라이언트 컴포넌트에서 흔히 사용되는 data fetching 방식의 예제이다.

```javascript
// Item.js
function Item(props) {
  const [item, setItem] = useState(null);

  useEffect(() => {
    fetch(`https://api.example.com/item/${props.id}`)
      .then(res => res.json())
      .then(
        (result) => {
          setItem(result);
        }
      )
  }, [props.id]])

  if (item == null) {
    return "Loading";
  } else {
    return (/* render note here... */);
  }
}
```

위의 코드는 각 컴포넌트에서 데이터를 요청하는 경우 실제 컴포넌트가 렌더링 될 때 필요한 데이터만 가져와 보여줄 수 있다는 장점이 있지만 클라이언트와 서버 사이의 API 요청은 늘어나게 된다.

하지만 부모 컴포넌트는 렌더링 후 필요한 데이터를 받고, 이 과정이 끝나기 전까지 자식 컴포넌트는 API 호출이 지연된다. 즉 성능 저하의 원인이 된다.

예) 서버 컴포넌트 도입

```javascript
// Item.server.js - 서버 컴포넌트의 파일 네임 컨벤션
import { fetch } from 'react-fetch';

function Note(props) {
  // 아래 요청은 client-to-server가 아닌 server-to-server로 진행됩니다.
  const item = fetch(`https://api.example.com/item/${props.id}`).json();

  if (item == null) {
    return <div>No Item Exist</div>;
  } else {
    return (/* render note here... */);
  }
}
```

서버 컴포넌트 도입 후 리액트 컴포넌트에서의 data-fetching은 간단해진다. 클라이언트 컴포넌트에서 발생하던 client-side waterfall을 제거해서 컴포넌트에 필요한 fetching하는 방식을 유지하면서 퍼포먼스를 향상할 수 있다.

하지만 중첩된 컴포넌트에서 여러 API요청에 따른 network waterfall은 여전히 존재한다. React팀은 이를 향상할 수 있는 데이터 preload api를 제공할 예정이다.

## React Server Component의 이점

서버컴포넌트는 서버에서 동작하고 렌더링된다는 특성 상 다양한 종류의 백엔드 리소스에 접근이 가능하고, HTML이 아닌 특별한 형태로 렌더링되어 클라이언트에 전달되기 때문에 클라이언트로 전달되는 번들 사이즈 또한 감소시킬 수 있다.

### 자유로운 서버 리소스 접근

서버 컴포넌트는 서버에서 동작하기에, DB 및 파일시스템, 인터널 서비스 같은 서버사이드 데이터 소스에 직접 접근이 가능하다.

```javascript
// Item.server.js - 서버 컴포넌트
import fs from 'react-fs';
import db from 'db.server';

function Item(props) {
  // NOTE: loads *during* render, low-latency data access on the server
  const item = db.notes.get(props.id); // 데이터베이스 접근
  const itemFromFile = JSON.parse(fs.readFile(`${id}.json`)); // 파일 접근

  if (item == null) {
    // handle missing note
  }
  return (/* render note here... */);
}
```

서버에서 fetching한 데이터는 클라이언트 컴포넌트에 props로 전달이 가능하다. 유의사항은 json으로 인코딩가능한 serializable prop만 가능하고, function은 전달할 수 없다.

### 제로 번들 사이즈 컴포넌트

프론트앤드 앱을 만들다보면 라이브러리를 사용하게 된다. 여럿 추가하다보면 번들 사이즈가 늘게 되고, 퍼포먼스에 악영향을 미치게 된다. 요즘 많은 번들러가 tree shaking을 지원하고 필요에 따라 code splitting을 지원해 번들 사이즈를 최대한 줄일 수 있지만, 결국 번들 사이즈가 늘어나는 것은 막을 수 없다.

예를 들어 클라이언트 컴포넌트 렌더링을 할 때 a,b 패키지가 번들에 추가되어야 한다고 가정하자. 하지만 서버 컴포넌트는 브라우저에서 다운되지 않고 `서버`에서 미리 렌더링된 static component를 전달하기에 패키지를 추가해도 번들 사이즈에 영향이 없다.

→ 유저 인터랙션이 없는 컴포넌트들을 서버 컴포넌트로 마이그레이션 한다면 동일한 뷰를 제공하면서 동시에 번들 사이즈와 초기 로딩 시간을 감소시킬 수 있다.

### 자동 코드 분할

Code splitting이란 하나의 거대한 자바스크립트 번들을 여러 개의 작은 번들로 쪼개어 필요할 때마다 클라이언트로 전송하는 방법이다.

클라이언트 컴포넌트에서 React.lazy와 dynamic import로 렌더링에 필요한 컴포넌트를 동적으로 불러온다.

```javascript
import React from 'react';

// one of these will start loading *when rendered on the client*:
const OldPhotoRenderer = React.lazy(() => import('./OldPhotoRenderer.js'));
const NewPhotoRenderer = React.lazy(() => import('./NewPhotoRenderer.js'));

function Photo(props) {
  // Switch on feature flags, logged in/out, type of content, etc:
  if (FeatureFlags.useNewPhotoRenderer) {
    return <NewPhotoRenderer {...props} />;
  } else {
    return <OldPhotoRenderer {...props} />;
  }
}
```

서버 컴포넌트는 이러한 단점을 두 가지 방식으로 해결했다.

서버 컴포넌트에서 import 되는 모든 클라이언트 컴포넌트를 code splitting 포인트로 간주하기 때문에 더 이상 `React.lazy`로 메뉴얼 하게 명시하지 않아도 된다.

## 리액트 서버 컴포넌트(RSC)와 서버사이드 렌더링(SSR)

### SSR

`클라이언트 사이드 렌더링`을 사용하는 앱의 경우 페이지 진입 시 HTML, 자바스크립트 그리고 모든 데이터가 로드되고 컴포넌트 렌더링이 끝나기 전까지 사용자는 아무런 기능이 없는 빈 화면만 보게 된다. 자바스크립트 번들의 용량이 크거나 사용자의 네트워크 속도가 느릴수록 사용자는 오랜 시간 빈 화면을 보게 되고 이는 사용자 경험을 저하시키게 된다.

반면 Next.js와 같은 `서버 사이드 렌더링`을 사용하는 애플리케이션은 클라이언트 애플리케이션의 자바스크립트 파일을 서버에서 먼저 HTML로 렌더링 한다. 페이지가 정상적으로 동작하기 위해서는 자바스크립트 번들이 모두 다운로드되고 [hydration](https://github.com/reactwg/react-18/discussions/46#discussioncomment-846714)이 완료되어야 하지만 빈 화면 대신 데이터가 존재하는 HTML을 제공함으로써 무거운 자바스크립트 파일이 다운로드되는 동안 사용자에게 의미 있는 콘텐츠를 제공할 수 있다.

서버 사이드 렌더링의 가장 큰 목적은 non-interactive한 버전의 클라이언트 컴포넌트를 최대한 빠르게 브라우저에 전달하여 초기 페이지의 First Contentful Paint 또는 Largest Contentful Paint 속도를 향상시는 것이다.

### 서버 컴포넌트와 서버 사이드 렌더링의 차이

서버 컴포넌트와 서버사이드 렌더링은 서버에서 렌더링 된다는 유사점이 있지만 해결하고자 하는 문제점이 다르다.

- **서버 컴포넌트의 코드는 클라이언트로 전달되지 않는다.** 하지만 서버 사이드 렌더링의 모든 컴포넌트 코드는 자바스크립트 번들에 포함되어 클라이언트로 전송된다.
- 서버 컴포넌트 페이지 레벨에 상관없이 모든 컴포넌트에서 서버에 접근이 가능하다. 하지만 Next.js는 가장 top level에서만 서버에 접근이 가능하다.
- **서버 컴포넌트는 클라이언트 상태를 유지하며 refetch 될 수 있다.** 서버 컴포넌트는 HTML이 아닌 특별한 형태로 컴포넌트를 전달할 수 있기에 필요한 경우 포커스, 인풋 입력값 같은 클라이언트 상태를 유지하며, 여러번 데이터를 가져오고, 리렌더링하여 전달할 수 있다.

  ↔ SSR의 경우 HTML로 전달되기 때문에 새로운 refetch가 필요한 경우 HTML 전체를 리렌더링해야 하며, 이로 인해 클라이언트 상태를 유지할 수 없다.
