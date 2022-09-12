# 5장 리포지터리의 조회 기능(JPA 중심)

- 스펙
  - [검색을 위한 스펙](#검색을-위한-스펙)
  - [스펙 조합](#스펙-조합)
- JPA 스펙 구현
  - [JPA를 위한 스펙 구현](#JPA를-위한-스펙-구현)
  - [AND/OR 스펙 조합을 위한 구현](#AND/OR-스펙-조합을-위한-구현)
  - [스펙을 사용하는 JPA 리포지터리 구현](#스펙을-사용하는-JPA-리포지터리-구현)
- 정렬과 페이징
  - [정렬 구현](#정렬-구현)
  - [페이징과 개수 구하기 구현](#페이징과-개수-구하기-구현)
- 동적 인스턴스와 @Subselect
  - [조회 전용 기능 구현](#조회-전용-기능-구현)
    - [동적 인스턴스 생성](#동적-인스턴스-생성)
    - [하이버네이트 @Subselect 사용](#하이버네이트-@Subselect-사용)

## 검색을 위한 스펙

리포지터리는 애그리거트의 저장소이다.

검색 조건의 조합이 다양할 경우 스펙을 이용해서 문제를 풀어야 한다.

- 스펙(Specification)은 애그리거트가 특정 조건을 충족하는지 여부를 검사한다.
- 리포지터리는 스펙을 전달받아 애그리거트를 걸러내는 용도로 사용한다.

```java
public class OrderService {

    public List<Order> findComplectedOrders() {
        Specification<Order> complectedOrderSpec = new OrdererSpec(OrderStatus.COMPLECTED);
        return orderRepository.findAll(spec);
    }
}

public class MemoryOrderRepository implements OrderRepository {

    // 스펙을 전달받아 조건에 맞는 애그리거트 조회
    public List<Order> findAll(Specification spec) {
        List<Order> all = findAll();

        return all.stream()
                .filter(spec::isSatisfiedBy)
                .collect(toList());
    }
}

public interface Specification<T> {

    // 인수 aggregate 는 검색 대상이 되는 애그리거트 객체
    public boolean isSatisfiedBy(T aggregate);
}

public class OrderSpec implements Specification<Order> {

    private String ordererId;

    public OrderSpec(String ordererId) {
        this.ordererId = ordererId;
    }

    // 검색 조건을 충족하면 true 리턴
    @Override
    public boolean isSatisfiedBy(Order aggregate) {
        return aggregate.getOrdererId()
                .getMemberId().getId()
                .equals(ordererId);
    }
}
```

### 스펙 조합

스펙의 장점은 조합에 있다.

```java
// 한 개 이상의 스펙을 AND 로 조합하는 스펙
public class AndSpec<T> implements Specification<T> {

    private List<Specification<T>> specs;

    public AndSpecification(Specification<T>... specs) {
        this.specs = Arrays.asList(specs);
    }

    public boolean isSatisfiedBy(T agg) {
        for (Specification<T> spec : specs) {
            if (!spec.isSatisfiedBy(agg)) {
                return false;
            }
        }

        return true;
    }
}

public class OrderService {

    private final OrderRepository orderRepository;

    public List<Order> findAll(String orderer, Date fromDate, Date toDate) {
        Specification<Order> ordererSpec = new OrdererSpec("madvirus");
        Specification<Order> orderDateSpec = new OrderDateSpec(fromDate, toDate);

        // 스펙 조합
        AndSpec<T> spec = new AndSpec(orderDateSpec, orderDateSpec);

        return orderRepository.findAll(spec);
    }
}
```

## JPA를 위한 스펙 구현

앞서 모든 애그리거트를 조회한 다음에 스펙을 이용해서 걸러내는 방식을 사용했다.

- 실행 속도 문제
- 메모리 문제, OOM 발생 가능성이 높다.
- 성능 이슈 발생

실제 스펙 구현은 쿼리의 where 절을 사용하는 방식으로 구현해야 한다.

- CriteriaBuilder
- Predicate

```java
import javax.persistence.criteria.CriteriaBuilder;
import javax.persistence.criteria.Predicate;
import javax.persistence.criteria.Root;

// Component
public interface Specification<T> {

    Predicate toPredicate(Root<T> root, CriteriaBuilder cb);

}

// Leaf
public class OrdererSpec implements Specification<Order> {

    private String ordererId;

    public OrdererSpec(String ordererId) {
        this.ordererId = ordererId;
    }

    // Root 엔티티 Order의 orderer.memberId.id 프로퍼티와 
    // 전달 받은 인수 비교
    @Override
    public Predicate toPredicate(Root<Order> root, CriteriaBuilder cb) {
        return cb.eqaul(root.get(Order_.orderer)
                .get(Orderer_.memberId)
                .get(MemberId_.id), ordererId);
    }
}

// JPA 정적 메타 모델 구현
@StaticMetamodel(Order.class)
public abstract class Order_ {

    public static volatile SingularAttribute<Order, OrderNo> number;
    public static volatile ListAttribute<Order, OrderLine> orderLines;
    public static volatile SingularAttribute<Order, Orderer> orderer;
    public static volatile SingularAttribute<Order, ShippinInfo> shippingInfo;
    public static volatile SingularAttribute<Order, OrderState> state;

}
```

> 정적 메타 모델은 @StaticMetamodel 애노테이션을 이용해서 관련 모델을 지정한다.<br/>
> 메타 모델 클래스는 모델 클래스의 이름 뒤에 '_'을 붙인 이름을 갖는다.<br/>
> 정적 메타 모델 클래스는 대상 모델의 각 프로퍼티와 동일한 이름을 갖는 정적 필드를 정의한다.<br/>
> 정적 필드는 프로퍼티에 대한 메타 모델로서 프로퍼티 타입에 따라 SingularAttribute, ListAttribute 등의 타입을 사용해서 메타 모델을 정의한다.</br>
> <br/>
> 정적 메타 모델을 사용하는 대신 문자열로 프로퍼티를 지정할 수도 있다.<br/>
> `root.get("orderer").get("memberId").get("id")`

Specification 구현 클래스를 개별적으로 만들지 않고 별도 클래스에 스펙 생성 기능을 모아도 된다.

```java
// 주문 스펙 기능 구현
public class OrderSpecs {

    public static Specification<Order> orderer(String ordererId) {
        return (root, cb) -> cb.eqaul(root.get(Order_.orderer)
                .get(Orderer_.memberId)
                .get(MemberId_.id), ordererId);
    }

    public static Specification<Order> between(Date from, Date to) {
        return (root, cb) -> cb.between(root.get(Order_.orderDate), from, to);
    }
}
```

### AND/OR 스펙 조합을 위한 구현

JPA를 위한 AND/OR 스펙은 각각 다음과 같이 구현할 수 있다.

```java
import javax.persistence.criteria.CriteriaBuilder;
import javax.persistence.criteria.Predicate;
import javax.persistence.criteria.Root;

// Composite
public class AndSpecification<T> implements Specification<T> {

    private List<Specification<T>> specs;

    public AndSpecification(Specification<T>... specs) {
        this.specs = Arrays.asList(specs);
    }

    @Override
    public Predicate toPredicate(Root<T> root, CriteriaBuilder cb) {
        Predicate[] predicates = specs.stream()
                .map(spec -> spec.toPredicate(root, cb))
                .toArray(size -> new Predicate[size]);

        // CriteriaBuilder and 메서드 활용
        return cb.and(predicates);
    }
}

// Composite
public class OrSpecification<T> implements Specification<T> {

    private List<Specification<T>> specs;

    public OrSpecification(Specification... specs) {
        this.specs = Arrays.asList(specs);
    }


    @Override
    public Predicate toPredicate(Root<T> root, CriteriaBuilder cb) {
        Predicate[] predicates = specs.stream()
                .map(spec -> spec.toPredicate(root, cb))
                .toArray(size -> new Predicate[size]);

        // CriteriaBuilder or 메서드 활용
        return cb.or(predicates);
    }
}

// AND/OR 스펙을 생성해주는 팩토리 클래스
public class Specs {

    public static <T> Specification<T> and(Specification<T>... specs) {
        return new AndSpecification<>(specs);
    }

    public static <T> Specification<T> or(Specification<T>... specs) {
        return new OrSpecification<>(specs);
    }
}

// 스펙 조합
public class OrderService {

    public List<Order> findAll(String orderer, Date fromTime, Date toTime) {
        Specification<Order> specs = Specs.and(
                OrderSpecs.orderer(orderer),
                OrderSpecs.between(fromTime, toTime)
        );

        return orderRepository.findAll(specs);
    }
}
```

### 스펙을 사용하는 JPA 리포지터리 구현

리포지터리 인터페이스는 스펙을 사용하는 메서드를 제공해야 한다.

```java
// 인수 스펙 제공
public interface OrderRepository {

    public List<Order> findAll(Specification<Order> spec);
}

// 스펙을 이용해서 검색을 수행하는 JPA 리포지터리 구현
@Repository
public class JpaOrderRepository implements OrderRepository {

    @PersistenceContext
    private EntityManager em;

    @Override
    public List<Order> findAll(Specification<Order> spec) {
        CriteriaBuilder cb = em.getCriteriaBuilder();

        CriteriaQuery<Order> query = cb.createQuery(Order.class);

        // 검색 조건 대상이 되는 Root 생성
        Root<Order> order = query.from(Order.class);

        // spec을 이용해서 Predicate 생성
        Predicate predicate = spec.toPredicate(root, cb);
        query.where(predicate);
        query.orderBy(
                cb.desc(root.get(Order_.number)
                        .get(OrderNo_.number)));

        TypedQuery<Order> typedQuery = em.createQuery(query);
        return typedQuery.getResultList();
    }
}

```

## 정렬 구현

JPA의 CriteriaQuery#orderBy()를 이용해서 정렬 순서를 지정한다.

```java

@Repository
public class JpaOrderRepository implements OrderRepository {

    public List<Order> findAll() {
        // JPQL
        TypedQuery<Order> query = entityManager.createQuery(
                "select o from Order o " +
                        "where o.orderer.memberId.id = :ordererId " +
                        "order by o.number.number desc", // 정렬
                Order.class);
    }

    @Override
    public List<Order> findAll(Specification<Order> spec, String... orders) {
        CriteriaBuilder cb = entityManager.getCriteriaBuilder();
        CriteriaQuery<Order> criteriaQuery = cb.createQuery(Order.class);

        Root<Order> root = criteriaQuery.from(Order.class);
        Predicate predicate = spec.toPredicate(root, cb);

        criteriaQuery.where(predicate);

        // 정렬
        if (orders.length > 0) {
            criteriaQuery.orderBy(JpaQueryUtils.toJpaOrders(root, cb, orders));
        }

        TypedQury<Order> query = entityManager.createQuery(criteriaQuery);
        return query.getResultList();
    }
}

// 문자열을 Order로 변경해 주는 보조 클래스
public class JpaQueryUtils {

    public static <T> List<Order> toJpaOrders(
            Root<T> root,
            CriteriaBuilder cb,
            String... orders) {
        if (orders == null || orders.length == 0) {
            return Collections.emptyList();
        }

        return Arrays.stream(orders)
                .map(orderStr -> toJpaOrder(root, cb, orderStr))
                .collect(toList());
    }

    public static <T> Order toJpaOrder(
            Root<T> root,
            CriteriaBuilder cb,
            String orderStr) {
        String[] orderClause = orderStr.split(" ");
        boolean ascending = true;

        if (orderClause.length == 2 && orderClause[1].equalsIgnoreCase("desc")) {
            ascending = false;
        }

        String[] paths = orderClause[0].split("\\.");
        Path<Object> path = root.get(paths[0]);
        for (int i = 1; i < paths.length; i++) {
            path = paths.get(paths[i]);
        }

        return ascending ? cb.asc(path) : cb.desc(path);
    }
}
```

## 페이징과 개수 구하기 구현

JPA 쿼리는 setFirstResult()와 setMaxResult() 메서드를 제공하고 있다.

두 메서드를 통해 페이징을 구현할 수 있다.

```java
public class JpaOrderRepository implements OrderRepository {

    public List<Order> findByOrderId(String ordererId, int startRow, int fetchSize) {
        TypedQuery<Order> query = entityManager.createQuery(
                "select ...", Order.class);

        query.setParameter("ordererId", ordererId);

        // pagination
        query.setFirstResult(startRow);
        query.setMaxResult(fetchSize);
        return query.getResultList();
    }
}
```

- setFirstResult: 첫 번째 행 번호를 지정, 첫 행은 0번 부터 시작한다.
- setMaxResult: 읽어올 행 개수를 지정한다.

## 조회 전용 기능 구현

리포지터리는 애그리거트의 저장소를 표현하는 것으로서 다음 용도로 리포지터리를 사용하는 것은 적합하지 않다.

- 여러 애그리거트를 조합해서 한 화면에 보여주는 데이터 제공
  - 글로벌 패치 전략 고민, 애그리거트 간에 ID를 활용한 간접 참조로 지원 불가
- 각종 통계 데이터 제공
  - 다양한 테이블을 조인하거나 DBMS 전용 기능 (UNION, Function) 을 사용해야 구할 수 있는데, 이는 JPQL 이나 Criteria로 처리하기 힘들다.

애초에 이런 기능은 조회 전용 쿼리로 처리해야 하는 것들이다.

JPA와 하이버네이트를 사용하면 동적 인스턴스 생성, 하이버네이트의 @Subselect 확장 기능, 네이티브 쿼리를 이용해서 조회 전용 기능을 구현할 수 있다.

### 동적 인스턴스 생성

- QueryProjection

```java
// JPQL에서 동적 인스턴스를 사용한 코드
@Repository
public class JpaOrderViewDao implements OrderViewDao {

    @PersistenceContext
    private EntityManager em;

    @Override
    public List<OrderView> selectByOrderer(String ordererId) {
        String selectQuery =
                // 동적 인스턴스 생성
                "select new OrderView(o, m, p) "
                        + "from Order o "
                        + "join o,orderLines ol, "
                        + "  Member m, "
                        + "  Product p "
                        + "where o.orderer.memberId.id = :ordererId "
                        + "order by o.number.number desc";

        TypedQuery<OrderView> query =
                em.createQuery(selectQuery, OrderView.class);
        query.setParameter("ordererId", ordererId);
        return query.getResultList();
    }

}
```

### 하이버네이트 @Subselect 사용

하이버네이트 JPA 확장 기능으로 `@Subselect`를 제공한다.

@Subselect: 쿼리 결과를 @Entity로 매핑할 수 있는 기능을 제공한다.

```java

@Entity
@Immutable // 상태 변경되어도 더티 체킹 무시
@Subselect("select o.order_number as number, "
        + "o.orderer_id, o.orderer_name, o.total_amounts, "
//        ... 
        + "p.name as product_name "
        + "from purchase_order o "
        + "inner join order_line ol on o.order_number = ol.order_number "
        + "where ol.line_idx = 0 and ol.product_id = p.product_id")
@Synchronize({"purchase_order", "order_line", "product"})
public class OrderSummary {

    @Id
    private String number;
    private String ordererId;
    
    // fields ...
    // getter methods ...
}

```

- @Immutable: 엔티티 상태가 변경되어도 DB에 반영하지 않는다. 
- @Synchronize: 지정한 테이블의 값이 변경되면 플러시를 먼저 실행하여 최신 정보로 읽어 온다.
- @Subselect: 일반 @Entity와 같기 때문에 `EntityManager#find()`, JPQL, Criteria 를 사용해서 조회할 수 있다.
