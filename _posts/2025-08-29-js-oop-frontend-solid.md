---
title: "프론트엔드와 SOLID"
date: 2025-08-29 13:14:00 +0900
categories: [javascript, oop]
tags: [javascript, solid]
---

## SOLID 원칙

프로그래밍 세계에서 SOLID 원칙은 굉장히 유명한 용어이다. 실무단의 아키텍쳐 설계부터, 코딩과 리뷰까지 두루두루 사용되는 원칙이며, 리액트 공식문서에서도 S(단일 책임 원칙)를 언급할 만큼 다양한 곳에서 언급되고 있다.

SOLID는 다음 5가지 원칙의 앞 글자만 따온 단어이다.

SRP : Single Responsibility Principle_단일 책임 원칙

OCP : Open/Close Principle_개방-폐쇄 원칙

LSP : Liskov Substitution Principle_리스코프 치환 원칙

ISP : Interface Segregation Principle_인터페이스 분리 원칙

DIP : Dependency Inversion Principle_의존성 역전 원리

### 콘웨이 법칙

SOLID는 원칙이다. 원칙이면 지키면 아주 좋은 것이고, 이정표로 삼기 적당한 것이다. 원칙보다 우선하는 것은 법칙이다. 법칙은 그것이 진리임을 의심할 여지가 없는 것으로, 하나의 현상으로 볼 수 있다.

소프트웨어 공학에서 유명한 법칙 중 하나인 콘웨이 법칙은 SOLID 원칙에서도 대전제가 된다.

소프트웨어 구조는 해당 소프트웨어를 개발한 조직의 커뮤니케이션 구조를 닮게 된다. - 콘웨이

이 법칙에는 많은 의미가 내포되어있고, 그 중 하나는 ‘소프트웨어 구조가 커뮤니케이션 구조와 다르다면 어딘가 잘못된 구조다’는 의미로 해석될 수 있다.

결국 소프트웨어를 만드는 것은 사람이고, 코드를 변경해야 하는 요구사항도 사람간의 커뮤니케이션으로부터 접수된다. 이때 이상적인 소프트웨어 구조는 변경된 요구사항이 전달되었을 때 다른 컴포넌트에 영향(side-effect)이 가지 않도록 원하는 부분만 변경할 수 있는, 응집도가 높고 결합도가 낮은 컴포넌트의 집합 구조이다.

만약 조직간의 커뮤니케이션 방식과 소프트웨어 구조가 많이 다르다면 커뮤니케이션이 어려워 일정에 영향이 발생하거나, 얽힌 의존성 이슈로 손대기 힘든 구조가 될 가능성이 크다.

콘웨이 법칙에 입각해 소프트웨어 구조를 조직의 커뮤니케이션 구조와 유사하게 만들 수 있도록 도와주는 원칙이 바로 SRP(Single Responsibility Principle)원칙이다.

## SRP : Single Responsibility Principle

SOLID는 원칙이다. 기법(Technique)이 아니라 원칙(Principle)이기 때문에 기법으로 접근하면 범위와 목적 설정도 어긋날 수 밖에 없다. SRP를 직역하면 `단일책임원칙`이다.

책임=동작으로 잘못 해석하고 단일한 “동작”만 가진 컴토넌트로 쪼개야 한다고 오해할 수 있지만, 동작으로 쪼개다 보면 기준이 애매하기도 하고 과할정도로 자잘하게 컴포넌트가 쪼개질 수 있다. 컴포넌트를 너무 잘게 쪼개면 전체 로직을 한눈에 파악하기 어렵고, 코드 네비게이션에 들어가는 공수를 늘어나게 한다.

따라서 단일한 동작을 갖도록 코딩하는 것은 컴포넌트가 아니라 `순수한 함수` 한정이 되어야 한다. 이는 SRP 원칙으로 도달한 아키텍쳐 영역이라기 보다는 클린 코드의 영역으로 볼 수 있다.

SRP의 최종버전은 다음과 같다. 하나의 모듈은 하나의, 오직 하나의 액터에 대해서만 책임을 져야 한다. - 로버트 C. 마틴

모듈은 컴포넌트로 볼 수 있고, 액터는 사용자 또는 이해관계자 집단을 의미한다. 이 집단은 팀 단위가 될 수 있고 책무 단위가 될 수 있다. 중요한 점은 SRP의 **책임**이 의미하는 것을 소프트웨어 내부의 동작이나 논리가 아닌 조직간 커뮤니케이션 영역으로 봐야 한다는 점이다.

