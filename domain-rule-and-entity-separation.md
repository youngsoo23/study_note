# 도메인 규칙의 복잡도와 엔티티 분리 기준

## 1. 도메인 규칙이 복잡하다는 의미

도메인 규칙이 복잡하다는 것은 단순히 데이터를 저장하고 조회하는 것을 넘어서,  
**여러 조건에 따라 행동 가능 여부와 상태 변화가 달라지는 경우가 많다**는 뜻이다.

단순한 CRUD는 보통 다음과 같은 작업에 집중한다.

```text
등록
조회
수정
삭제
```

반면 도메인 규칙이 있는 서비스는 다음을 판단해야 한다.

```text
언제 이 행동을 할 수 있는가?
어떤 조건에서는 거절해야 하는가?
상태가 어떻게 변경되어야 하는가?
다른 객체와 함께 어떤 처리를 해야 하는가?
```

---

## 2. 단순한 주문 예시

주문 정보를 저장하고 조회하는 정도라면 도메인 규칙이 단순한 편이다.

```java
@Entity
public class Order {

    @Id
    private Long id;

    private String status;
}
```

주요 기능도 다음 정도에 머문다.

```text
주문 저장
주문 조회
주문 상태 수정
```

이런 구조는 일반적인 CRUD에 가깝다.

---

## 3. 주문 취소 규칙이 생긴 경우

다음과 같은 규칙이 추가될 수 있다.

```text
결제 전에는 취소 가능
결제 완료 후에도 배송 전이면 취소 가능
배송 시작 후에는 취소 불가
```

```java
public void cancel() {
    if (status == OrderStatus.SHIPPED) {
        throw new IllegalStateException(
                "배송 시작 후에는 취소할 수 없습니다."
        );
    }

    this.status = OrderStatus.CANCELED;
}
```

이제 `Order`는 단순히 주문 데이터를 담는 것이 아니라  
**주문 취소 가능 여부를 판단하는 비즈니스 규칙**을 가진다.

---

## 4. 도메인 규칙이 더 복잡해지는 경우

주문 취소에 다음 조건까지 붙는다고 생각해보자.

```text
일반 상품은 배송 전까지 취소 가능
디지털 상품은 다운로드 전까지만 취소 가능
예약 상품은 이용일 3일 전까지만 취소 가능
쿠폰을 사용했다면 쿠폰 복원 필요
포인트를 사용했다면 포인트 환불 필요
결제된 주문이라면 결제 취소 필요
일부 배송된 주문은 일부 상품만 취소 가능
```

이제 주문 취소 하나에도 여러 조건과 후속 작업이 연결된다.

```java
public void cancel(LocalDateTime now) {
    validateCancelable(now);
    restoreCoupon();
    refundPoint();
    cancelPayment();
    changeStatus();
}
```

이처럼 하나의 행동에 조건과 상태 변화가 많이 얽히면  
도메인 규칙이 복잡하다고 볼 수 있다.

---

## 5. 쉬운 예시: 카페 쿠폰

### 단순한 쿠폰

규칙이 단순하면 쿠폰 사용 여부만 변경하면 된다.

```java
public void use() {
    this.used = true;
}
```

### 규칙이 많은 쿠폰

다음과 같은 조건이 붙을 수 있다.

```text
유효기간 안에서만 사용 가능
이미 사용한 쿠폰은 다시 사용 불가
특정 매장에서만 사용 가능
최소 주문 금액 이상일 때만 사용 가능
다른 쿠폰과 중복 사용 불가
주문 취소 시 쿠폰 복원 여부가 다름
```

```java
public void use(
        Order order,
        Store store,
        LocalDateTime now
) {
    if (used) {
        throw new IllegalStateException(
                "이미 사용한 쿠폰입니다."
        );
    }

    if (now.isAfter(expiredAt)) {
        throw new IllegalStateException(
                "만료된 쿠폰입니다."
        );
    }

    if (!availableStoreIds.contains(store.getId())) {
        throw new IllegalStateException(
                "사용할 수 없는 매장입니다."
        );
    }

    if (order.getAmount() < minimumOrderAmount) {
        throw new IllegalStateException(
                "최소 주문 금액을 충족하지 못했습니다."
        );
    }

    this.used = true;
}
```

이제 쿠폰은 단순한 데이터가 아니라  
여러 비즈니스 조건을 스스로 판단하는 도메인 객체가 된다.

---

## 6. 도메인 규칙이 복잡한지 판단하는 기준

다음 항목이 많아질수록 도메인 규칙이 복잡한 편이다.

- 상태가 많다.
- 상태마다 가능한 행동이 다르다.
- 조건에 따라 결과가 달라진다.
- 하나의 행동에 여러 객체가 함께 관여한다.
- 비즈니스 규칙 변경이 자주 발생한다.
- 단순한 setter만으로 처리하기 어렵다.
- 잘못된 상태로 변경되지 않도록 검증이 필요하다.

예를 들어 주문 상태가 다음과 같이 많아질 수 있다.

```text
CREATED
PAYMENT_PENDING
PAID
PREPARING
SHIPPED
DELIVERED
CANCELED
REFUNDED
```

상태마다 가능한 행동이 달라지면 규칙도 복잡해진다.

```text
PAID 상태에서만 배송 준비 가능
SHIPPED 상태에서는 전체 취소 불가
DELIVERED 상태에서는 반품만 가능
CANCELED 상태에서는 결제 재시도 불가
```

