# 4장 리포지터리와 모델구현(JPA 중심)

- [JPA를 이용한 리포지터리 구현](#JPA를-이용한-리포지터리-구현)
- [모듈 위치](#모듈-위치)
- [리포지터리 기본 기능 구현](#리포지터리-기본-기능-구현)
- [매핑 구현](#매핑-구현)
    - [엔티티와 밸류 기본 매핑 구현](#엔티티와-밸류-기본-매핑-구현)
    - [기본 생성자](#기본-생성자)
    - [필드 접근 방식 사용](#필드-접근-방식-사용)
    - [AttributeConverter를 이용한 밸류 매핑 처리](#AttributeConverter를-이용한-밸류-매핑-처리)
    - [밸류 컬렉션: 별도 테이블 매핑](#밸류-컬렉션:-별도-테이블-매핑)
    - [밸류 컬렉션: 한 개 컬럼 매핑](#밸류-컬렉션:-한-개-컬럼-매핑)
    - [밸류를 이용한 아이디 매핑](#밸류를-이용한-아이디-매핑)
    - [별도 테이블에 저장하는 밸류 매핑](#별도-테이블에-저장하는-밸류-매핑)
    - [밸류 컬렉션을 @Entity로 매핑하기](#밸류-컬렉션을-@Entity로-매핑하기)
    - [ID 참조와 조인 테이블을 이용한 단방향 M:N 매핑](#ID-참조와-조인-테이블을-이용한-단방향-M:N-매핑)
- [애그리거트 로딩 전략](#애그리거트-로딩-전략)
- [애그리거트의 영속성 전파](#애그리거트의-영속성-전파)
- [식별자 생성 기능](#식별자-생성-기능)

## JPA를 이용한 리포지터리 구현

애그리거트를 어떤 저장소에 저장해야 할까?

자바의 ORM 표준인 JPA를 이용해서 리포지터리와 애그리거트를 구현하는 방법에 대해 살펴본다.

## 모듈 위치

- 고수준 모듈: 리포지터리 인터페이스는 애그리거트와 같이 도메인 영역에 위치한다.
- 저수준 모듈: 리포티터리 인터페이스를 구현한 클래스
    - 인프라스트럭처 계층

- domain
  - Model
  - ModelRepository
- infra
  - JpaModelRepositoryImpl

팀 표준에 따라 리포지터리 구현 클래스를 도메인 패키지와 동일한 수준으로 배치하는 경우도 있다. 이는 리포지터리 인터페이스와 구현체를 분리하기 위한 타협안 같은 것이지 좋은 설계 원칙을 따르는 것은 아니다.

## 리포지터리 기본 기능 구현

리포지터리의 기본 기능은 다음의 두 가지이다.

- PK로 애그리거트 조회
- 애그리거트 저장

인터페이스는 애그리거트 루트를 기준으로 작성한다.

## 매핑 구현

### 엔티티와 밸류 기본 매핑 구현

애그리거트와 JPA 매핑을 위한 기본 규칙은 다음과 같다.

```java

@Entity
public class Order {

    @Embedded
    private OrderId id;

    @Embedded
    @AttributeOverrides({
            @AttributeOverride(
                    name = "id",
                    column = @Column(name = "member_id"))
    })
    private MemberId memberId;
}

@Embeddable
public class OrderId implements Serializable {

    private String id;
}

@Embeddable
public class MemberId implements Serializable {

    @Column(name = "id")
    private String id;
}
```

- 애그리거트 루트는 엔티티이므로 @Entity로 매핑 설정한다.
- 한 테이블에 엔티티와 밸류 데이터가 같이 있다면,
    - 밸류는 @Embeddable로 매핑 설정한다.
    - 밸류 타입 프로퍼티는 @Embedded로 매핑 설정한다.

### 기본 생성자

엔티티와 밸류 객체는 데이터 일관성을 유지하기 위해 Immutable 객체로 구성해야 한다.

따라서 기본 생성자를 추가할 필요가 없지만, JPA는 연관관계의 객체를 cglib 기반으로 Proxy 객체를 생성하여 지연 로딩을 구현하기 때문에 기본 생성자가 필요하다.

> 지연 로딩의 대상이 되는 @Entity, @Embeddable 의 기본 생성자는 private 가 아닌 protected 로 지정해야 한다.

### 필드 접근 방식 사용

JPA의 매핑 전략은 필드와 메서드의 두 가지 방식으로 나뉜다.

- @Id 어노테이션이 선언된 위치를 우선시 한다.
    - @Access 어노테이션으로 전략을 선택할 수 있다.
- 필드 매핑 전략: @Id 가 필드에 선언
- 메서드 맵핑 전략: @Id 가 메서드에 선언
    - getter/setter 메서드 구현

무의미한 setter 메서드는 캡슐화가 깨지는 원인이 됨으로 필드 매핑 전략을 따라야 한다.

### AttributeConverter를 이용한 밸류 매핑 처리

JPA 2.1 버전 이상부터 AttributeConverter 지원한다.

> JPA 2.0 버전에서 모델의 getter/setter 메서드를 재정의 하여 DB 데이터를 변환하는 로직 추가했었다.

```java
package javax.persistence;

public interface AttributeConverter<X, Y> {

    public Y convertToDatabaseColumn(X attribute);

    public X convertToEntityAttribute(Y dbData);
}

@Converter(autoApply = true)
public class MoneyConverter implements AttributeConverter<Moeny, Integer> {

    @Override
    public Integer convertToDatabaseColumn(Money money) {
        if (money == null) {
            return null;
        }

        return money.getValue();
    }

    @Override
    public Money convertToEntityAttribute(Integer value) {
        if (value == null) {
            return null;
        }

        return new Money(value);
    }
}
```

@Converter 애노테이션의 autoApply 속성 값을 true로 지정한 경우 모든 Money 타입의 프로퍼티에 대해 MoneyConverter를 자동으로 적용한다.

```java
public class Product {

    // 직접 컨버터 명시
    // @Convert(converter = MoneyConverter.class)
    @Column(name = "price")
    private Moeny price;
}
```

### 밸류 컬렉션: 별도 테이블 매핑

밸류 타입의 컬렉션은 별도 테이블에 보관한다.

- @ElementCollection
- @CollectionTable
    - 벨류를 저장할 테이블 지정
    - name: 테이블 이름
    - joinColumns: 외부 키 지정, 복합 키일 경우 @JoinColumn 사용
- @OrderColumn
    - 인덱스 값 저장

```java

@Entity
@Table
public class Order {

    @ElementCollection
    @CollectionTable(name = "order_line",
            joinColumns = @JoinColumn(name = "order_number"))
    @OrderColumn(name = "line_idx")
    private List<OrderLine> orderLines;

}

@Embeddable
public class OrderLine {

    @Embedded
    private ProductId productId;

    @Column(name = "price")
    private Money price;

    @Column(name = "quantity")
    private int quantity;

    @Column(name = "amounts")
    private Moeny amounts;

}
```

### 밸류 컬렉션: 한 개 컬럼 매핑

밸류 컬렉션을 별도 테이블이 아닌 한 개 컬럼에 저장해야할 경우

- AttributeConverter

```java

@Entity
public class Order {

    // converter 지정
    @Column(name = "emails")
    @Converter(converter = EmailSetConverter.class)
    private EmailSet emailSet;
}

// converter 구현
@Converter
public class EmailSetConverter implements AttributeConverter<EmailSet, String> {

    @Override
    public String convertToDatabaseColumn(EmailSet attribute) {
        if (attribute == null) {
            return null;
        }

        return attribute.getEmails().stream()
                .map(Email::toString)
                .collect(Collectors.joining(","));
    }

    @Override
    public EmailSet convertToEntityAttribute(String dbData) {
        if (dbData == null) {
            return null;
        }
        Set<Email> emailSet = Arrays.stream(emails)
                .map(Email::new)
                .collect(toSet());

        return new EmailSet(emailSet);
    }
}

// 밸류 구현
public class EmailSet {

    private Set<Email> emails = new HashSet<>();

    private EmailSet() {
    }

    private EmailSet(Set<Email> emails) {
        this.emails.addAll(emails);
    }

    public Set<Email> getEmails() {
        return Collections.unmodifiableSet(emails);
    }
}
```

### 밸류를 이용한 아이디 매핑

식별자는 최종적으로 문자열이나 숫자와 같은 기본 타입으로 기본 타입을 사용하는 것은 나쁘지 않지만 식별자라는 의미를 부각시키기 위해서 식별자 자체를 별도 밸류 타입으로 만들 수 있다.

```java

@Entity
@Table
public class Order {

    @EmbeddedId
    private OrderNo number;

}

@Embeddable
public class OrderNo implements Serializable {

    @Column(name = "order_number")
    private String number;

}
```

JPA 에서 식별자 타입은 Serializable 타입이어야 하므로 식별자로 사용될 밸류 타입은 Serializable 인터페이스를 상속받아야 한다.

### 별도 테이블에 저장하는 밸류 매핑

애그리거트에서 루트 엔티티를 뺀 나머지 구성요소는 대부분 밸류이다.

- 루트 엔티티 외에 또 다른 엔티티가 있다면 진짜 엔티티인지 의심해봐야 한다.
- 단지 별도 테이블에 데이터를 저장한다고 해서 엔티티인 것은 아니다.

밸류가 아니라 엔티티가 확실하다면 다른 애그리거트가 아닌지 확인해봐야 한다.

- 특히, 자신만의 독자적인 라이프사이클을 갖는다면 다른 애그리거트일 가능성이 높다.
    - 동일한 라이프사이클이라면 변경하는 주체가 같고, 도메인이 함께 생성되고, 수정된다.

애그리거트에 속한 객체가 밸류인지 엔티티인지 구분하는 방법은 고유 식별자를 갖는지 여부를 확인하는 것이다.

식별 관계의 하위 엔티티는 밸류다.

매핑되는 테이블의 식별자를 애그리거트 구성요소의 식별자와 동일한 것으로 착각하면 안 된다.

- 엔티티는 도메인의 지식을 투영한 객체다.
- 별도의 테이블로 저장되고 테이블에 PK가 있다고 해서 테이블과 매핑되는 애그리거트 구성요소가 고유 식별자를 갖는 것은 아니다.
- 대표적으로 식별 관계의 식별자는 DB 테이블간의 데이터를 연결하기 위함이지, 별도의 식별자가 필요하지 않은 경우가 있다.

```java

@Entity
@Table
@SecondaryTable(
        name = "user_option",
        pkJoinColumns = @PrimaryKeyJoinColumn(name = "id")
)
public class User {

    @Id
    private Long id;
    private String name;

    @AttributeOverrides({
            @AttributeOverride(name = "enabled",
                    column = @column(table = "user_option"))
    })
    private UserOption option;

}
```

- @SecondaryTable
    - name: 밸류를 저장할 테이블 지정
    - pkJoinColumns: 밸류 테이블에서 엔티티 테이블로 조진할 때 사용할 컬럼
    - `em.find(User.class, 1L);` UserOption 테이블을 조인하여 데이터 조회
- @AttributeOverride

### 밸류 컬렉션을 @Entity로 매핑하기

개념적으로 밸류인데 구현 기술의 한계나 팀 표준 때문에 @Entity를 사용해야 할 때도 있다.

예를 들어 계층 구조를 갖는 밸류 타입을 설계할 경우, JPA는 @Embeddable 타입의 클래스 상속 매핑을 지원하지 않는다.

상속 구조를 갖는 밸류 타입을 사용하려면 @Embeddable 대신 @Entity 를 이용한 상속 매핑으로 처리해야 한다.

- @Inheritance
- @DiscriminatorColumn

### ID 참조와 조인 테이블을 이용한 단방향 M:N 매핑

애그리거트간 집합 연관은 성능상의 이유로 피해야 한다.

그럼에도 불구하고 요구사항을 구현하는 데 집합 연관을 사용하는 것이 유리하다면 ID 참조를 이용한 단방향 집합 연관을 적용해 볼 수 있다.

```java
// 단방향 집합 연관
@Entity
public class Product {

    @EmbeddedId
    private ProductId id;

    @ElementCollection
    @CollectionTable(name = "product_category",
            joinColumns = @JoinColumn(name = "product_id"))
    private Set<CategoryId> categoryIdSet;
}

```

- 집합의 값에 밸류 대신 연관을 맺는 식별자가 온다는 점이다.
- @CollectionTable 를 사용하기 때문에 Product 를 삭제할 때 매핑에 사용한 조인 테이블의 데이터도 함께 삭제된다.
    - 에그리거트를 직접 참조하는 방식을 사용했다면 영속성 전파나 로딩 전략을 고민해야 하는데, ID 참조 방식을 사용함으로써 이런 고민을 할 필요가 없다.

## 애그리거트 로딩 전략

애그리거트의 상황에 맞게 글로벌 패치 전략을 선택해야 한다.

JPA 매핑 설정할 때 항상 기억해야 할 점은 애그리거트에 속한 객체가 모두 모여야 완전한 하나가 된다는 것이다.

- 즉 애그리거트 루트를 로딩하면 루트에 속한 모든 객체가 완전한 상태여야 함을 의미한다.
- 글로벌 패치 전략 -> FetchType.EAGER
    - 조회 시점에서 애그리거트를 완전한 상태가 되도록 하려면 애그리거트 루트에서 연관 매핑의 조회 방식을 즉시 로딩으로 설정하면 된다.

컬렉션 연관관계의 즉시 로딩 패치 전략은 카타시안 조인 문제로 성능상 문제가 될 수 있다.

> 카타시안 곱 문제는 쿼리 결과에 중복행이 발생하는 문제다.

애그리거트는 개념적으로 하나여야 한다.

- 상태를 변경하는 기능을 실행할 때 애그리거트 상태가 완전해야 한다.
- 표현 영역에서 애그리거트의 상태 정보를 보여줄 때 필요하기 때문이다.

그렇다고 루트 엔티티가 조회하는 시점에 애그리거트에 속한 모든 객체를 로딩해야 되는건 아니다.

표현 영역에서 애그리거트의 상태 정보를 조회할 컬럼만 별도로 조회 전용 기능을 구성할 경우가 많다.

실제적으로 애그리거트의 완전한 로딩 문제는 조회보단 상태 변경과 더 관련이 있다.

- 상태 변경 기능을 실행하기 위해 조회 시점에 즉시 로딩을 이용해서 에그리거트를 완전한 상태로 로딩할 필요는 없다.
- JPA 더티 체킹 개념은 트랜잭션 범위 내에서 지연 로딩을 허용하기 때문이다.
- 지연 로딩은 동작 방식이 항상 동일하기 때문에 즉시 로딩처럼 경우의 수를 따질 필요가 없는 장점이 있다.
    - 지연 로딩은 즉시 로딩보다 쿼리 실행 횟수는 많아질 가능성이 있다.
    - 즉시 로딩은 루트 엔티티 조회 시점에 카타시안 조인을 하여 조회한다.

## 애그리거트의 영속성 전파

애그리거트가 완전한 상태여야 한다는 것은 애그리거트 루트를 조회할 때뿐만 아니라 저장하고 삭제할 때도 하나로 처리해야 함을 의미한다.

- 저장 메서드는 애그리거트 루트만 저장하면 안 되고 애그리거트에 속한 모든 객체를 저장해야 한다.
- 삭제 메서드는 애그리거트 루트뿐만 아니라 애그리거트에 속한 모든 객체를 삭제해야 한다.

밸류 타입(Embeddable) 의 경우 같은 테이블 내에 함께 저장되고 삭제되므로 cascade 속성을 추가로 설정하지 않아도 된다.

반면 엔티티 타입(Entity) 의 경우 cascade 속성을 사용해서 저장과 삭제 시에 함께 처리되도록 설정해야 한다.

기본적으로 `@OneTo*` 어노테이션엔 cascade 속성의 기본값이 없으므로 설정해줘야 한다.

```java

@Entity
public class Product {

    @OneToMay(cascade = {CascadeType.PERSIST, CascadeType.REMOVE},
            orphanRemoval = true)
    @JoinColumn(name = "product_id")
    @OrderColumn(name = "list_dix")
    private List<Image> images = new ArrayList<>();
}
```

## 식별자 생성 기능

식별자는 크게 세 가지 방식 중 하나로 생성한다.

- 사용자가 직접 생성
- 도메인 로직으로 생성
- DB를 이용한 일련번호 사용(시퀀스 또는 데이터베이스 벤더에서 지원하는 증가 값 시스템 활용)

식별자 생성 규칙은 도메인 규칙이므로 도메인 영역에 식별자 생성 기능을 위치시켜야 한다.

- 응용 서비스 계층
    - 응용 서비스는 식별자 생성 모듈을 이용해서 식별자를 구하고 엔티티를 생성한다.
    - 별도의 식별자 생성 서비스 구성하여 활용
- 리파지터리 계층
    - 리파지터리 인터페이스에 식별자를 생성하는 메서드 추가 (ex. `nextId()`)
- DB의 자동 증가 컬럼 사용
    - @GeneratedValue 사용
