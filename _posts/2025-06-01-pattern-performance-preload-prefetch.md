---
title: "Performance Patterns 6 : Preload, Prefetch"
date: 2025-06-01 14:16:00 +0900
categories: [design-pattern, performance]
tags: [design-pattern, preload]
---

## 1. Preload

Preload는 늦게 발견될 수 있는 **중요한 리소스들을 더 일찍 요청**할도록 할 수 있는 브라우저 측 최적화 기능이다.

앱이 렌더링되는데 중요한 리소스들의 로딩 순서를 수동으로 지정할 수 있다면 CoreWeb Vitals에 긍정적인 영향을 줄 수 있을 것이다. 다만 TradeOff를 아는 것도 중요하다.

### Vanilla-JS

```
<link rel="preload" href="emoji-picker.js" as="script">  ...  </head>  <body>    ...    <script src="stickers.js" defer></script>    <script src="video-sharing.js" defer></script>    <script src="emoji-picker.js" defer></script>
```

```javascript
<link rel="preload" href="emoji-picker.js" as="script">
  ...
  </head>
  <body>
    ...
    <script src="stickers.js" defer></script>
    <script src="video-sharing.js" defer></script>
    <script src="emoji-picker.js" defer></script>
```

TTI나 FIP(First Input Delay)를 최적화할 때 preload는 상호작용 코드들이 포함된 번들을 로드할 때 유용하다.

주의점 : 우선순위가 높아진 스크립트 때문에 이미지, 폰트 등의 리소스들이 밀려 FCP, LCP가 늦어지지 않도록 해야 한다.

퍼스트 파티 코드들의 로딩을 최적화하기 위해 `<script defer>`를 사용하며 `<head>`태그 안에 둘 수도 있고. `<body>`태그 안에 두어 해당 리소스가 빨리 발견되도록 할 수도 있다.

### React에서의 Preload

prefetching이 곧 사용될 지 모르는 리소스를 캐시하는 방법이라면, preload는 즉시 사용될 리소스를 미리 로드한다.

앱 내 웹폰트들은 초기 렌더링과 즉시 적용되어야 하거나, 이미지가 사용자에게 즉시 보여져야 할 수 있다.

가정 :  **`EmojiPicker`**컴포넌트가 페이지의 초기 렌더링 시점에 즉시 보여야 한다.

이 코드가 메인 번들에 포함되어있지 않겠지만, 병렬로 로드되어야 한다. 이 때 prefetch처럼 코드에 특정 주석을 사용하여 Webpack에게 해당 모듈을 preload해야 한다고 알려줄 수 있다.

```
const EmojiPicker = import(/* webpackPreload: true */ './EmojiPicker')
```

```javascript
const EmojiPicker = import(/* webpackPreload: true */ './EmojiPicker')
```

번들 결과를 확인해 보면 문서의 `<head>`태그에 `rel="preload"`속성이 있는 `<link>`태그가 추가된 것을 확인할 수 있다.

```javascript
<link rel="prefetch" href="emoji-picker.bundle.js" as="script" />
<link rel="prefetch" href="vendors~emoji-picker.bundle.js" as="script" />
```

이렇게 처리된 EmojiPicker는 초기 번들과 병렬로 다운된다. → 초기 렌더링 후 EmojiPicker를 따로 받는 시간을 기다려야 하는 상황과는 달리, 거의 즉시 해당 리소스를 사용할 수 있게 되었다. 하지만 이런 식으로 리소스의 우선순위를 정하면 사용자의 기기 성능이나, 인터넷 연결 상태에 따라 초기 로딩을 꽤 지연시킬 수 있다.

결론 : 초기 렌더링 후 1초 정도 이내에 즉시 출력되어야 하는 리소스에만 제한적으로 사용되는 것이 좋다.

`prefetch`는 **브라우저가 네트워크 상황을 고려하여** 리소스를 받아오기를 결정하지만, preload는 미리 로드된다.

### Preload와 script async 속성 함께 사용하기

스크립트는 우선순위를 높여 다운로드하고 싶지만 파서 처리 과정이 메인스레드를 블록하는것을 피하고 싶다면 preload와 async속성을 함께 이용하는것도 방법이다. 하지만 다른 리소스들의 다운로드가 지연될 수 있으니 주의가 필요하다.

```javascript
<link rel="preload" href="emoji-picker.js" as="script">
<script src="emoji-picker.js" async>
```

### 결론

preload를 사용할 때 주의가 필요하며, 앱에 미치는 영향을 측정할 것을 권장한다. preload를 이미지에 설정한 경우 브라우저가 빨리 발견하도록 할 수 있지만, 잘못 사용한 경우 preload는 오히려 FCP를 증가시키고, 의도한대로 동작하지 않을 수 있다.

## 2. Prefetch

prefetch는 브라우저 측 최적화 기능으로, 앱의 두 번째 라우팅이나 페이지들이 로드되기 전에 필요한 리소스들을 미리 받을 수 있게 해준다.

Prefetch는 아래와 같이 HTML에 선언적으로 사용, HTTP 헤더를 이용하는 방법, 서비스 워커를 이용하는 방법 혹은 웹팩의 특정 기능을 이용하여 구현할 수 있다.

```javascript
<link rel="prefetch">
Link: </js/chat-widget.js>; rel=prefetch // HTTP 헤더
```

### 이전의 예제 문제점

뷰포트에 들어오고 상호작용이 발생할 때 모듈을 불러오는 이전의 예제에서, 버튼을 눌러 컴포넌트를 토글할 때 종종 딜레이가 발생될 수 있다. → 버튼을 눌렀을 때에도 해당 모듈을 아직 불러오는 중이기 때문에 발생하는 현상이다.

컴포넌트나 리소스들은 앱 이용 중에 prefetch될 수 있는데, 이를 위해 import문에 Webpack의 magic comment를 추가할 수 있다.

```javascript
const EmojiPicker = import(/* webpackPrefetch: true */ './EmojiPicker')
```

이 경우 번들 결과에서는 `head`태그에 아래처럼 `rel=prefetch`속성을 가진 `link`태그를 작성하게 된다.

```javascript
<link rel="prefetch" href="emoji-picker.bundle.js" as="script" />
<link rel="prefetch" href="vendors~emoji-picker.bundle.js" as="script" />
```

이렇게 처리된 모듈들은 사용자가 모듈을 요청하기 전에 브라우저가 알아서 요청하여 로드 처리한다. 브라우저 프로세스가 유휴 상태이며, 네트워크가 좋다고 판단되면 해당 모듈들을 순서대로 불러와 캐시해둔다.

리소스를 캐시해뒀기 때문에 로딩타임은 크게 줄어들어 버튼을 클릭할 때의 기다리는 시간 역시 짧아진다. prefetch하여 캐시된 모듈을 받아오기 때문이다.

### 유의점

prefetch가 로딩 타임을 줄이기 위한 좋은 방법이긴 하지만, 사용자가 끝까지 prefetch된 컴포넌트를 사용하지 않을 경우 불필요하게 리소스를 로드하게 되어, 메모리를 낭비하고 앱을 느리게 할 수 있다. 꼭 필요한 리소스에만 prefetch를 적용해야 한다.