---

## 7. 보험 서비스 예시

보험은 도메인 규칙이 복잡한 대표적인 분야다.

```text
나이에 따라 가입 가능 여부가 달라짐
여행 기간에 따라 보험료가 달라짐
방문 국가에 따라 가입 제한이 있음
상품별 보장 조건이 다름
가입자와 동반인의 조건이 다름
결제 완료 후에만 계약이 확정됨
계약 시작일 이후에는 변경이 제한됨
```

이 경우 단순히 데이터를 저장하는 것보다  
가입 가능 여부, 보험료 계산, 계약 확정 조건 같은 판단이 더 중요하다.

---

## 8. 도메인 엔티티와 JPA 엔티티를 꼭 분리해야 할까?

도메인 규칙이 복잡하다고 해서 반드시 도메인 엔티티와 JPA 엔티티를 분리해야 하는 것은 아니다.

다음처럼 하나의 객체가 두 역할을 함께 담당할 수도 있다.

```java
@Entity
public class Order {

    @Id
    private Long id;

    private OrderStatus status;

    public void cancel() {
        if (status == OrderStatus.SHIPPED) {
            throw new IllegalStateException(
                    "배송된 주문은 취소할 수 없습니다."
            );
        }

        this.status = OrderStatus.CANCELED;
    }
}
```

이 `Order`는 다음 역할을 동시에 수행한다.

```text
도메인 엔티티
+
JPA 엔티티
```

일반적인 Spring/JPA 프로젝트에서는 이런 방식도 충분히 실용적이다.

---

## 9. 분리를 고려해야 하는 시점

다음과 같은 문제가 생기면 도메인 엔티티와 JPA 엔티티의 분리를 고려할 수 있다.

### 1. JPA 제약 때문에 도메인 표현이 불편한 경우

```java
protected Order() {
}
```

```java
@OneToMany(fetch = FetchType.LAZY)
private List<OrderItem> orderItems;
```

JPA 기본 생성자, 지연 로딩, 연관관계 매핑 때문에  
도메인 객체 설계가 계속 영향을 받는 경우다.

### 2. DB 구조와 도메인 구조가 크게 다른 경우

비즈니스에서는 하나의 주문이지만 DB에서는 여러 테이블로 나뉠 수 있다.

```text
orders
order_items
order_payments
order_histories
```

도메인 구조와 DB 구조를 억지로 같게 맞추면  
도메인 모델이 오히려 복잡해질 수 있다.

### 3. 도메인을 저장 기술과 독립시키고 싶은 경우

도메인 객체가 다음 기술을 몰라도 되게 만들고 싶을 수 있다.

```text
JPA
Hibernate
MySQL
PostgreSQL
MongoDB
```

이때 도메인 엔티티와 영속성 객체를 분리한다.

---

## 10. 합쳐서 사용하는 경우

```text
Order
= 도메인 엔티티
= JPA 엔티티
```

### 장점

- 구조가 단순하다.
- 클래스 수가 적다.
- Mapper가 필요 없다.
- 개발 속도가 빠르다.
- 일반적인 CRUD 서비스에 적합하다.

### 단점

- 도메인 객체가 JPA에 의존한다.
- JPA 제약이 도메인 설계에 영향을 줄 수 있다.
- DB 구조와 도메인 구조가 다르면 표현이 어려울 수 있다.

---

## 11. 분리해서 사용하는 경우

```text
Order
= 도메인 엔티티

OrderJpaEntity
= JPA 영속성 객체
```

그리고 중간에 다음 객체들이 필요하다.

```text
OrderRepositoryAdapter
OrderMapper
```

### 장점

- 도메인이 JPA에 의존하지 않는다.
- 비즈니스 규칙에 집중할 수 있다.
- DB 구조와 도메인 구조를 독립적으로 설계할 수 있다.

### 단점

- 클래스 수가 늘어난다.
- 변환 코드가 필요하다.
- 같은 필드가 중복될 수 있다.
- 단순한 프로젝트에서는 과한 구조가 될 수 있다.

---

## 12. 현실적인 선택 기준

### 하나로 합쳐도 좋은 경우

```text
일반적인 CRUD 서비스
비즈니스 규칙이 단순함
DB 구조와 도메인 구조가 비슷함
빠른 개발이 중요함
팀 규모가 작음
```

이 경우에는 도메인 엔티티에 `@Entity`를 붙여도 충분하다.

### 분리를 고려할 경우

```text
비즈니스 규칙이 많고 복잡함
상태 변화와 조건이 많음
DB 구조와 도메인 구조가 크게 다름
JPA 제약이 도메인 설계를 방해함
도메인을 영속성 기술과 독립시키고 싶음
```

---

## 핵심 정리

> 도메인 규칙이 복잡하다는 것은 단순히 데이터를 저장하고 조회하는 것을 넘어서, 어떤 조건에서 어떤 행동이 가능하고 상태가 어떻게 변해야 하는지를 판단해야 할 내용이 많다는 뜻이다.

> 규칙이 조금 복잡하더라도 도메인 엔티티에 `@Entity`를 붙여 함께 사용할 수 있다.

> JPA 제약 때문에 도메인 표현이 불편해지거나, DB 구조와 도메인 구조가 크게 달라질 때 도메인 엔티티와 JPA 엔티티의 분리를 고려한다.

> 처음부터 무조건 분리하기보다, 실제 복잡성이 생겼을 때 분리하는 것이 현실적이다.
