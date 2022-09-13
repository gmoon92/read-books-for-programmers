# 7장 도메인 서비스

- 도메인 서비스
  - [여러 애그리거트가 필요한 기능](#여러-애그리거트가-필요한-기능)
  - [도메인 서비스](#도메인-서비스)
  - [도메인 서비스의 패키지 위치](#도메인-서비스의-패키지-위치)
  - [도메인 서비스의 인터페이스와 클래스](#도메인-서비스의-인터페이스와-클래스)

## 여러 애그리거트가 필요한 기능

여러 애그리거트로 기능을 구한할 경우가 있다.

- 애그리거트의 도메인 상태 변경은 주체가 필요하다.
- 애그리거트의 상태의 책임을 지닌 주체는 루트다.
- 그렇다면 여러 애그리거트로 기능을 구현할 때, 주체가 되는 애그리거트는?
- 사용 주체가 되는 기준이 애그리거트의 상태를 변경하는 주체가 된다.
- 한 애그리거트에 넣기에 애매한 도메인 기능을 특정 애그리거트에서 억지로 구현하면 안 된다.
  - 일반적으로 하나의 애그리거트 루트 엔티티에 모든 기능을 넣는 방법을 선택한다.
  - 이 경우 애그리거트는 자신의 책임 범위를 넘어서는 기능을 구현하게 된다.
  - 코드는 길어지게 되고 외부에 대한 의존이 높아지게 된다.
  - 코드의 복잡성은 높이고, 코드 수정을 어렵게 만드는 요인이다.

이런 경우 `도메인 서비스`를 별도로 구현해야 한다.

## 도메인 서비스

한 애그리거트에 넣기 애매한 도메인 개념을 구현하려면 도메인 서비스를 이용해서 도메인 개념을 명시적으로 드러내면 된다.

응용 영역의 서비스는 응용 로직을 다룬다면 도메인 서비스는 도메인 로직을 다룬다.

- 도메인 서비스는 상태 없이 로직만 구현한다.
- 도메인 서비스를 구현하는 데 `필요한 상태`는 애그리거트나 다른 방법으로 전달받는다.
- 도메인 서비스는 애그리거트의 상태를 변경하거나 애그리거트의 상태 값을 계산하는지 로직을 내포한다.
- 도메인 로직이면서 한 애그리거트에 넣기 적합하지 않을 경우 도메인 서비스로 구현한다.

```java
// Domain Service
public class DiscountCalculationService {

    // 특징 1. 상태 없이 행위만 존재
    // 도메인의 의미가 드러나는 용어를 타입과 메서드 이름으로 갖는다.
    public Moeny calculateDiscountAmounts(
            // 특징 2. 필요한 상태는 외부로 주입 받는다.
            List<OrderLine> orderLines,
            List<Coupon> coupons,
            MemberGrade grade) {
        // 여러 애그리거트를 이용하여 기능 구현 ...
        Moeny couponDiscount = coupons.stream()
                .map(this::calculateDiscount)
                .reduce(Moeny(0), (v1, v2) -> v1.add(v2));

        Moeny membershipDiscount = calculateDiscount(orderer.getMember().getGrade());

        return couponDisCount.add(membershipDiscount);
    }

}
```

- 서비스를 사용하는 주체는 애그리거트가 될 수도 있고 응용 서비스가 될 수도 있다.
- DiscountCalculationService를 다음과 같이 애그리거트의 결제 금액 계산 기능에 전달하면 사용 주체는 애그리거트가 된다.
- 도메인 서비스 객체를 애그리거트에 **의존 주입**하지 않는다.

> 의존 주입하지 않는다.: 에그리거트 도메인에 필드로 도메인 서비스를 명시하지 않는다.

```java
public class OrderService {

    // 도메인 서비스
    private DiscountCalculationService discountCalculationService;

    @Transactional
    public OrderNo placeOrder(OrderReqeust orderReqeust) {
        OrderNo orderNo = orderRepository.nextId();
        Order order = createOrder(orderNo, orderReqeust);

        orderRepository.save(order);
        
        // 응용 서비스 실행 후 표현 영역에서 필요한 값 리턴
        return orderNo;
    }

    public Order createOrder(OrderNo orderNo, OrderReqeust orderReqeust) {
        // create new domain
        Order order = new Order(
                orderNo,
                orderLines,
                shippingInfo
        );

        // DiscountCalculationService를 다음과 같이 
        // 애그리거트의 결제 금액 계산 기능에 전달하면 
        // 사용 주체는 애그리거트가 된다.
        // 단 도메인 서비스 객체를 애그리거트에 주입하지 않는다!.
        order.cacluateAmounts(discountCalculationService, member.getGrade());

        return order;
    }
}
```

- 도메인 서비스 객체를 애그리거트에 주입하지 않는다.
- 애그리거트가 도메인 서비스에 의존한다.
  - 애그리거트의 메서드를 실행할 때 도메인 서비스 객체를 파라미터로 전달
  - 그렇다하여 루트 도메인에 도메인 서비스를 의존 주입하지 않도록 설계해야 한다.
  - 의존성 흐름은 상위 계층에서 하위 계층으로 의존해야 된다.
  - 도메인 객체는 필드로 구성된 데이터와 메서드를 이용한 기능을 이용해서 개념적으로 하나인 모델을 표현한다.
  - 모델의 데이터를 담는 필드는 모델에서 중요한 구성요소다.
    - 도메인 서비스는 데이터 자체와는 관련이 없다.
    - 도메인 서비스는 다른 필드와 달리 영속화 대상이 아니다.
    - 일부 기능을 위해 굳이 애그리거트에 의존 주입할 이유는 없다.
    - 이는 개발자의 욕심을 채우는 것에 불과하다.

```java
// bad
@Entity
public class Order {
    
    @Autowired
    private DiscountCalculationService discountCalculationService;
}
```

## 도메인 서비스의 패키지 위치

도메인 서비스는 도메인 로직을 실행하므로 도메인 서비스 위치는 다른 도메인 구성 요소와 동일한 패키지에 위치한다.

- com.gmoon.order
  - infra
    - OrderService
  - domain
    - Order
    - OrderRepository
    - `DiscountCalculationService`

## 도메인 서비스의 인터페이스와 클래스

도메인 서비스의 로직이 고정되어 있지 않은 경우 도메인 서비스 자체를 인터페이스로 구현한다.

- 도메인 서비스의 구현이 특정 구현 기술에 의존적이거나 외부 시스템의 API 를 실행한다면 도메인 영역의 도메인 서비스는 인터페이스로 추상해야 한다.
- 도메인 영역이 특정 인프라스트럭처 영역의 구현에 종속되는 것을 방지할 수 있다.

특히 도메인 로직을 외부 시스템이나 별도 엔진을 이용해서 구현해야 할 경우에 인터페이스와 클래스를 분리하게 된다.

도메인 서비스에서 외부 시스템을 사용할 경우 도메인 영역에는 도메인 서비스 인터페이스가 위치하고 실제 구현은 인프라스트럭처 영역에 위치시킬 수 있다.

- infra
  - TossPaymentService
- domain
  - Order
  - OrderRepository
  - PaymentService
