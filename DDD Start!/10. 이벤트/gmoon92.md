# 10장 이벤트

- 이벤트의 용도와 장점
  - [시스템 간 강결합의 문제](#시스템-간-강결합의-문제)
  - [이벤트 개요](#이벤트-개요)
  - [이벤트 관련 구성요소](#이벤트-관련-구성요소)
  - [이벤트의 구성](#이벤트의-구성)
  - [이벤트 용도](#이벤트-용도)
  - [이벤트 장점](#이벤트-장점)
- 핸들러 디스패치와 핸들러 구현
  - [이벤트, 핸들러, 디스패처 구현](#이벤트,-핸들러,-디스패처-구현)
  - [이벤트 클래스](#이벤트-클래스)
  - [EventHandler 인터페이스](#EventHandler-인터페이스)
  - [이벤트 디스패처인 Events 구현](#이벤트-디스패처인-Events-구현)
  - [흐름 정리](#흐름-정리)
  - [AOP를 이용한 Events.reset() 실행](#AOP를-이용한-Events.reset()-실행)
- 비동기 이벤트 구현
  - [비동기 이벤트 처리](#비동기-이벤트-처리)
    - [로컬 핸들러를 비동기로 실행](#로컬 핸들러를 비동기로 실행)
    - [메시지 시스템을 이용한 비동기 구현](#메시지-시스템을-이용한-비동기-구현)
    - [이벤트 저장소를 이용한 비동기 처리](#이벤트-저장소를-이용한-비동기-처리)
    - [이벤트 저장소와 이벤트 제공 API 사용하기](#이벤트-저장소와-이벤트-제공-API-사용하기)
- [이벤트 적용시 추가 고려사항](#이벤트-적용시-추가-고려사항)

## 시스템 간 강결합의 문제

시스템적으로 분리된 BOUNDED CONTEXT 를 기능적으로 통합할 경우 시스템 간 강결합의 문제가 생긴다.

시스템 간 강결합(high coupling) 문제는 다음과 같다.

- 외부 서비스가 정상이 아닐 경우 트랜잭션 처리를 어떻게 해야 할지 애매하다.
  - 트랜잭션을 커밋해야 될 경우
  - 트랜잭션을 롤백해야 될 경우
  - 이외에도 트랜잭션 처리의 복잡도가 증가한다.
- 성능에 대한 문제
  - 외부 서비스 성능에 직접적인 영향을 받는다.
  - 응답 시간만큼 대기 시간이 늘어난다.
- 설계상 문제
  - 같은 애그리거트가 아닌 다른 애그리거트의 도메인 지식이 섞이게 된다.
  - 도메인 객체에 서로 다른 도메인 로직이 섞이는 문제가 발생한다.
  - 다른 BOUNDED CONTEXT 의존적이게 되므로 기능 수정 및 확장에 어려움이 있다.

이벤트를 사용하면 강결합 문제를 해결할 수 있다.

특히 **`비동기 이벤트`** 를 사용하면 두 시스템 간의 결합을 크게 낮출 수 있다.

## 이벤트 개요

이벤트(event) 라는 용어는 `과거에 벌어진 어떤 것`을 뜻한다.

- 이벤트가 발생했다.
  - 과거에 어떤 행위가 벌어졌다.
- 과거에 어떤 도메인의 상태가 변경됐다.
  - 이벤트가 발생한다는 것은 상태가 변경됐다는 것을 의미한다.
- 도메인의 상태 변경에 따른 동작을 수행한다.
  - 이벤트가 발생하면 그 이벤트에 반응하여 원하는 동작을 수행하는 기능을 구현해야 한다.

도메인 모델에서도 도메인의 상태 변경을 이벤트로 표현할 수 있다.

- `~할 때`
- `~가 발생하면`
- `만약 ~하면`

이런 요구사항을 이벤트를 이용해서 구현할 수 있다.

## 이벤트 관련 구성요소

이벤트는 도메인 객체가 도메인 로직을 실행해서 도메인의 상태가 바뀌면 발생한다.

도메인 모델에 이벤트를 도입하려면 네 개의 구성요소를 구현해야 한다.

```text
       event             event
domain -----> dispatcher -----> handler
```

- 이벤트 생성 주체
  - 모든 도메인 객체
    - 엔티티
    - 벨류
    - 도메인 서비스
- 이벤트 디스패처(dispatcher, 이벤트 퍼블리셔)
  - 여러 이벤트 핸들러를 등록하고 관리한다.
  - 이벤트 생성 주체와 이벤트 핸들러를 연결해 준다.
  - 발생한 이벤트를 핸들러에게 전파한다.
  - 디스패처 구현 방식에 따라 동기나 비동기로 실행한다.
- 이벤트 핸들러(handler, 이벤트 구독자)
  - 발생한 이벤트를 처리한다.
  - 이벤트 생성 주체가 발생한 이벤트에 반응한다.
  - 핸들러는 생성 주체가 발생한 이벤트를 전달받아 이벤트에 담긴 데이터를 이용해서 원하는 기능을 실행한다.

## 이벤트의 구성

이벤트는 발생한 이벤트에 대한 정보를 담는다.

- 이벤트의 종류: 클래스 이름으로 이벤트 종류를 표현
- 이벤트 발생 시간
- 추가 데이터: 이벤트와 관련된 정보

```java
// 이벤트 클래스 (이벤트 종류)
public class ShippingInfoChangedEvent {

    // 이벤트 발생 시간
    private long timestamp;

    // 추가 데이터(이벤트와 관련된 정보)
    private String orderNumber;
    private ShippingInfo newShippingInfo;

    // constructor, getter...
}
```

- 이벤트 클래스 네이밍은 과거 시제 사용한다.

```java
// 배송지 변경 이벤트의 주체
public class Order {

    public void changShippingInfo(ShippingInfo newShippingInfo) {
        verifyNotYetShipped();
        setShippingInfo(newShippingInfo);

        // event 발생 
        // 디스패처(Event.raise())로 이벤트 전달
        Event.raise(new ShippingInfoChangedEvent(number, newShippingInfo));
    }

}

// 이벤트 핸들러
public class ShippingInfoChangedHandler
        implements EventHandler<ShippingInfoChangedEvent> {

    @Override
    public void handle(ShippingInfoChangedEvent evt) {
        // 이벤트가 필요한 데이터를 담고 있지 않으면,
        // 이벤트 핸들러는 리포지터리, 조회 API, 직접 DB 접근 동의
        // 방식을 통해 필요한 데이터를 조회해야 한다.
        Order order = orderRepository.findById(evt.getOrderNo());
        shippingInfoSynchronizer.sync(
                order.getNumber(),
                evt.getNewShippingInfo()
        );
    }

}
```

- 이벤트는 이벤트 핸들러가 작업을 수행하는 데 필요한 최소한의 데이터를 담아야 한다.
  - 추가적인 필요한 데이터는 API를 호출하거나 DB에서 데이터를 직접 읽어온다.
- 이벤트 자체와 관련 없는 데이터를 포함할 필요는 없다.

## 이벤트 용도

이벤트는 크게 두 가지 용도로 쓰인다.

- 트리거
  - 이벤트는 다른 기능을 실행하는 트리거가 된다.
  - 도메인의 상태가 바뀔 때 다른 후처리를 해야 할 경우 후처리를 실행하기 위한 트리거로 이벤트를 사용할 수 있다.
- 서로 다른 시스템 간의 데이터 동기화
  - 각각의 BOUNDED CONTEXT 의 애그리거트의 상태를 동기화하는 목적으로 사용된다.

## 이벤트 장점

- 이벤트는 서로 다른 도메인 로직이 섞이는 것을 방지할 수 있다.
  - 도메인엔 특정 디스패처에게 이벤트를 전달하고, 디스패처는 이벤트 핸들러에게 이벤트를 전파한다.
  - 이벤트 핸들러에서 다른 도메인 로직을 구성한다.
- 도메인 로직에 영향 없이 기능 확장 가능하다.
  - 이벤트 핸들러를 사용하여 기능을 확장한다.

## 이벤트, 핸들러, 디스패처 구현

- 이벤트 클래스
- EventHandler
  - 이벤트 핸들러를 위한 상위 타입으로 모든 핸들러는 이 인터페이스를 구현한다.
- Events
  - 이벤트 디스패처
  - 이벤트 발행, 이벤트 핸들러 등록, 이벤트를 핸들러에 등록하는 등의 기능을 제공한다.

## 이벤트 클래스

- 이벤트 자체를 위한 상위 타입은 존재하지 않는다.
  - 예외적으로 모든 이벤트가 공통으로 갖는 프로퍼티가 존재한다면 관련 상위 클래스를 구성할 수 있다. ex. 이벤트 발생 시간
- 이벤트 클래스 이름은 과거 시제로 사용해야 한다.
  - `Canceled`**`Event`**
    - 접미사로 Event를 사용해서 이벤트로 사용하는 클래스라는 것을 명시적으로 표현
  - Order`Canceled`
    - 간결함을 위해 과거 시제만 사용
    - 이벤트 주체가 되는 도메인 명을 함께 사용한다.
- 이벤트를 처리하는 데 필요한 최소한의 데이터를 포함해야 한다.

## EventHandler 인터페이스

이벤트는 다른 BOUNDED CONTEXT 와 데이터 동기화나 외부 시스템의 기능을 트리거하기 위해 사용한다.

인프라스트럭처 영역과의 결합도를 낮추기 위해, 이벤트 핸들러는 별도의 인터페이스를 구성해야 한다.

```java
public interface EventHandler<T> {

    void handler(T event);

    // 핸들러가 이벤트를 처리할 수 있는지 여부 검사한다.
    default boolean canHandler(Object event) {
        Class<?>[] typeArgs = TypeResolver.resolveRawArguments(EventHandler.class, this.getClass());
        return typeArgs[0].isAssignableFrom(event.getClass());
    }
}
```

## 이벤트 디스패처인 Events 구현

도메인을 사용하는 응용 서비스는 이벤트를 받아 처리할 핸들러를 Events.handler()로 등록하고, 도메인 기능을 실행한다.

```java
public class CancelOrderService {

    private OrderRepository orderRepository;
    private RefundService refundService;

    @Transactional
    public void cancel(OrderNo orderNo) {
        // 이벤트 핸들러에게 이벤트 전파
        // OrderCanceledEvent 이벤트 발생
        // Events는 내부적으로 핸들러 목록을 유지하기 위해 ThreadLocal을 사용한다.
        Events.handle(
                (OrderCanceledEvent evt) -> refundService.refund(evt.getOrderNumber())
        );

        Order order = findOrder(orderNo);
        order.cancel();

        // ThreadLocal 변수를 초기화해서
        // OOME가 발생하지 않도록 함
        Events.reset();
    }
}

public class Order {

    public void cancel() {
        verifyNotTetShipped();
        this.status = OrderStatus.CANCELED;

        // 이벤트 발생
        Events.raise(new OrderCanceledEvent(number.getNumber()));
    }
}

// 이벤트 디스패처
public class Events {

    // EventHandler 목록 관리 보관
    private static ThreadLocal<List<EventHandler<?>>> handlers
            = new ThreadLocal();
    // 이벤트를 처리 중인지 여부를 보관
    private static ThreadLocal<Boolean> publishing
            = new ThreadLocal<Boolean>() {

        @Override
        protected Boolean initialValue() {
            return Boolean.FALSE;
        }
    };

    // 이벤트 처리
    public static void raise(Object event) {
        // 이벤트를 이미 출판 중이면 출판하지 않는다.
        // 이벤트 핸들러에게 이벤트를 출판하려 할 때 발생하는 무한 재귀 문제를 방지한다.
        if (publishing.get()) {
            return;
        }
        
        
        try {
            // 출판 상태로 변경한다.
            publishing.set(Boolean.TRUE);

            List<EventHandler<?>> eventHandlers = handlers.get();
            if (eventHandlers == null) {
                return;
            }

            for (EventHandler handler : eventHandlers) {
                if (handler.canHandle(event)) {
                    handler.handle(event);
                }
            }
            
        } finally {
            publishing.set(Boolean.FALSE);
        }
    }

    // 이벤트 핸들러 등록
    public static void handle(EventHandler<?> handler) {
        // 이벤트를 처리 중이면 등록하지 않는다.
        if (publishing.get()) {
            return;
        }
        
        List<EventHandler<?>> eventHandlers = handlers.get();
        if (eventHandlers == null) {
            eventHandlers = new ArrayList<>();
            handlers.set(eventHandlers);
        }

        eventHandlers.add(handler);
    }

    // 핸들러 제거
    public static void reset() {
        if (!publishing.get()) {
            handlers.remove();
        }
    }
}
```

## 흐름 정리

도메인의 상태 변경과 이벤트 핸들러는 같은 트랜잭션 범위에서 실행된다.

![Start DDD!](/img/start-ddd/event-flow-sequence-diagram.png)

_출처 Start DDD!_

1. 이벤트 처리에 필요한 이벤트 핸들러 생성
2. 이벤트 발생 전에 이벤트 핸들러 등록
3. 이벤트를 발생하는 도메인 기능 실행
4. 도메인은 Events.raise()를 이용해서 이벤트 발생
5. 디스패처는 이벤트를 처리할 수 있는지 확인
6. 디스패처는 발생한 이벤트를 이벤트 핸들러에게 위임하여 이벤트 처리
7. 이벤트 결과 응답
8. ThreadLocal 초기화

## AOP를 이용한 Events.reset() 실행

응용 서비스가 끝나면 ThreadLocal에 등록된 핸들러 목록을 초기화 로직이 중복된다. 

이러한 횡단 로직은 AOP를 활용하면 중복을 제거할 수 있다.

## 비동기 이벤트 처리

이벤트의 후속 조치를 바로할 필요 없이 일정 시간 안에만 처리하면 되는 경우에 비동기 방식으로 구현한다.

- `A하면 일정 시간 안에(최대 언제까지) B하라`

비동기 이벤트 방식은 매우 다양한데, 다음 네 가지 방식으로 비동기 이벤트 처리를 구현하는 방법에 대해 살펴본다.

- 로컬 핸들러를 비동기로 실행
- 메시지 큐를 사용하기
- 이벤트 저장소와 이벤트 포워더 사용하기
- 이벤트 저장소와 이벤트 제공 API 사용하기

## 로컬 핸들러를 비동기로 실행

이벤트 핸들러를 별도의 스레드로 실행하는 방법이다.

- java.util.concurrent.ExecutorService 활용
- TimeOut 설정
  - `executor.awaitTermination(10, TimeUnit.SECONDS)`

비동기로 실행된 핸들러 로직은 별도의 스레드로 동작되기 때문에, 호출한 응용 서비스 메서드와 **서로 다른 트랜잭션 범위에서 실행된다.**

> 스프링의 트랜잭션 관리자는 보통 스레드를 이용해서 트랜잭션을 전파한다. <br/>
> 물론, 스레드가 아닌 다른 방식을 이용해서 트랜잭션을 전파할 수 있지만 일반적으로 사용하는 트랜잭션 관리자는 스레드를 이용해서 트랜잭션을 전파한다.<br/>
> 이런 이유로 다른 스레드에서 실행되는 두 메서드는 서로 다른 트랜잭션을 사용하게 된다.

하나의 트랜잭션에서 처리해야될 이벤트는 비동기 방식으로 처리해선 안된다. 

## 메시지 시스템을 이용한 비동기 구현

메시징 시스템을 사용하여 비동기 이벤트를 처리하는 방식이다.

![Start DDD!](/img/start-ddd/async-event-with-messaging-system.png)

_출처 Start DDD!_

1. 디스패처는 이벤트를 메시지 큐에게 보낸다.
2. 메시지 큐는 이벤트를 메시지 리스너에게 전달
3. 메시지 리스너는 알맞은 이벤트 핸들러를 이용해서 이벤트 처리한다.

메시지 큐에 저장하는 과정과 메시지 큐에서 이벤트를 읽어와 처리하는 과정은 별도 스레드나 프로세스로 처리된다.

필요하다면 이벤트를 발생하는 도메인 기능과 메시지 큐에 이벤트를 저장하는 절차를 한 트랜잭션으로 묶어야 한다.

다음 이벤트 과정이 하나의 같은 트랜잭션 범위에서 실행하려면 글로벌 트랜잭션이 필요하다. 

- 도메인 기능을 실행한 결과를 DB에 반영
- 이 과정에서 발생한 이벤트를 메시지 큐에 저장

글로벌 트랜잭션은 안전하게 이벤트를 메시지 큐에 전달할 수 있는 장점이 있지만, 전체 성능이 떨어지는 단점도 있다.

일반적으로 메시징 시스템은 글로벌 트랜잭션 지원과 함께 클러스터와 고가용성(HA)을 지원하기 때문에 안정적으로 메시지를 전달할 수 있는 장점이 있다.

## 이벤트 저장소를 이용한 비동기 처리

이벤트 저장소 방식은 DB에 이벤트를 저장한 뒤 별도 프로그램을 이용해서 이벤트 핸들러에게 전달하는 방식이다.

- 도메인의 상태와 이벤트 저장소로 동일한 DB를 사용한다.
- 도메인의 상태 변경과 이벤트 저장은 로컬 트랜잭션으로 처리된다.

![Start DDD!](/img/start-ddd/async-event-with-persistence-store.png)

_출처 Start DDD!_

1. 이벤트가 발생하면 핸들러는 스토리지에 이벤트를 저장한다.
2. 포워더는 주기적으로 이벤트 저장소에서 이벤트를 가져와 이벤트 핸들러를 실행한다.
3. 포워드는 별도 스레드를 이용하기 때문에 이벤트 발행과 처리가 비동기로 처리된다.

두 번째 방법은 이벤트를 외부에 제공하는 API를 사용하는 것이다.

![Start DDD!](/img/start-ddd/async-event-with-persistence-store2.png)

_출처 Start DDD!_

### API 방식 포워더 방식

차이점은 이벤트를 전달하는 방식에 있다.

- 포워드 방식
  - 포워더를 이용해서 이벤트를 외부에 전달하는 방식
  - 이벤트를 어디까지 처리했는지 추적하는 역할이 포워더에 있다.
- API 방식
  - 외부 핸들러가 API 서버를 통해 이벤트 목록을 가져오는 방식이다.
  - 이벤트를 어디까지 처리했는지 추적하는 역할은 이벤트 목록을 요구하는 외부 핸들러가 자신이 어디까지 이벤트를 처리했는지 기억해야 한다.

### 이벤트 저장소 구현

포워더 방식과 API 방식 모두 이벤트 저장소를 사용한다.

- eventstore
  - ui
    - EventApi
      - REST API를 이용해서 이벤트 목록을 제공하는 컨트롤러
  - api
    - EventStore (인터페이스)
      - 인벤트 저장, 조회 기능 제공하는 인터페이스
    - EventEntry
      - 이벤트 저장소에 보관할 데이터
  - infra
    - JdbcEventStore (EventStore 구현)
      - JDBC를 이용한 EventStore 구현 클래스

이벤트는 과거에 벌어진 사건이므로 데이터가 변경되지 않는다.

EventStore 인터페이스는 새로운 이벤트를 추가하는 기능과 조회하는 기능만 제공하고 기존 이벤트 데이터를 수정하는 기능은 제공하지 않는다.

### 이벤트 저장을 위한 이벤트 핸들러 구현

이벤트 저장소를 위한 기반이 되는 클래스는 모두 구현했다. 발생한 이벤트를 이벤트 저장소에 추가하는 이벤트 핸들러가 필요하다.

```java
public class EventStoreHandler implements EventHandler<Object> {

    private EventStore eventStore;

    @Override
    public void handle(Object event) {
        // 이벤트 저장소에 이벤트 저장
        eventStore.save(event);
    }
}
```

### REST API 구현

이벤트 저장소에서 이벤트 목록을 조회하는 기능을 제공한다.

이벤트를 수정하는 기능이 없으므로 REST API 는 단순 조회 기능만 존재한다.

클라이언트에서 제공한 REST API 를 주기적으로 요청하여 언제든지 원하는 이벤트를 가져올 수 있기 때문에 이벤트 처리에 실패하면 다시 실패한 이벤트부터 읽어와 이벤트를 재처리할 수 있다.

API 서버에 장애가 발생한 경우에도 주기적으로 재시도를 해서 API 서버가 살아나면 이벤트를 처리할 수 있다.

### 포워더 구현

포워더는 앞서 봤던 API 방식에서 클라이언트의 구현과 유사하다.

포워더는 일정 주기로 EventSource로부터 이벤트를 읽어와 이벤트 핸들러에 전달하면 된다.

API 방식에서 클라이언트와 마찬가지로 마지막으로 전달한 이벤트의 오프셋을 기억해 두었다가 다음 조회 시점에 마지막으로 처리한 오프셋부터 이벤트를 가져오면 된다.

## 이벤트 적용시 추가 고려사항

1. 이벤트 소스를 EventEntry에 추가할지 여부다.
   - 이벤트 발생 주체가 없다면, 특정 주체가 발생한 이벤트만 조회하는 기능을 구현할 수 없다.
   - 이 기능을 구현하려면 다음의 다섯 가지를 추가해야 한다.
     - Event.raise()에 source를 파라미터로 추가한다.
     - EventHandler.handler() 에 source를 파라미터로 추가한다.
     - EventEntry에 source 필드를 추가한다.
     - EventStore.save() 에 source 파라미터를 추가한다.
     - EventStore.get()에 필터 조건으로 source 파라미터를 추가한다.
2. 포워더에서 전송 실패를 얼마나 허용할 것이냐에 대한 것이다.
   - 포워더는 이벤트 전송에 실패하면 실패한 이벤트부터 다시 읽어와 전송을 시도한다.
   - 만약 특정 이벤트에서 계속 전송에 실패하면 나머지 이벤트를 전송할 수 없게 된다.
   - 포워더를 구현할 때는 실패한 이벤트의 재전송 횟수에 제한을 두어야 한다.
3. 이벤트 손실
   - 이벤트 저장소를 사용하는 방식은 이벤트 발생과 이벤트 저장을 한 트랜잭션으로 처리하기 때문에 이벤트 저장소에 보관된다는 것을 보장할 수 있다.
   - 반면, 로컬 핸들러를 이용해서 이벤트를 비동기로 처리할 경우 이벤트 처리에 실패하면 이벤트를 유실하게 된다.
4. 이벤트 순서
   - 이벤트를 발생 순서대로 외부 시스템에 전달해야 할 경우 이벤트 저장소를 사용하는 것이 좋다.
   - 이벤트 저장소는 일단 저장소에 이벤트를 발생 순서대로 저장하고, 그 순서대로 이벤트 목록을 제공한다.
   - 반면에 메시징 시스템은 사용 기술에 따라 이벤트 발생 순서와 메시지 전달 순서가 다를 수도 있다.
5. 이벤트 재처리
   - 동일한 이벤트를 다시 처리해야 할 때 이벤트를 어떻게 할지 결정해야 한다.
   - 가장 쉬운 방법은 마지막으로 처리한 이벤트의 순번을 기억해 두었다가 이미 처리한 순번의 이벤트가 도착하면 해당 이벤트를 처리하지 않고 무시하는 것이다.
   - 이 외에 이벤트 처리를 멱등(idempotent)으로 처리하는 방법도 있다.

> 멱등성, 연산을 여러 번 적용해도 결과가 달라지지 않는 성징을 멱등성(idempotent)이라고 한다.<br/>
> 이벤트 처리도 동일 이벤트를 한 번 적용하나 여러 번 적용하나 시스템이 같은 상태가 되도록 핸들러를 구현할 수 있다.<br/>
> 이벤트 핸들러가 멱등성을 가지면 시스템 장애로 인해 같은 이벤트가 중복해서 발생해도 결과적으로 동일 상태가 된다.<br/>
> 이는 이벤트 중복 발생이나 중복 처리에 대한 부담을 줄여준다.