- 결론 : 따라서 SRP 원칙을 지닌다는 것은 컴포넌트를 설계할 때 요구사항을 전달하는 책무 단위로 설계한다는 것을 의미한다.

### 책무 기반 컴포넌트 설계

소프트웨어는 요구사항에 의해 만들어진다. 회사의 경우 요구사항은 여러 부서로부터 수립되고, 구현 중엔 개발 조직과 개발자 본인으로부터 요구사항이 수립된다. 그리고 컴포넌트 설계는 요구사항을 전달하는 책무 단위로 설립되어야 한다.

백엔드도 같은 맥락으로 요구사항 기반 설계를 하겠지만, 프론트엔드는 사용자와 맞닿아있는 영역이라 더 다채롭다. 각 영역의 요구사항을 명확히 파악하고, 영역을 구분해 의존성없는 **독립적인 컴포넌트로 만들어** 각 책무의 요구사항 변경에도 **사이드 이펙트 없이 유연하게 대처**할 수 있도록 설계하고 구현하는 것이 키포인트이다.

SOLID, 특히 SRP는 이러한 지향점에 도달하기 쉽게 해주는 패턴이라 볼 수 있다. 또한 프로젝트 아키텍쳐 설계 시 명확하게 컴포넌트를 나눌 수 있는 공신력 있는 기준이 될 수 있다.

기획과 디자인 조직이 분리되어있다면 각각의 컴포넌트로 분리되어야 한다. 만약 디자이너가 기획도 같이 진행하는 조직이라면 UI와 비즈니스가 결합된 컴포넌트로 만들어도 무방하다. 중요한 것은 컴포넌트와 요구사항의 변경 범위가 최대한 같도록 설계하는 것이다.

**기획(UX) 책무**

기획에선 주로 API에서 전달된 데이터를 어떤 디자인과 연결할지 정의하고 컴포넌트의 제약조건과 예외사항을 정의한다. 기획 조직에서는 와이어프레임이나 상세 기획서가 나오기 전 정보구조도(IA)를 먼저 작성한다.

![](/assets/img/2025-08-29/js-oop-frontend-solid-1.png)

정보구조도가 있으면 기획 책무의 컴포넌트 설계는 매우 편해진다. 일단 구조화된 그대로 네이밍도 맞춰서 만들면 되고, 그렇게 되면 단일 책무를 갖는 컴포넌트가 만들어지게 된다.

![](/assets/img/2025-08-29/js-oop-frontend-solid-2.png)

그리고 이 문서 그대로 컴포넌트를 작성한다. 네이밍도 그대로 직역해서 사용한다.

```javascript
function SingleSection({title, more, items}) {
	return (
			<Section>
				<Section.Title title={title}>
				<Section.More text={more}>
				<Section.items items={items}>
			</Section>
	)
}
```

기획서에는 각 컴포넌트의 요구사항도 존재한다. SRP를 만족하면 무엇을 테스트할지에 대한 문제도 자연스럽게 해결된다. 기획서에 적혀진 제약조건과 예외사항을 만족하는지 테스트하면 된다.

```javascript
describe('Section,Title', function () {
    idText('섹션 타이틀은 최대 15자 이내로 보여져야 합니다.', () => {
        const givenTitle = '15자가 넘으면 15자까지만 보여집니다.';
        const { getByText } = render(<Selection.title title={givenTitle} />);
        experimental_useEffectEvent(getByText(givenTitle)).not.beInDocument();
        experimental_useEffectEvent(getByText(givenTitle.substring(0, 15))).beInDocument();
    });
});
```

프로젝트 성격이나 규모에 따라 별도의 구조화 과정 없이 프로젝트를 진행하게 될 수도 있다. 이 경우 컴포넌트의 변경에 대한 책임이 어떻게 흘러가는지 커뮤니케이션 구조를 파악하고, 그에 맞게 설게한다면 운영에 도움이 될 수 있다.

### 디자인(UI) 책무

디자인에서 주는 요구사항은 명확한 UI 컴포넌트이다. 일반적으로 비즈니스 로직은 존재하지 않지만, 최근에 들어서 디자인에서도 UI를 컴포넌트화하려는 시도가 일어나고 있다. 이는 디자인 시스템인데, 디자인 팀의 성숙도가 높다면 디자인 시스템을 구축하고 있을 가능성이 높다. 만약 디자인 시스템이 없다면 조금 복잡해진다. 디자인에서 바라보는 화면과 실제 개발되는 컴포넌트의 구조가 다르다는 것은 유지보수 측면에서 매우 큰 허들이다.

