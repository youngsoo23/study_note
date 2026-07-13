# 순환 참조와 CQRS 정리

## 1. 순환 참조란?

순환 참조는 **객체 또는 클래스가 서로를 다시 참조하여 의존 관계가 원처럼 연결된 상태**를 말합니다.

가장 단순한 구조는 다음과 같습니다.

```java
class A {
    private B b;
}

class B {
    private A a;
}
```

의존 관계를 그림으로 표현하면 다음과 같습니다.

```text
A → B
↑   ↓
└───┘
```

즉,

- `A`가 `B`를 필요로 하고
- `B`도 다시 `A`를 필요로 하는 상태입니다.

이를 **순환 참조** 또는 **순환 의존성**이라고 부릅니다.

---

## 2. 스프링에서의 순환 참조

스프링에서는 주로 두 개 이상의 빈이 서로를 주입받을 때 문제가 발생합니다.

```java
@Component
public class OrderService {

    private final PaymentService paymentService;

    public OrderService(PaymentService paymentService) {
        this.paymentService = paymentService;
    }
}
```

```java
@Component
public class PaymentService {

    private final OrderService orderService;

    public PaymentService(OrderService orderService) {
        this.orderService = orderService;
    }
}
```

스프링이 빈을 생성하는 과정을 생각해 보면 다음과 같습니다.

```text
OrderService를 만들려면 PaymentService가 필요함
PaymentService를 만들려면 OrderService가 필요함
OrderService를 만들려면 PaymentService가 필요함
...
```

결국 스프링은 **어떤 객체를 먼저 생성해야 하는지 결정하지 못합니다.**

생성자 주입을 사용하는 경우 보통 애플리케이션 실행 시 다음과 비슷한 오류가 발생합니다.

```text
The dependencies of some of the beans in the application context form a cycle
```

---

## 3. 순환 참조가 생기는 이유

순환 참조는 보통 두 클래스가 서로의 책임을 너무 많이 알고 있을 때 발생합니다.

예를 들어 다음과 같은 흐름이 있다고 가정해 봅니다.

```text
주문 서비스 → 결제 처리 요청
결제 서비스 → 주문 상태 변경 요청
```

코드로 작성하면 다음과 같은 구조가 될 수 있습니다.

```java
class OrderService {

    private PaymentService paymentService;

    void order() {
        paymentService.pay();
    }

    void completeOrder() {
        // 주문 완료 처리
    }
}
```

```java
class PaymentService {

    private OrderService orderService;

    void pay() {
        // 결제 처리
        orderService.completeOrder();
    }
}
```

이 구조에서는 `OrderService`와 `PaymentService`가 서로 강하게 연결됩니다.

그 결과 다음과 같은 문제가 생길 수 있습니다.

- 한 클래스를 수정하면 다른 클래스에도 영향이 생길 수 있음
- 테스트가 어려워짐
- 책임의 경계가 모호해짐
- 코드 흐름을 이해하기 어려워짐
- 스프링 빈 생성에 실패할 수 있음

---

## 4. 순환 참조 해결 방법

### 4.1 제3의 서비스가 흐름을 관리하도록 분리

두 서비스가 서로를 직접 호출하지 않게 하고, 별도의 조정 역할을 하는 서비스를 둘 수 있습니다.

```java
@Component
public class OrderPaymentFacade {

    private final OrderService orderService;
    private final PaymentService paymentService;

    public OrderPaymentFacade(
            OrderService orderService,
            PaymentService paymentService
    ) {
        this.orderService = orderService;
        this.paymentService = paymentService;
    }

    public void orderAndPay() {
        orderService.createOrder();
        paymentService.pay();
        orderService.completeOrder();
    }
}
```

의존 관계는 다음처럼 바뀝니다.

```text
OrderPaymentFacade
    ├── OrderService
    └── PaymentService
```

이제 `OrderService`와 `PaymentService`는 서로를 직접 참조하지 않습니다.

---

### 4.2 이벤트를 사용하여 결합도 낮추기

