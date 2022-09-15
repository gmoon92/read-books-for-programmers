# 9장 도메인 모델과 BOUNDED CONTEXT

- BOUNDED CONTEXT
  - [도메인 모델과 경계](#도메인-모델과-경계)
  - [BOUNDED CONTEXT](#BOUNDED-CONTEXT)
  - [BOUNDED CONTEXT의 구현](#BOUNDED-CONTEXT의-구현)
- BOUNDED CONTEXT 간 통합과 관계
  - [BOUNDED CONTEXT 간 통합](#BOUNDED-CONTEXT-간-통합)
  - [BOUNDED CONTEXT 간 관계](#BOUNDED-CONTEXT-간-관계)
  - [컨텍스트 맵](#컨텍스트-맵)

## 도메인 모델과 경계

도메인 모델은 처음부터 완벽하게 표현하는 단일 모델로 만들려 하지 마라.

- 한 도메인은 여러 하위 도메인으로 구분된다.
- 한 개의 모델로 여러 하위 도메인을 모두 표현하려고 시도하게 되면 모든 하위 도메인에 맞지 않는 모델을 만들게 된다.

한 개의 모델로 모든 하위 도메인을 표현하려는 시도는 올바른 방법이 아니며 표현할 수도 없다.

- 사용자 모델
  - 회원 도메인 -> 회원
  - 주문 도메인 -> 주문자
  - 배송 도메인 -> 보내는 사람

논리적으로 같은 존재처럼 보이지만 하위 도메인마다 같은 용어라도 의미가 다르고, 다르게 해석하여 다른 용어를 사용하는 경우도 있다.

올바른 도메인 모델을 개발하려면 하위 도메인마다 모델을 만들어야 한다.

- 각 모델은 명시적으로 구분되는 경계를 가져서 섞이지 않도록 해야 한다.
- 여러 하위 도메인의 모델이 섞이기 시작하면 모델의 의미가 약해진다.
- 각 하위 도메인별로 다르게 발전하는 요구사항을 모델에 반영하기 어려워진다.

모델은 특정한 컨텍스트(문맥)하에서 완전한 의미를 갖는다.

- 하위 도메인은 완전히 구분되는 경계를 갖는 컨텍스트내에서 모델 바라보는 해석과 그 의미가 다르다.
- 구분되는 경계를 갖는 컨텍스트를 DDD에서는 BOUNDED CONTEXT라고 부른다.

## BOUNDED CONTEXT

BOUNDED CONTEXT는 모델의 경계를 결정한다.

- BOUNDED CONTEXT 는 실제로 사용자에게 기능을 제공하는 `물리적 시스템`으로, 도메인 모델은 BOUNDED CONTEXT 안에서 도메인을 구현한다.
- 한 개의 BOUNDED CONTEXT는 논리적으로 한 개의 모델을 갖는다.
- BOUNDED CONTEXT는 용어를 기준으로 구분한다.
  - 카탈로그 컨텍스트와 재고 컨텍스트는 서로 다른 용어를 사용하므로 용어 기준으로 컨텍스트를 분리할 수 있다.

조직 구조에 따라 BOUNDED CONTEXT가 결정된다.

![Start DDD!](/img/start-ddd/bounded-context.png)

_출처 Start DDD!_

- 이상적으로 하위 도메인과 BOUNDED CONTEXT가 일대일 관계를 가지면 좋겠지만 현실은 그렇지 않을 때가 많다.
- 하나의 BOUNDED CONTEXT 는 하나의 팀에만 할당되어야 한다.
  - 하나의 팀은 여러 개의 BOUNDED CONTEXT를 다룰 수 있다.
- 각각의 BOUNDED CONTEXT는 각각의 개발 환경을 가질 수 있다.
- 용어를 명확하게 하지 못해 두 하위 도메인을 한 BOUNDED CONTEXT 에서 구현하기도 한다.

여러 하위 도메인을 한 개의 BOUNDED CONTEXT에서 구현할 수도 있다.

여러 하위 도메인을 하나의 BOUNDED CONTEXT에서 개발할 때 주의할 점은 하위 도메인마다 구분되는 패키지를 갖도록 구현하여 하위 도메인의 모델이 뒤섞이지 않도록 하는 것이다.

- shop
  - order
  - catalog
  - member

내부적으로 패키지를 활용해서 논리적으로 BOUNDED CONTEXT를 구성한다.

- BOUNDED CONTEXT 는 도메인 모델을 구분하는 경계가 된다.
- BOUNDED CONTEXT 는 각자 구현하는 하위 도메인에 맞는 모델을 갖는다.
- 예를 들어 사용자 모델은 각 컨텍스트마다 갖는 모델이 달라진다.
  - 회원 도메인 -> 애그리거트 루트
  - 주문 도메인 -> Orderer 밸류

## BOUNDED CONTEXT의 구현

BOUNDED CONTEXT가 도메인 모델만 포함하는 것은 아니다.

BOUNDED CONTEXT는 도메인 기능을 제공하는 데 필요한 요소를 모두 포함한다.

- 표현 영역
- 응용 서비스
- 도메인
  - 도메인 모델
  - 도메인 기능
  - 도메인 서비스
- 인프라스트럭처
  - 테이블

모든 BOUNDED CONTEXT를 반드시 도메인 주도로 개발할 필요는 없다.

각 BOUNDED CONTEXT는 도메인에 알맞은 아키텍처를 사용한다.

![Start DDD!](/img/start-ddd/bounded-context2.png)

_출처 Start DDD!_

- BOUNDED CONTEXT는 서로 다른 구현 기술을 사용할 수도 있다.
- BOUNDED CONTEXT가 반드시 사용자에게 보여지는 UI를 가져야 하는 것은 아니다.

## BOUNDED CONTEXT 간 통합

기존 BOUNDED CONTEXT 내에 새로운 BOUNDED CONTEXT가 생길 수 있다.

- 두 BOUNDED CONTEXT간 통합이 필요하다.
- 각 BOUNDED CONTEXT의 하위 도메인 모델은 서로 다르다.
  - 각 도메인은 표현 방식이 다르다
    - 간접 참조로 ID 객체
- 도메인 관점에서 도메인 서비스 인터페이스를 정의한다.
- 도메인 서비스를 구현한 클래스는 인프라스트럭처 영역에 위치한다.
  - 외부 연동을 위한 도메인 서비스 구현 클래스는 도메인 모델과 외부 시스템 간의 모델 변환을 처리한다.
  - 다른 BOUNDED CONTEXT의 도메인 기능을 호출하기 위함이다.
  - BOUNDED CONTEXT 다르다는 것은 다른 시스템으로 분리할 수 있기 때문이다.

다른 BOUNDED CONTEXT 도메인의 데이터를 자신의 모델 데이터로 변환하는 작업이 필요하다. 통합 방법은 다음과 같다.

- 직접 통합 방법
  1. REST API
     - convertTo: item data -> domain
     - Translator Object
- 간접 통합 방법
  1. 메시지 큐
    - 비동기 방식
    - 메시지 시스템을 관리할 BOUNDED CONTEXT를 정해야한다.
    - 큐에 담을 메시지 데이터 구조의 협의가 필요하다.

MSA와 BOUNDED CONTEXT

- MSA에서 개별 서비스를 독립된 프로세스로 실행하고 각 서비스가 REST API나 메시징을 이용해서 통신하는 구조를 갖는다. 
- 별도 프로세스로 개발한 BOUNDED CONTEXT는 독립적으로 배포하고 모니터링하고 확정하게 된다.

## BOUNDED CONTEXT 간 관계

각 BOUNDED CONTEXT는 다양한 방식으로 관계를 맺는다.

- 직접 통합: REST API
- 간접 통합: 메시징 큐를 이용한다.

일반적으로 가장 흔한 관계는 한쪽에서 API를 제공하고 다른 한쪽에서 그 API를 호출하는 관계다.

- 상류(upstream) 컴포넌트
  - 데이터를 제공하는 주체
  - 서비스 공급자(호스트) 역할
  - 공개 호스트 서비스(OHS, OPEN HOST SERVICE)
    - 하류 컴포넌트가 사용할 수 있는 통신 프로토콜 정의 및 공개
    - 서비스의 일관성을 유지하기 위해, 여러 하류 컴포넌트의 요구사항을 수용할 수 있는 API 를 단일 서비스 형태로 공개한다.
    - ex) 검색 엔진
- 하류(downstream) 컴포넌트
  - 데이터를 받는 주체
  - 서비스 사용자 역할
  - 안티코럽션 계층(ACL, Anti-corruption Layer)
    - 각 BOUNDED CONTEXT 모델을 변환하는 작업을 한다. 자신의 도메인 모델을 침범하지 않도록 막아주는 역할
    - 상류 서비스의 모델이 자신의 도메인 모델에 영향을 주지 않도록 보호해 주는 완충 지대를 만들어야 한다.

고객과 공급자 관계에 있는 각 BOUNDED CONTEXT는 상호 협력 관계를 이룬다.

- 기능을 개발하거나 결정을 해야할 때, 개발 계획을 서로 공유하고 일정을 협의해서 결정해야 한다.

두 BOUNDED CONTEXT가 같은 모델을 공유하는 경우도 있다.

- 공유 커널(SHARED KERNEL)
  - 각 BOUNDED CONTEXT 에서 공유하는 도메인 모델
  - 각각의 BOUNDED CONTEXT에서 같은 모델을 공유함으로써 도메인의 중복 개발을 막을 수 있다.
  - 두 팀에서 동일한 모델을 공유하기 때문에, 협의를 통한 개발이 필요하다.

마지막으로 살펴볼 관계는 독립 방식(SEPERATE WAY) 관계다.

- 독립 방식(SEPARATE WAY) 관계
  - 그냥 서로 통합하지 않는 방식이다.
  - 두 BOUNDED CONTEXT 간에 통합을 하지 않으므로 서로 독립적으로 모델을 발전시킨다.
  - 독립 방식에서 통합은 수동으로 이뤄진다.
  - 서비스 규모가 커지면, 결국 통합해 주는 별도의 시스템을 구축해야 된다.

## 컨텍스트 맵

전체 BOUNDED CONTEXT 의 비즈니스를 조망할 수 있는 지도가 필요한데, 그것이 바로 컨텍스트 맵이다.

컨텍스트 맵은 BOUNDED CONTEXT 간의 관계를 표시한 것이다. 컨텍스트 맵은 전체 시스템의 이해 수준을 보여준다.

즉, 시스템을 더 잘 이해하거나 시간이 지나면서 컨텍스트 간 관계가 바뀌면 컨텍스트 맵도 함께 바뀐다.

- BOUNDED CONTEXT 의 경계가 명확하게 드러나야 한다.
- BOUNDED CONTEXT 의 관계를 표현해야 한다.
  - 하류 컴포넌트와 상류 컴포넌트 관계
    - 오픈 호스트 서비스(OHS)와 안티코럽션 계층(ACL)
  - 공유 커널(SHARED KERNEL)이 존재하는 관계
  - 독립 방식(SEPARATE WAY) 관계
- BOUNDED CONTEXT 영역에 주요 애그리거트를 함께 표시하여 모델에 대한 관계를 명확히 드러나도록 한다.
- BOUNDED CONTEXT의 하위 도메인이나 조직 구조를 함께 표시하면 도메인을 포함한 전체 관계를 이해하는 데 도움이 된다.

컨텍스트 맵은 시스템의 전체 구조를 보여준다.

- 하위 도메인과 일치하지 않는 BOUNDED CONTEXT를 찾아 도메인에 맞게 BOUNDED CONTEXT를 조절하고 사업의 핵심 도메인을 위해 조직 역량을 어떤 BOUNDED CONTEXT에 집중할지 파악하는 데 도움을 준다.
- 컨텍스트 맵을 표현하는 방식 규약은 없다.
  - 각 컨텍스트의 관계를 이해할 수 있는 수준에서 표현하면 된다.