디자인 시스템을 구축하면 응집도 높은 UI 컴포넌트들이 도출된다. 이렇게 도출된 컴포넌트를 개발에서 그대로 사용하기 위해선 **`아토믹 디자인`**을 차용하는게 도움이 된다.

아토믹 디자인은 UI영역의 SRP를 만족할 수 있게 해주는 중요한 개념이다. Atom, Molecule, Organism, Template, Page 각각이 하나의 책임(책무)를 담당할 수 있도록 구조화할 수 있게 해준다. 한가지 팁은 디자인 컴포넌트와 1:1로 매치되는 컴포넌트를 만드는 것이다. 여기엔 네이밍, Variant, 마진, 패딩 모두가 포함된다. 컴포넌트의 내부 패딩이나 컴포넌트 간 마진 또한 디자인 책무에서 단일하게 책임져야 하는 부붑이다.

[https://fe-developers.kakaoent.com/2022/220505-how-page-part-use-atomic-design-system/](https://fe-developers.kakaoent.com/2022/220505-how-page-part-use-atomic-design-system/)

## OCP : Open-Closed Principle

OCP는 요구사항이 변경될 때 기존 코드를 변경하는 것이 아니라, **새로운 코드를 추가**하는 방향을 추구하는 원칙이다. 아래는 컴포넌트 설계에 OCP를 적용하는 방법이다.

**⇒ 확장엔 열려있고, 수정엔 닫혀있다.**

이 말은 **새로운 기능을 추가할 때는 기존 코드를 수정하지 말고, 기존 코드를 "확장"하는 방식으로 해야 한다**는 거다.

![](/assets/img/2025-08-29/js-oop-frontend-solid-3.png)

해당 섹션은 OCP를 도입하기 좋은 케이스인데, 서비스 운영 도중 추가/변경/삭제되는 것이 매우 명확하다.

예) Backend API에서 다음과 같이 전달

```json
[{
  type: "BANNER",
  items: [...]
},
{
  type: "RECENTLY_VIEWED",
  items: [...]
}]
```

예1) OCP를 적용하지 않음

```javascript
sections.map((section) => {
    if (section.type === 'BANNER') {
        return section.items.map((item) => <Banner item={item} />);
    } else if (section.type === 'RECENTLY_VIEWED') {
        return section.items.map((item) => <PosterView item={item} />);
    }
});
```

OCP를 적용하지 않으면 코드가 다음과 같아질 것이다. 이 구조는 닫혀있는 구조이다. 만약 섹션이 추가되면 else-if를 추가해야 하기 때문이다. 만약 OCP를 만족해야 한다면 어떻게 해야 할까?

예2) 확장에 개방될 수 있도록 `다형성` 이용

```javascript
sections.map((section) => {
    <Section section={section}>
        {section.items.map((section) => (
            <Item section={section} item={item} />
        ))}
    </Section>;
});
```

확장에 개방될 수 있도록 다형성을 이용한 코드이다. 이제 섹션을 추가해도 코드는 변경이 없다.

하지만 기획 스펙이 중간에 광고를 노출해야 한다고 변경이 되고, API가 다음과 같이 변경이 된다고 가정을 하자

```json
[{
  type: "BANNER",
  items: [...],
  ad: "xxx",
  ad_index: 3
},
{
  type: "RECENTLY_VIEWED",
  items: [...]
}]
```

지금 구조라면 프론트엔드도 많은 변화가 일어나야 한다. 섹션 구조라면 이러한 변경은 당연히 일어날 수 밖에 없는 현상이라 미리 대비해야 하고, 섹션 구조의 OCP를 해결하기 위한 패턴 중 애플의 개발자 공식 문서에서 제안하는 Compositional Layout 패턴이 존재한다.

![](/assets/img/2025-08-29/js-oop-frontend-solid-4.png)

섹션을 일반화해보면 Sectino, Group, Item 형태로 대부분의 모습을 표현할 수 있는 컨셉이다. 아래는 애플의 이 컨셉을 차용해 API 스펙 변경에도 OCP를 만족하는 코드이다.

```javascript
const { factory } = require('typescript');

function generateSections(apiSection) {
    const sections = [];
    for (section in apiSection) {
        sections.push({
            type: section.type,
            groups: [
                {
                    items: section.items.map(factory),
                },
                ...(section.ad && {
                    items: [
                        {
                            type: 'AD',
                            key: section.ad,
                        },
                    ],
                }),
                ...(section.items_below_ad && {
                    items: section.items_below_ad.map(factory),
                }),
            ],
        });
    }
}

const sections = generateSections(apiSection);
sections.map((section) => (
    <Section section={section}>
        {section.groups.map((group) => (
            <Group group={group}>
                {group.items.map((item) => (
                    <Item item={item} />
                ))}
            </Group>
        ))}
    </Section>
));
```