결제 서비스가 주문 서비스를 직접 호출하지 않고, 결제 완료 이벤트를 발생시키는 방식도 사용할 수 있습니다.

```text
OrderService → PaymentService
PaymentService → PaymentCompleted 이벤트 발생
OrderService ← 이벤트 수신
```

예시:

```java
public record PaymentCompletedEvent(Long orderId) {
}
```

```java
@Service
@RequiredArgsConstructor
public class PaymentService {

    private final ApplicationEventPublisher eventPublisher;

    public void pay(Long orderId) {
        // 결제 처리

        eventPublisher.publishEvent(
                new PaymentCompletedEvent(orderId)
        );
    }
}
```

```java
@Component
@RequiredArgsConstructor
public class PaymentCompletedEventHandler {

    private final OrderService orderService;

    @EventListener
    public void handle(PaymentCompletedEvent event) {
        orderService.completeOrder(event.orderId());
    }
}
```

이 방식은 서비스 간 직접 참조를 줄이는 데 도움이 됩니다.

---

## 5. 순환 참조와 메모리 누수

순환 참조가 있다고 해서 반드시 메모리 누수가 발생하는 것은 아닙니다.

```text
A ↔ B
```

Java의 가비지 컬렉터는 `A`와 `B`가 서로 참조하고 있더라도, 프로그램의 다른 곳에서 더 이상 두 객체에 접근할 수 없다면 가비지 컬렉션 대상으로 판단할 수 있습니다.

따라서 스프링에서 말하는 순환 참조 문제는 주로 다음에 가깝습니다.

- 메모리 문제
  - 항상 발생하는 것은 아님
- 빈 생성 문제
  - 실제로 발생 가능
- 설계 문제
  - 책임이 강하게 얽혀 있다는 신호

### 순환 참조 한 문장 정리

> 순환 참조란 A가 B를 필요로 하고, B도 다시 A를 필요로 하여 의존 관계가 원처럼 연결된 상태입니다.

---

# 6. CQRS란?

CQRS는 다음 문장의 약자입니다.

> Command Query Responsibility Segregation

한국어로는 보통 **명령과 조회의 책임 분리**라고 표현합니다.

쉽게 말하면 다음 두 가지 역할을 분리하는 설계 방식입니다.

```text
데이터를 변경하는 기능
+
데이터를 조회하는 기능
```

즉,

```text
Command = 변경
Query = 조회
```

입니다.

---

## 7. 일반적인 서비스 구조

일반적인 구조에서는 하나의 서비스가 변경과 조회를 모두 담당할 수 있습니다.

```java
@Service
public class OrderService {

    public void createOrder() {
        // 주문 생성
    }

    public void cancelOrder() {
        // 주문 취소
    }

    public OrderResponse getOrder(Long orderId) {
        // 주문 조회
        return null;
    }
}
```

이 구조 자체가 잘못된 것은 아닙니다.

작은 프로젝트나 단순한 CRUD에서는 충분히 사용할 수 있습니다.

---

## 8. CQRS를 적용한 구조

CQRS를 적용하면 변경과 조회를 서로 다른 서비스로 나눌 수 있습니다.

```java
@Service
public class OrderCommandService {

    public void createOrder() {
        // 주문 생성
    }

    public void cancelOrder() {
        // 주문 취소
    }
}
```

```java
@Service
public class OrderQueryService {

    public OrderResponse getOrder(Long orderId) {
        // 주문 단건 조회
        return null;
    }

    public List<OrderResponse> getOrders() {
        // 주문 목록 조회
        return List.of();
    }
}
```

구조를 정리하면 다음과 같습니다.

```text
주문 생성·수정·취소 → Command
주문 단건·목록 조회 → Query
```

---

## 9. Command란?

Command는 시스템의 상태를 변경하는 작업입니다.

예시:

- 회원 가입
- 주문 생성
- 주문 취소
- 결제 완료
- 배송지 변경
- 쿠폰 사용
- 포인트 차감

