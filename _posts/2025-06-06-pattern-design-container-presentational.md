---
title: "Design Patterns 2 : Container/Presentational Pattern"
date: 2025-06-06 18:30:00 +0900
categories: [design-pattern, design]
tags: [design-pattern, container-presentational]
---

리액트에서 **관심사의 분리(SoC)**를 강제하는 방법은 Container/Presentational Pattern을 이용하는 방법이 있다. 이를 통해 비즈니스 로직에서 뷰를 분리할 수 있다.

### 가정

6개의 강아지 사진을 다운받아 화면에 렌더하는 앱을 만들기

이상적으로 이 프로세스를 아래 두 가지로 분리하여 관심사의 분리를 강제하고 싶다.

1. Presentational Components : 데이터가 어떻게 사용자에게 보여질 지에 대해서만 다루는 컴포넌트, 예제에서는 강자지 사진 목록을 렌더링하는 부분
2. Container Components : 어떤 데이터가 보여질 지에 대해 다루는 컴포넌트, 예제에서는 강아지 사진들을 다운로드 한다.

강아지 사진을 다운로드 하는 것은 비즈니스 로직의 역할이고, 이미지를 보여주는 것은 뷰의 역할이다.

### Presentational Component

Presentational 컴포넌트는 props를 통해 데이터를 받는다. 이 컴포넌트의 주요 기능은 데이터를 화면에 표현하는 것이며, 그 목적을 위해 스타일 시트를 포함한다. 데이터는 건들이지 않는다.

아래 강아지 사진을 출력하는 예제를 확인해보자. 강아지 사진을 렌더링할 때 단순히 각 이미지를 API로부터 다운로드 받고 화면에 렌더링하는데, 이미지들을 props로 받아 **함수형 컴포넌트**만 만드는 것이다.

```javascript
// DogImages.js
import React from "react";

export default function DogImages({ dogs }) {
  return dogs.map((dog, i) => <img src={dog} key={i} alt="Dog" />);
}

// DogImagesContainer.js
import React from "react";
import DogImages from "./DogImages";

export default class DogImagesContainer extends React.Component {
  constructor() {
    super();
    this.state = {
      dogs: []
    };
  }

  componentDidMount() {
    fetch("https://dog.ceo/api/breed/labrador/images/random/6")
      .then(res => res.json())
      .then(({ message }) => this.setState({ dogs: message }));
  }

  render() {
    return <DogImages dogs={this.state.dogs} />;
  }
}
```

여기서 DogImages 컴포넌트는 Presentational 컴포넌트이다.

Presentational 컴포넌트는 UI 변경을 위한 상태 외에는 상태를 갖지 않는다. prop을 통해 받은 데이터는 Presentational 컴포넌트에 의해 수정되지 않는다.

Presentational 컴포넌트는 Container 컴포넌트로부터 데이터를 받는다.

### Container 컴포넌트

Container 컴포넌트의 주요 기능은 **Presentational 컴포넌트**에 **데이터를 전달**하는 것이다.

Container 컴포넌트 자체는 화면에 아무것도 렌더링하지 않는다. → 아무것도 화면에 그리지 않으니 스타일시트도 포함하지 않는다.

```javascript
import React from "react";
import DogImages from "./DogImages";

export default class DogImagesContainer extends React.Component {
  constructor() {
    super();
    this.state = {
      dogs: []
    };
  }

  componentDidMount() {
    fetch("https://dog.ceo/api/breed/labrador/images/random/6")
      .then(res => res.json())
      .then(({ message }) => this.setState({ dogs: message }));
  }

  render() {
    return <DogImages dogs={this.state.dogs} />;
  }
}
```

강아지 사진 목록을 DogImages 컴포넌트에게 전달하기 위해 Container 컴포넌트을 만들었다. → Container 컴포넌트는 외부 API로부터 강아지 이미지를 호출하고, Presentational 컴포넌트인 DogImages 컴포넌트에게 전달하고 있다.

**Presentational Component + Container Component를 조합하면 비즈니스 로직과 뷰를 분리할 수 있다.**

### Hooks

Container/Presentational 패턴은 React Hooks로 대체 가능하다. React에 Hooks가 추가되면서 Container 컴포넌트 없이도 StateLess 컴포넌트를 쉽게 만들 수 있게 되었다.

DogImagesContainer 컴포넌트에 있는 데이터 로드 코드를 아래와 같이 커스텀 훅으로 만들 수 있다.

```javascript
export default function useDogImages() {
  const [dogs, setDogs] = useState([])

  useEffect(() => {
    fetch('https://dog.ceo/api/breed/labrador/images/random/6')
      .then(res => res.json())
      .then(({ message }) => setDogs(message))
  }, [])

  return dogs
}
```

위의 훅을 사용하면 데이터를 받아오기 위해 DogImagesContainer 컴포넌트를 사용할 필요가 없다. 대신 Presentational 컴포넌트인 DogImages에서 훅을 직접 호출해 사용하면 된다.

```javascript
// DogImages.js

import React from "react";
import useDogImages from "./useDogImages";

export default function DogImages() {
  const dogs = useDogImages();

  return dogs.map((dog, i) => <img src={dog} key={i} alt="Dog" />);
}
```

`useDogImages` 훅을 사용하여 Container/Presentational 패턴을 사용한 것과 같이 비즈니스 로직과 뷰를 분리했다.

### FSD 아키텍쳐에서의 Container/Presentational

해당 useDogImages Hook은 feature의 model에, UI를 그리는 Presentational 컴포넌트인 DogImages는 widget의 ui 폴더에 목적대로 포함시키면 된다.

index.ts를 통해 export할 컴포넌트 (여기선 model에 존재하는 hook)는 export 후 widget의 ui에 존재하는 DogImages에 import하면 된다.

### 장점

1. 해당 패턴을 사용하면 자연스레 관심사의 분리를 구현하게 된다. Presentational 컴포넌트는 UI를 담당하는 순수함수로 작성되는 반면 Container 컴포넌트는 상태와 기타 데이터를 책임진다.
2. Presentational 컴포넌트는 데이터 변경과는 무관하므로 여러 곳에서 다양한 목적으로 재사용할 수 있다.
3. Presentational 컴포넌트는 앱의 비즈니스 로직을 수정하지 않으므로, 코드 베이스에 대해 이해가 깊지 않은 개발자더라도, 쉽게 수정이 가능하다. 공통으로 쓰이는 Presentational 컴포넌트가 디자인의 요구사항에 따라 변경만 하면 된다.
4. Presentational 컴포넌트는 테스트하기도 쉽다.

### 단점

Container/Presentation 패턴은 비즈니스 로직과 렌더링 로직을 쉽게 분리할 수 있지만, 훅을 활용하면 클래스형 컴포넌트를 사용하지 않고도, 그리고 이 패턴을 따르지 않아도 같은 효과를 볼 수 있다.