## LSP : Liskov Substitution Principle

리스코프 치환 법칙을 설명할 때 일반적으로 클래스의 상속을 통해 설명하고 증명하는데, 아키텍처 관점에서는 좀 더 넒은 의미로 적용할 수 있다. 상속 관계를 얘기할 땐 간단하게 `is-a`를 만족하는지 여부로 상속 관계인지 판별할 수 있다. 리스코프 치환 법칙의 증명은 복잡하지만 얘기하고자 하는 것은 명확하다.  **“상속(is-a)로 이어진 관계에서 예상 못할 행동을 하지 말라”**

아래는 예상하지 못할 행동을 하는 예시이다

- GET method의 REST API로 정의했는데, 실제 동작에선 DB 상태를 변경
- API 응답주기로 약속한 모델을 화면마다 다르게 내려줌
- Label이라고 하고 체크박스 요구
- 필드명이 같은 것을 기준으로 타입스크립트 인터페이스 정의를 무지성으로 상속 관계로 만듦

실제로 SOLID원칙 중 장애상황, 버그와 가장 밀접하게 관련된 부분이 LSP원칙이다. 인터페이스나 상위 정의된 부분과 실제 구현된 부분이 예상과 다르다면 잘못 실행될 가능성이 높아진다. 이슈 발생 시 원인을 찾아내는 것도 어려워진다. 상속관계로 간주할 수 있는 명제가 예상과 다르게 동작하는 것이 LSP 위반이다.

코드 레벨로 예를 들면 `ApiErrorBoundary`라는 컴포넌트로 이름을 지었을 때 사용하는 입장에선 API 요청 중 발생하는 에러를 처리하는 컴포넌트구나라고 생각하고 사용하게 된다. 하지만 실제로 개발하다 보면 API와 상관없는 경우에 넣는 상황이 발생할 수 있다. 이렇게 되면 에러가 발생했을 때 매우 찾기 어려운 상황이 생기게 될 수 있다.

## ISP, DIP

ISP : Interface Segregation Principle

DIP : Dependency Inversion Principle

SRP를 SOLID의 시작과 끝이라고 한 이유가 여기서 나온다. 컴포넌트를 설계할 때 단일한 책무에 맞게 쪼개져 있다면 불필요한 속성 없는 독립된 컴포넌트로 구성될 수 있다. 하지만 컴포넌트만 있으면 페이지를 완성할 수 없는데, 페이지를 만드려면 컴포넌트들을 조합해야 한다. `ISP(인터페이스 분리 원칙)`과 `DIP(의존성 역전 법칙)`은 컴포넌트를 조합할 때 큰 도움을 주는 원칙이다.

아래는 티켓 정보와 키워드 정보 총 세개의 API를 통해서 그려주는 코드이다.

```javascript
function TicketInfoContainer() {
    const { isLoading, data: ticketInfo, error } = useTicketInfoQuery();
    const { data: commentInfo, error: commentApiError } = useCommentInfoQuery();
    const { data: keywordInfo } = useKeywordInfoQuery();

    if (isLoading) { // 너무 강한 결합 + 공통적으로 사용
        return <Loading />;
    }

    if (error) {
        return <Error />;
    }

    return (
        <TicketInfo // 너무 많은 prop들
            waitfreePeriod={ticketInfo.waitfreePeriod}
            waitfreeChargedDate={ticketInfo.waitfreeChargedDate}
            rentalTicketCount={ticketInfo.rentalTicketCount}
            ownTicketCount={ticketInfo.ownTicketCount}
            commentCount={commentInfo.commentCount}
            commentError={commentApiError /* 코멘트 에러 발생 시 재시도 버튼 추가해야함 */} // 잘못된 구조
            keywordInfo={keywordInfo}
        />
    );
}
```

1. Loading과 Error 처리가 TicketInfoContainer 컴포넌트 내부에 강하게 결합되어있다. ⇒ API 로딩, 에러는 보통 공통으로 정의하기에 여러 컴포넌트에서 중복될 것 같다.
2. API에러 발생 시 재시도 동작을 위해 TicketInfo 내부에 commentApiError props를 넘기고 있다. ⇒ 구조가 잘못된 방향으로 가고 있다.
3. TicketInfo 컴포넌트가 너무 많은 props를 가지고 있다. ⇒ 의존되는 곳이 많아 한번에 파악이 어렵고, 테스트코드도 만들기 어렵다.