```java
public Long createOrder(CreateOrderCommand command) {
    // 주문 생성
    return orderId;
}
```

Command는 보통 다음과 같은 역할에 집중합니다.

- 비즈니스 규칙 검증
- 엔티티 상태 변경
- 트랜잭션 처리
- 데이터 정합성 보장
- 저장

Command의 반환값은 필요 이상으로 크게 만들지 않는 경우가 많습니다.

예를 들어 생성된 식별자만 반환할 수 있습니다.

```java
return order.getId();
```

---

## 10. Query란?

Query는 시스템의 상태를 변경하지 않고 데이터를 읽는 작업입니다.

예시:

- 회원 조회
- 주문 목록 조회
- 결제 내역 조회
- 상품 상세 조회
- 통계 조회
- 검색

```java
public OrderResponse getOrder(Long orderId) {
    return orderRepository.findById(orderId)
            .map(OrderResponse::from)
            .orElseThrow();
}
```

Query는 주로 다음에 집중합니다.

- 빠른 조회
- 검색 조건
- 페이징
- 정렬
- 여러 테이블 조인
- 화면에 필요한 형태의 DTO 반환

---

## 11. 왜 Command와 Query를 분리할까?

변경과 조회는 중요하게 보는 기준이 다르기 때문입니다.

### 변경에서 중요한 것

```text
비즈니스 규칙
트랜잭션
정합성
상태 변경
도메인 로직
```

예를 들어 주문 생성에서는 다음과 같은 검증이 필요할 수 있습니다.

- 주문 가능한 상품인지
- 재고가 남아 있는지
- 사용 가능한 쿠폰인지
- 결제 금액이 맞는지
- 이미 처리된 주문인지

### 조회에서 중요한 것

```text
조회 속도
검색
페이징
정렬
조인
화면 전용 데이터 구조
```

예를 들어 주문 목록 화면에서는 다음 정보가 한 번에 필요할 수 있습니다.

- 주문 번호
- 회원 이름
- 상품명
- 주문 상태
- 결제 금액
- 주문일
- 배송 상태

이 데이터는 여러 테이블을 조인해야 할 수 있습니다.

따라서 저장에 적합한 모델과 조회에 적합한 모델을 각각 다르게 설계할 수 있습니다.

---

## 12. 저장 모델과 조회 모델의 차이

### 저장에서는 엔티티 중심

```java
Order order = Order.create(
        memberId,
        orderItems
);

orderRepository.save(order);
```

저장 로직에서는 도메인 규칙과 상태 변경이 중요합니다.

### 조회에서는 DTO 중심

```java
public List<OrderListResponse> getOrders() {
    return queryFactory
            .select(Projections.constructor(
                    OrderListResponse.class,
                    order.id,
                    member.name,
                    order.status,
                    order.createdAt
            ))
            .from(order)
            .join(order.member, member)
            .fetch();
}
```

조회에서는 엔티티 전체를 가져오기보다 화면에서 필요한 값만 DTO로 바로 조회할 수 있습니다.

---

## 13. CQRS 적용 수준

CQRS를 적용한다고 해서 반드시 데이터베이스까지 분리해야 하는 것은 아닙니다.

CQRS는 여러 수준으로 적용할 수 있습니다.

### 13.1 코드만 분리

가장 가벼운 형태입니다.

```text
OrderCommandService
OrderQueryService
```

두 서비스는 같은 데이터베이스와 같은 테이블을 사용합니다.

```text
CommandService ─┐
                ├─ 동일한 DB
QueryService ───┘
```

실무에서 부담 없이 시작하기 좋은 방식입니다.

---

### 13.2 조회 모델 분리

변경에 사용하는 엔티티와 조회에 사용하는 DTO 또는 조회 전용 모델을 분리합니다.

```text
Command → 주문 엔티티
Query → 주문 조회 DTO
```

필요하다면 조회 전용 테이블이나 뷰를 둘 수도 있습니다.

```text
Command → orders 테이블
Query → order_summary 테이블
```

---

