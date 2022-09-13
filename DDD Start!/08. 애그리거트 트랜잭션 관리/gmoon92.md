# 8장 애그리거트 트랜잭션 관리

- 애그리거트의 트랜잭션
  - [애그리거트와 트랜잭션](#애그리거트와-트랜잭션)
- 애그리거트 잠금 기법
  - [선점 잠금](#선점-잠금)
  - [선점 잠금과 교착 상태](#선점-잠금과-교착-상태)
  - [비선점 잠금](#비선점-잠금)
  - [강제 버전 증가](#강제-버전-증가)
  - [오프라인 선점 잠금](#오프라인-선점-잠금)
  - [오프라인 선점 잠금을 위한 LockManager 인터페이스와 관련 클래스](#오프라인-선점-잠금을-위한-LockManager-인터페이스와-관련-클래스)
  - [DB를 이용한 LockManager 구현](#DB를-이용한-LockManager-구현)

## 애그리거트와 트랜잭션

한 애그리거트를 두 사용자가 거의 동시에 변경할 때 트랜잭션이 필요하다.

트랜잭션의 동시성 제어 문제는 다음과 같다.

- 두 사용자가 동시에 애그리거트의 상태를 수정한다.
- 애그리거트 조회 기능은 캐시가 적용되어 있지 않다.
- 트랜잭션마다 리포지터리는 새로운 애그리거트 객체를 생성한다.
- 두 스레드는 개념적으로 동일한 애그리거트이지만 물리적으로 서로 다른 애그리거트 객체를 사용한다.
- 한 스레드에서 애그리거트 상태를 변경한다고 해서 다른 스레드의 애그리거트엔 영향을 주지 않는다.
- 예를 들어 관리자가 특정 기능의 옵션을 비활성화(상태 변경)할 경우, 사용자 스레드는 활성화된 기능을 사용할 수 있다.
- 이 경우 각각 트랜잭션을 커밋할 때 수정한 내용을 DBMS에 반영한다.
- 두 스레드가 변경한 애그리거트 상태가 DB에 반영된다.
- 동시성 문제로 애그리거트의 일관성이 깨진다.

동시성 제어 방안은 크게 두 가지다.

- 관리자가 애그리거트를 조회하고 상태를 변경하는 동안 사용자 스레드는 애그리거를 수정하지 못하도록 막는다.
- 관리자가 애그리거트를 조회한 이후에 사용자 스레드가 애그리거트를 수정했다면 다시 조회한 뒤 수정한다.

이 두 가지는 애그리거트 자체의 트랜잭션과 관련이 있다.

DBMS가 지원하는 트랜잭션과 함께 애그리거트를 위한 추가적인 트랜잭션 처리 기법이 필요하다.

애그리거트를 위한 추가적인 트랜잭션 처리 방식 기법은 다음과 같다.

- 비관적 잠금(Pessimistic Lock, 선점 잠금)
- 낙관적 잠금(Optimistic Lock, 비선점 잠금)

## 선점 잠금

선점 잠금(Pessimistic Lock, 비관적 잠금) 은 먼저 애그리거트 상태를 변경하는 스레드에게 상태 변경에 대한 우선 순위를 주는 방식이다.

먼저 애그리거트 상태를 점유한 스레드의 사용이 끝날 때까지 다른 스레드가 해당 애그리거트를 수정하는 것을 막는 방식이다.

![Start DDD!](/img/start-ddd/pessimistic-lock.png)

_출처 Start DDD!_

- 선점한 스레드의 작업이 완료될 때까지 다른 스레드는 블로킹된다.
- 한 트랜잭션 범위에서만 적용된다.
- 단일 트랜잭션에서 동시 변경을 막는다.
- DBMS 가 제공하는 행 단위 잠금을 사용해서 구현한다.
  - `for update`와 같은 쿼리를 사용해서 특정 레코드에 한 사용자만 접근할 수 있는 잠금 장치를 제공한다.
  - `for update`란 동시성 제어를 위해 특정 데이터(ROW)에 대해 베타적 LOCK 을 거는 기능

JPA 의 `LockModeType.PESSIMISTIC_WRITE` 옵션으로 선점 잠금 방식을 설정할 수 있다.

```html
Order oder = entityManager.find(Order.class, orderNo, LockModeType.PESSIMISTIC_WRITE);
```

- LockModeType.PESSIMISTIC_WRITE
- @Lock
- javax.persistence.lock.scope
- javax.persistence.lock.timeout
- [Hibernate docs - locking](https://docs.jboss.org/hibernate/orm/6.1/userguide/html_single/Hibernate_User_Guide.html#locking)

## 선점 잠금과 교착 상태

선점 잠금은 쓰레드 교착 상태(deadlock)이 발생하지 않도록 주의해야 한다.

1. 스레드1: A 애그리거트에 대한 선점 잠금 구함
2. 스레드2: B 애그리거트에 대한 선점 잠금 구함
3. 스레드1: B 애그리거트에 대한 선점 잠금 시도
4. 스레드2: A 애그리거트에 대한 선점 잠금 시도

선점 잠금에 따른 교착 상태는 상대적으로 사용자 수가 많을 때 발생할 가능성이 높다.

교착 상태를 빠지지 않도록 자금을 구할 때 최대 대기 시간을 지정한다.

- `javax.persistence.lock.timeout`

```java
public class SampleRepository {

    public void findByPessimisticLock() {
        Map<String, Object> hints = new HashMap<>();
        // ms 단위
        hints.put("javax.persistence.lock.timeout", 2000);

        Order oder = entityManager.find(
                Order.class,
                orderNo,
                LockModeType.PESSIMISTIC_WRITE,
                hints
        );
    }
}
```

선점 잠금을 사용하려면 사용하는 DBMS에 대해 JPA가 어떤 식으로 대기 시간을 처리하는지 반드시 확인해야 한다.

- DBMS 에 따라 교착 상태에 빠진 커넥션을 처리하는 방식이 다르다.
  - 쿼리별로 대기 시간을 지정하는 방식
  - 커넥션 단위로만 대기 시간을 지정할 수 있는 방식
  - 지원을 안하는 경우도 있다.

## 비선점 잠금

비선점 잠금(Optimistic Lock) 방식은 잠금을 해서 동시에 접근하는 것을 막는 대신 변경한 데이터를 실제 DBMS에 반영하는 시점에 변경 가능 여부를 확인하는 방식이다.

- version 증가 값을 컬럼이 필요하다.
- 트랜잭션 커밋 이후 version 값 확인을 통해 충돌 여부를 판별한다.

```sql
UPDATE agg
SET version = version + 1,
    name    = :name
WHERE agg.id = :id
  AND version = :현재 버전
```

- @Version

비선점 잠금은 쿼리 실행 결과로 수정된 행의 개수가 0이면 이미 누군가 앞서 데이터를 수정한 것으로, 트랜잭션이 충돌한 것으로 간주한다. 트랜잭션 종료 시점에 `OptimisticLockingFailureException` 예외가 발생한다.

## 강제 버전 증가

기능 실행 도중 루트 엔티티가 아닌 다른 엔티티의 값만 변경된다고 하자.

- 이 경우 JPA는 **루트 엔티티의 버전 값을 증가하지 않는다.**
- 연관된 엔티티의 값이 변경된다고 해도 루트 엔티티 자체의 값은 바뀌는 것이 없으므로 루트 엔티티의 버전 값을 갱신하지 않는다.

애그리거트 관점에서 보면 문제가 된다.

- 루트 엔티티의 상태가 변경되지 않더라도, 애그리거트의 구성요소 중 일부 값이 바뀌면 논리적으로 그 애그리거트는 바뀐 것이다.
- 애그리거트 내에 어떤 구성요소의 상태가 바뀌면 루트 애그리거트의 버전 값을 증가해야 비선점 잠금이 올바르게 동작한다.

JPA 는 강제로 버전 값을 증가시키는 잠금 모드를 지원하고 있다.

- LockModeType.OPTIMISTIC_FORCE_INCREMENT

엔티티의 상태가 변경되었는지 여부에 상관없이 트랜잭션 종료 시점에 버전 값 증가 처리를 한다.

## 오프라인 선점 잠금

오프라인 선점 잠금(Offline Pessimistic Lock) 은 조회 시점에 잠금하는 기능이다.

![Start DDD!](/img/start-ddd/offline-pessimistic-lock.png)

- 오프라인 선점 잠금은 여러 트랜잭션에 걸쳐 동시 변경을 막는다.
- 오프라인 선점 잠금은 최대 대기 시간(timeout) 설정을 해야한다.
  - 조회 시점에 잠금 요청
  - 수정 시점에 잠금 해제
  - 락 최대 대기 시간 연장
- 데이터 수정 화면을 누군가 선점하고 있다면 다른 누군가 수정 화면 자체를 실행할 수 없도록 구현할 수 있다.
  - 하나의 트랜잭션 범위에서만 적용되는 선점 잠금 방식이나 버전 충돌을 확인하는 비선점 잠금 방식으로는 이를 구현할 수 없다.

## 오프라인 선점 잠금을 위한 LockManager 인터페이스와 관련 클래스

오프라인 선점 잠금은 크게 네 가지 기능을 제공해야 한다.

```java
public interface LockManager {

    // type: 잠글 대상
    LockId tryLock(String type, String id) throws LockException;
    void checkLock(LockId lockId) throws LockException;
    void releaseLock(LockId lockId) throws LockException;
    void extendLockExpiration(LockId lockId, long inc) throws LockException;
}

public class LockId {

    private final String value;

    public LockId(String value) {
        this.value = value;
    }

    public String getValue() {
        return value;
    }
}
```

- 잠금 선점 시도
- 잠금 잠금 확인
- 잠금 해제
- 락 유효 시간 연장

컨트롤러가 오프라인 선점 잠금 기능을 이용한 예제 코드다.

```java

@Controller
public class SomeFormController {

    private LockManager lockManager;

    // 오프라인 잠금 설정
    @GetMapping("/edit/{id}")
    public String editForm(@PathVariable Long id, ModelMap model) {

        // 1. 오프라인 선점 잠금 시도
        LockId lockId = lockManager.tryLock("data", id);

        // 2. 기능 실행
        Data data = someDao.select(id);
        model.addAttribute("data", data);

        // 3. 잠금 해제에 사용할 LockId를 모델에 추가
        model.addAttribute("lockId", lockId);
        return "editFrom";
    }

    // 오프라인 잠금 해제
    @PostMapping("/edit/{id}")
    public String edit(@PathVariable Long id,
            @ModelAttribute EditRequest editReq) {
        LockId lockId = editReq.getLockId();

        // 1. 잠금 선점 확인
        lockManager.checkLock(lockId);

        // 2. 기능 실행
        someEditService.edit(editReq);
        model.addAttribute("data", data);

        // 3. 잠금 해제
        lockManager.releaseLock(lockId);
        return "editSuccess";
    }
}
```

- 잠금을 선점한 이후에 실행하는 기능은 LockId를 갖는 잠금이 유효한지 검사해야 한다.
- 잠금의 유효 시간이 지났으면 이미 다른 사용자가 잠금을 선점한다.
- 잠금을 선점하지 않은 사용자가 기능을 실행했다면 기능 실행을 막아야 한다.

## DB를 이용한 LockManager 구현

```java

@Entity
public class Locks {

    @EmbeddedId
    private Id id = new Id();
    private String lockId;
    private long expirationTime;

    @Embeddable
    public static class Id {

        private UUID id;
        private String type;
    }

    // 락 잠금 시간 만료 여부
    public boolean isExpired() {
        return expirationTime < System.currentTimeMillis();
    }
}

@Repository
public class SpringLockManager implements LockManager {

    private static final int TIMEOUT = Duration.ofMinutes(5)
            .toMillis();

    // 오프라인 잠금 시도
    @Transactional
    @Override
    public LockId tryLock(String type, String id) throws Exception {
        // 잠금이 존재하는지 검사
        checkAlreadyLocked(type, id);
        LockId lockId = new Locks(UUID.randomUUID());
        
        // 잠금 생성
        locking(type, id, lockId);
        return lockId;
    }

    private void checkAlreadyLocked(String type, String id) {
        // 오프라인 잠금 락 조회
        List<Locks> locks = findAll(type, id);
        
        // 선점이된 잠금이 있는지 확인
        Optional<Locks> renewalLocks = handleExpiration(locks);

        // 잠금이 선점되었다면 예외 발생
        if (renewalLocks.isPresendt()) {
            throw new AlreadyLockedException();
        }
    }

    private Optional<Locks> handleExpiration(List<Locks> locks) {
        if (locks.isEmpty()) {
            return Optional.empty();
        }
        
        // 유효 시간이 지난 잠금 데이터 삭제
        Locks result = locks.get(0);
        if (result.isExpired()) {
            delete(result);
            return Optional.empty();
        }
        return Optional.of(result);
    }

    // 잠금 생성
    private void locking(String type, String id, LockId lockId) {
        try {
            save(new Locks(type, id, lockId));
        } catch (DuplicateKeyException e) {
            throw new LockingFailException(e);
        }
    }

    private long getExpirationTime() {
        return System.currentTimeMillis() + TIMEOUT;
    }
}
```
