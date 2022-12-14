# 1장 도메인 모델 시작

- 도메인 모델
  - [도메인](#도메인)
  - [도메인 모델](#도메인-모델)
  - [도메인 모델 패턴](#도메인-모델-패턴)
  - [도메인 모델 도출](#도메인-모델-도출)
- 엔티티와 밸류
  - [엔티티와 밸류](#엔티티와-밸류)
    - [엔티티](#엔티티)
    - [밸류 타입](#밸류-타입)
    - [엔티티 식별자와 밸류 타입](#엔티티-식별자와-밸류-타입)
    - [도메인 모델에 set 메서드 넣지 않기](#도메인-모델에-set-메서드-넣지-않기)
- 도메인 용어
  - [도메인 용어](#도메인-용어)

## 도메인

도메인이란 소프트웨어로 해결하고자 하는 문제 영역이다.

- 문제 영역을 해결하기 위해 여러 도메인이 존재한다.
- 도메인들은 서로 협력하여 완전한 기능을 제공한다.

특정 도메인을 위한 소프트웨어라고 해서 도메인이 제공해야 할 모든 기능을 구현하는 것은 아니다.

때론, 일부는 자체 시스템에서 기능을 구현하고 외부 시스템의 기능을 사용하여 구현하기 때문이다.

- 도메인마다 고정된 하위 도메인이 존재하는 것은 아니다.
- 하위 도메인은 상황에 따라 구성하는 방식이 다르다.
  - 각 하위 도메인이 다루는 영역이 서로 다르기 때문에 같은 용어라도 하위 도메인마다 의미가 달라질 수 있다.
  - 주문 도메인
    - 결제 팀: 주문은 결제다.
    - 배달 팀: 주문은 배달이다.
    - 가맹점 팀: 주문은 발행이다.

## 도메인 모델

도메인 모델이란 특정 도메인을 개념적으로 표현하는 수단으로, 도메인 자체를 이해하기 위한 개념 모델이다.

도메인 모델은 여러 이해 관계자(도메인 전문가)들이 동일한 모습으로 도메인을 이해하고 도메인 지식을 공유하는 데 목적이 있다.

도메인 모델을 표현할 때는 모든 도메인 관계자가 이해할 수 있는 표현 방식을 선택해야 한다.

- 객체 기반 도메인 모델링
- 시퀀스 기반 도메인 모델링
- 상태 다이어그램 도메인 모델링
- UML 표기법 도메인 모델링
- 그래프 기반 도메인 모델링

도메인 모델링 단계는 기능 구현에 앞서 도메인 자체를 이해하기 위한 논리적인 단계임으로 구현에 필요한 구현 모델이 필요하다.

개념 모델과 구현 모델은 서로 다른 것이지만 구현 모델이 개념 모델을 최대한 따르도록 할 수 있다. 

- 중요한 점은 도메인 모델링을 통해 구현 모델을 도출하는 일련의 과정이 필요하다.
- 구현 모델은 개념 모델의 연장선선이다.
  - 개념 모델의 도메인 자체를 바라보는 개념, 즉 구현 기능은 개념과 동일시 해야되기 때문이다.

## 도메인 모델 패턴

도메인 모델은 개념 모델을 의미하지만, 도메인 모델 패턴에서 도메인 모델은 아키텍처상의 도메인 계층을 객체 지향 기법으로 구현하는 패턴(구현 모델)을 의미한다.

일반적인 애플리케이션의 아키텍처는 네 개의 계층으로 구성된다.

- 표현(Presentation 또는 사용자 인터페이스(UI))
  - 사용자의 요청을 처리하고 사용자에게 정보를 보여준다.
  - 소프트웨어를 사용하는 사용자 또는 외부 시스템의 사용자
- 응용(Application)
  - 사용자가 요청한 기능 실행
  - 업무 로직을 직접 구현하지 않고, 도메인 계층을 조합해서 기능 구현
- 도메인(Domain)
  - 시스템이 제공할 **도메인의 규칙 구현**
- 인프라스트럭처(Infrastructure)
  - 데이터베이스나 메시징 시스템과 같은 외부 시스템과의 연동 처리

도메인 계층은 도메인의 핵심 규칙을 구현한다.

- 상태 검증
- 상태 흐름
- 상태 변경
- 상태 저장

핵심 규칙은 응용 계층에서 기능을 구현하는 것이 아닌, 업무 규칙에 따라 도메인에서 구현해야 된다.

- SRP, 핵심 규칙을 구현한 코드는 도메인 모델에만 위치한다.
- 유연한 확장 가능
  - OCP, 규칙을 확장해야 할 때 다른 코드에 영향을 덜 주고 변경 내역을 모델에 반영할 수 있다.

### 개념 모델 vs 구현 모델

완벽한 개념 모델을 만들기보다는 전반적인 개요를 알 수 있는 수준으로 개념 모델을 작성해야 한다.

도메인에 대한 전체 윤곽을 이해하는데 집중하고, 구현하는 과정에서 개념 모델을 구현 모델로 점진적으로 발전시켜 나가야 한다.

- 개념 모델: 순수하게 문제를 분석한 결과물
  - 프로젝트 초기에 참여하는 모든 도메인 전문가의 도메인 지식을 공유하기 위해 사용
  - 데이터베이스, 트랜잭션 처리, 성능, 구현 기술과 같은 것들은 고려하지 않는다.
  - 실제 구현에서 개념 모델을 그대로 사용할 수 없다.
  - 구현 가능한 형태의 모델로 전환하는 과정이 필요하다.
- 구현 모델: 개념 모델의 도메인 지식을 기반으로 실제 구현에 맞게 도출한 결과물
  - 완벽한 구현 모델은 없다.
  - 도메인에 대한 새로운 지식이 쌓이면서 모델을 보완하거나 수정하는 일이 발생한다.

## 도메인 모델 도출

구현에 앞서 도메인에 대한 이해가 필요하다.

- 구현에 앞서 도메인 모델 초안을 만들어야 비로소 코드를 작성할 수 있다.
- 구현을 시작하려면 도메인에 대한 초기 모델이 필요하다.

도메인을 모델링할 때 기본이 되는 작업은 모델을 구성하는 핵심 구성요소, 규칙, 기능을 찾는 것이다.

- 요구사항을 기반으로 도메인 모델을 도출한다.
  - 구성요소 (상태)
  - 규칙 (제약 조건)
  - 기능 (행위)
  - 하위 도메인과의 관계

> 특정 조건이나 상태에 따라 제약이나 규칙이 달리 적용되는 경우도 존재한다.

요구사항을 정련을 위해 도메인 전문가나 다른 개발자와 논의하는 과정에서 공유하기도 한다.
도출된 도메인 모델은 문서화가 필요하다.

- 문서화를 하는 주된 이유는 지식을 공유하기 위함이다.
- 코드는 상세한 모든 내용을 다루고 있기 때문에 코드를 이용해서 전체 소프트웨어를 분석하려면 많은 시간을 투자해야 한다.
- 코드도 문서화의 대상이 된다.
- 도메인 지식이 잘 묻어나도록 코드를 작성해야한다.

## 엔티티와 밸류

도출한 도메인 모델은 크게 엔티티(Entity)와 밸류(Value)로 구분한다.

### 엔티티

엔티티는 고유한 식별자를 갖는다.

엔티티의 식별자 생성 방식은 다음과 같다.

- 특정 규칙에 따라 생성
- UUID 사용
- 값을 직접 입력
- 일련 번호 사용(시퀀스나 DB의 자동 증가 컬럼 사용)

### 밸류 타입

밸류 타입은 개념적으로 완전한 하나를 표현할 때 사용한다.

- 밸류 타입은 데이터를 표현하기 위한 수단이다.
- 단순한 값이 아닌 도메인의 데이터의 의미를 부여한다.
- Immutable 객체로 구현한다.

### 도메인 모델에 set 메서드 넣지 않기

도메인 모델에 getter/setter 를 지양하자.

- 도메인 모델의 행위는 도메인 지식을 투영한다.
- 무의미한 getter/setter 메서드는 도메인의 핵심 개념이나 의도를 코드에서 사라지게 한다.
  - setShippingInfo -> changeShippingInfo
  - 단순히 값을 설정한다는 의미에서 새로 변경한다는 의미로 도메인 지식을 반영한 이름으로 짓자.

## 도메인 용어

도메인에서 사용하는 용어를 코드에 반영한다.

- 도메인 용어를 코드에서 사용하여 불필요한 변환 과정(소스를 이해하는 과정)을 없애라.
- 의미를 변환하는 과정에서 발생하는 버그도 줄어들게 된다.
- 가독성을 높이기 위함이다.