### 13.3 데이터베이스까지 분리

쓰기와 읽기를 서로 다른 데이터베이스로 나눌 수 있습니다.

```text
Command → 쓰기 DB
Query → 읽기 DB
```

데이터 변경 후 이벤트를 통해 조회 DB를 갱신할 수 있습니다.

```text
주문 생성
   ↓
쓰기 DB 저장
   ↓
OrderCreated 이벤트
   ↓
읽기 DB 갱신
```

이 경우에는 쓰기 DB의 변경 내용이 읽기 DB에 즉시 반영되지 않을 수도 있습니다.

이를 **최종적 일관성**이라고 합니다.

---

## 14. 최종적 일관성이란?

데이터가 즉시 완전히 일치하지는 않지만, 일정 시간이 지나면 결국 같은 상태가 되는 것을 말합니다.

예를 들어 다음과 같습니다.

```text
10:00:00 주문 생성 완료
10:00:01 이벤트 발행
10:00:02 조회 DB 반영 완료
```

사용자가 주문 생성 직후 즉시 조회하면 아주 짧은 순간 동안 주문이 보이지 않을 수도 있습니다.

따라서 DB까지 분리한 CQRS에서는 다음을 고려해야 합니다.

- 이벤트 전달 실패
- 중복 이벤트
- 순서가 뒤바뀐 이벤트
- 재처리
- 조회 반영 지연
- 데이터 불일치 복구

---

## 15. 간단한 CQRS 예시

### Command Service

```java
@Service
@RequiredArgsConstructor
public class OrderCommandService {

    private final OrderRepository orderRepository;

    @Transactional
    public Long createOrder(CreateOrderCommand command) {
        Order order = Order.create(
                command.memberId(),
                command.productId()
        );

        orderRepository.save(order);

        return order.getId();
    }
}
```

### Query Service

```java
@Service
@RequiredArgsConstructor
public class OrderQueryService {

    private final OrderQueryRepository orderQueryRepository;

    @Transactional(readOnly = true)
    public OrderResponse getOrder(Long orderId) {
        return orderQueryRepository.findOrder(orderId);
    }
}
```

### Command Controller

```java
@RestController
@RequiredArgsConstructor
public class OrderCommandController {

    private final OrderCommandService orderCommandService;

    @PostMapping("/orders")
    public Long createOrder(
            @RequestBody CreateOrderRequest request
    ) {
        return orderCommandService.createOrder(
                request.toCommand()
        );
    }
}
```

### Query Controller

```java
@RestController
@RequiredArgsConstructor
public class OrderQueryController {

    private final OrderQueryService orderQueryService;

    @GetMapping("/orders/{orderId}")
    public OrderResponse getOrder(
            @PathVariable Long orderId
    ) {
        return orderQueryService.getOrder(orderId);
    }
}
```

---

## 16. CQRS의 장점

### 책임이 명확해진다

```text
OrderCommandService → 주문 변경 규칙
OrderQueryService → 주문 조회
```

각 서비스의 역할이 분명해집니다.

### 조회를 자유롭게 최적화할 수 있다

QueryDSL, Native Query, MyBatis 등을 사용해 조회 전용 쿼리를 작성해도 변경 로직과 분리할 수 있습니다.

### 변경 로직을 보호할 수 있다

조회 화면 요구사항이 바뀌어도 도메인 엔티티와 변경 로직에 미치는 영향을 줄일 수 있습니다.

### 독립적인 확장이 가능하다

조회 요청이 쓰기 요청보다 훨씬 많은 서비스라면 조회 서버만 더 늘릴 수 있습니다.

```text
조회 요청 증가 → 조회 서버 확장
쓰기 요청 증가 → 쓰기 서버 확장
```

---

## 17. CQRS의 단점

### 클래스 수가 늘어난다

기존에는 하나였던 구조가 다음처럼 늘어날 수 있습니다.

```text
OrderService
```

CQRS 적용 후:

```text
OrderCommandService
OrderQueryService
OrderRepository
OrderQueryRepository
OrderCommandController
OrderQueryController
```