결론 : SRP 원칙에 맞게 쪼개면 좋을 것 같다.

**이론 적용**

TicketInfoContainer 컴포넌트에서 로딩, 에러에 대한 부분만 쪼개려니 TicketInfo 컴포넌트와의 의존성이 걸린다. 로딩/에러 상태가 아니라면 TicketInfo 보여줘야 하는 의존성이 있는 상황이다. 이 경우에 의존성을 역전시켜 느슨하게 만들어주는 원칙이 DIP(의존성 역전 원칙)이다.

**리액트를 이용할 시 의존성을 역전**할 때 매우 유용한 단위가 있는데 `children`이다.

아래의 예시는 TicketInfoContainer 컴포넌트에서 API로딩, 에러부분만 분리한 후 `children`을 이용해 의존성 역전을 하는 예시이다.

예) `children`을 이용한 의존성 역전

```javascript
function Fetcher({query, children}) {
	const { isLoading, error, data } = query();
	if (isLoading) {
		return <Loading />
	}
	if (error) {
		return <Error />
	}
	
	return children;
}
```

간단한 실천으로 내부 구현으로부터 독립된 컴포넌트를 얻을 수 있다. 의존성이 역전되면 관심사만 테스트하기가 편해진다.

예) 테스트 코드

```javascript
describe('Fetcher', () => {
    it('chlidren이 없으면 데이터 요청만 하고 아무것도 렌더링하지 않는다.', async () => {
        const { container } = render(<Fetcher />);
        expect(container).toBeEmptyDOMElement();
    });

    it('children이 있으면 로딩중 컴포넌트를 보여줍니다.', async () => {
        render(<Fetcher>something</Fetcher>);
        expect(await screen.getByTestId(FETHCER_FETCHING_TEST_ID).toBeEmptyDOMElement());
    });
});
```

의존성을 느슨하게 관리함으로써 조립(Composition)이 가능하도록 만들어준다.

---

**ISP**

그 다음은 TicketInfoContainer에서 이미 의존성을 역전시킨 API 로딩중/에러 처리에 대한 부분을 제외한 영역에 인터페이스 분리 원칙(ISP)를 적용해 보는 코드이다. `Compound Pattern`으로 쪼개는게 가능하다. 하나의 TicketInfo 컴포넌트를 통해 모두 컨트롤하는 부분들을 Compound Pattern을 통해 기획 단위인 기다림정보/티켓정보/댓글정보로 나눌 수 있다.

```javascript
<TicketInfo>
    <TicketInfo.WaitfreeArea props={props} />
    <TicketInfo.TicketArea props={props} />
    <TicketInfo.CommentArea props={props} commentError={commentError} />
</TicketInfo>;
```

컴포넌트가 꼭 필요한 인터페이스만 받도록 변경되었다. 하지만 commentError가 분리되어야 하는데, DIP 원칙을 적용한 Fetcher 컴포넌트에서 API 로딩/에러 처리를 모두 다루는 것이 목표이었기에 TicketInfo의 의존성을 변경할 필요가 있다. API와 에러를 담당하는 Fetcher때문에 컴포넌트를 적용할 때 댓글 API에서 에러를 Fetcher에서 처리하도록 구현한다면 의존성이 더 잘 분리된다.

```javascript
function TicketInfoContainer() {
    return (
        <TicketInfoFetcher>
            <TicketInfo>
                <TicketInfo.WaitfreeArea props={props} />
                <TicketInfo.TicketArea props={props} />
                <CommentInfoFetcher>
                    <TicketInfo.CommentArea props={props} />
                </CommentInfoFetcher>
            </TicketInfo>
        </TicketInfoFetcher>
    );
}
```

### 결론

커뮤니케이션 구조를 기반으로

- 컴포넌트의 네이밍부터 시작해 각각에 맞는 책임, 책무를 정의하고(SRP)
- 변경하는 요구사항을 최소한의 코드 변경을 통해 유연하게 대응하고(OCP)
- `is-a`를 의도했다면 그 의도에 맞게 구현하도록 힘쓰고(LSP)
- 컴포넌트에 꼭 필요한 인터페이스만 의존할 수 있도록 구성하고(ISP)
- 의존성을 역전시켜 독립된 컴포넌트로 만들 수 있도록(DIP)

생각하고 개선해나가야 한다.