### 구조가 복잡해질 수 있다

단순한 CRUD에도 과도하게 적용하면 코드 탐색이 어려워질 수 있습니다.

### DB까지 분리하면 운영 난이도가 높아진다

- 이벤트 발행
- 이벤트 소비
- 동기화
- 재처리
- 중복 방지
- 장애 복구

같은 추가 기능이 필요합니다.

---

## 18. CQRS에 대한 오해

### 메서드가 나뉘어 있다고 무조건 CQRS는 아니다

```java
createOrder();
getOrder();
```

메서드가 변경과 조회로 나뉘어 있는 것은 일반적인 서비스에서도 당연히 존재합니다.

CQRS의 핵심은 단순히 메서드 이름을 나누는 것이 아니라 다음에 있습니다.

> 변경 모델과 조회 모델의 책임을 의도적으로 분리하는 것

### CQRS는 Kafka가 반드시 필요하지 않다

CQRS를 적용한다고 해서 다음 기술이 모두 필요한 것은 아닙니다.

```text
Kafka
마이크로서비스
이벤트 소싱
읽기 DB 분리
쓰기 DB 분리
```

서비스 클래스만 나누는 가벼운 CQRS도 가능합니다.

### CQRS와 이벤트 소싱은 같은 것이 아니다

- CQRS: 변경과 조회의 책임을 분리
- 이벤트 소싱: 현재 상태 대신 변경 이벤트를 저장

두 방식을 함께 사용하는 경우가 많지만 서로 다른 개념입니다.

---

## 19. 언제 CQRS를 적용하면 좋을까?

다음과 같은 경우 CQRS가 도움이 될 수 있습니다.

- 조회 조건이 매우 복잡함
- 화면마다 필요한 조회 데이터가 크게 다름
- 읽기 요청이 쓰기 요청보다 훨씬 많음
- 도메인 변경 로직이 복잡함
- 조회 성능을 별도로 최적화해야 함
- 변경 모델과 조회 모델의 요구사항이 다름

반대로 다음과 같은 경우에는 굳이 적용하지 않아도 됩니다.

- 단순 CRUD
- 작은 프로젝트
- 조회 로직이 단순함
- 팀이 CQRS 구조에 익숙하지 않음
- 클래스 분리로 얻는 이점보다 복잡도가 더 큼

---

## 20. 순환 참조와 CQRS의 관계

순환 참조와 CQRS는 서로 다른 개념입니다.

### 순환 참조

의존 관계가 원처럼 연결된 설계 문제입니다.

```text
A → B → A
```

### CQRS

변경과 조회의 책임을 나누는 설계 방식입니다.

```text
Command → 변경
Query → 조회
```

다만 책임을 명확히 분리한다는 점에서는 공통점이 있습니다.

책임이 불분명하면 여러 서비스가 서로를 호출하면서 순환 참조가 생길 가능성이 커집니다.

CQRS처럼 역할을 명확히 나누면 의존 관계를 정리하는 데 도움이 될 수 있지만, CQRS를 적용한다고 순환 참조가 자동으로 해결되는 것은 아닙니다.

---

# 최종 요약

## 순환 참조

```text
A가 B를 필요로 하고
B도 다시 A를 필요로 하는 상태
```

- 빈 생성 실패 가능
- 강한 결합
- 책임 분리 부족
- 제3의 서비스나 이벤트로 해결 가능

## CQRS

```text
Command = 데이터 변경
Query = 데이터 조회
```

- 변경과 조회의 책임을 분리
- 코드만 분리하는 가벼운 적용 가능
- DB까지 반드시 나눌 필요 없음
- 복잡한 조회와 도메인 로직을 각각 최적화 가능
- 단순 CRUD에 무조건 적용할 필요는 없음

## 핵심 문장

> 순환 참조는 서로의 의존 관계가 원처럼 연결된 문제이고, CQRS는 변경과 조회의 책임을 분리하는 설계 방식입니다.
