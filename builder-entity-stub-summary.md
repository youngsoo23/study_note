# 빌더 패턴, 엔티티, Stub 정리

## 1. 빌더 패턴

빌더 패턴은 생성자에 많은 값을 한 번에 넣는 대신, 각 필드의 이름을 지정하면서 객체를 생성하는 방식이다.

```java
Member member = Member.builder()
        .name("오영수")
        .age(40)
        .address("서울")
        .build();
```

마지막의 `build()`를 호출하면 실제 객체가 생성된다.

### 장점

* 여러 개의 생성자를 만들지 않고 하나의 빌더로 관리할 수 있다.
* 생성자 파라미터가 많아도 각 값의 의미를 쉽게 알 수 있다.
* 파라미터 순서를 잘못 넣는 실수를 줄일 수 있다.
* 선택적인 값만 넣어서 객체를 만들기 편하다.

```java
new Member("오영수", 40, "서울", true);
```

위 생성자 방식은 각 값의 의미를 바로 알기 어렵다.

```java
Member.builder()
        .name("오영수")
        .age(40)
        .address("서울")
        .active(true)
        .build();
```

빌더 방식은 어떤 값이 어떤 필드에 들어가는지 명확하다.

### 단점

* 필수 파라미터를 누락해도 컴파일러가 잡지 못할 수 있다.
* 값 검증을 하지 않으면 잘못된 상태의 객체가 생성될 수 있다.
* 필드가 적은 단순한 객체에서는 코드가 오히려 길어질 수 있다.

```java
Member member = Member.builder()
        .age(40)
        .build();
```

`name`이 필수값이어도 컴파일 오류가 발생하지 않을 수 있다.

### 핵심 정리

> 빌더 패턴은 많은 생성자와 긴 파라미터를 정리해 가독성을 높여주지만, 필수값 누락을 컴파일 시점에 발견하기 어렵다는 단점이 있다.

---

## 2. 도메인과 엔티티

### 도메인

도메인은 소프트웨어가 해결하려는 업무나 문제 영역을 의미한다.

예를 들어 쇼핑몰의 도메인에는 다음과 같은 개념이 있다.

* 회원
* 주문
* 상품
* 결제
* 배송

보험 서비스라면 다음과 같은 개념이 도메인에 포함된다.

* 보험 계약
* 가입자
* 보험 상품
* 보장
* 보험료

---

## 3. 엔티티

엔티티는 고유한 식별자를 가지고 있으며, 속성이 변해도 같은 대상으로 구분되는 객체이다.

예를 들어 주문 상태가 변해도 주문 ID가 같으면 같은 주문이다.

```text
주문 ID: 100
상태: 결제 완료
```

배송이 완료된 후에도:

```text
주문 ID: 100
상태: 배송 완료
```

주문 ID가 같기 때문에 같은 주문이다.

### 핵심 정리

> 엔티티의 핵심은 DB가 아니라 식별자와 정체성이다.

---

## 4. 도메인 엔티티

도메인 엔티티는 비즈니스에서 중요한 대상을 표현하며, 고유한 식별자와 비즈니스 규칙을 가진 객체이다.

```java
public class Order {

    private Long id;
    private OrderStatus status;

    public void cancel() {
        if (status == OrderStatus.SHIPPED) {
            throw new IllegalStateException(
                    "배송된 주문은 취소할 수 없습니다."
            );
        }

        status = OrderStatus.CANCELED;
    }
}
```

이 `Order` 객체는 다음 내용을 표현한다.

* 주문 ID
* 주문 상태
* 배송된 주문은 취소할 수 없다는 비즈니스 규칙

따라서 도메인 엔티티는 단순히 데이터를 저장하는 객체가 아니다.

### DTO와의 차이

DTO는 데이터를 전달하는 것이 주요 목적이다.

```java
public class OrderResponse {

    private Long orderId;
    private String status;
}
```

도메인 엔티티는 상태와 함께 비즈니스 행동과 규칙을 가진다.

```java
public class Order {

    private Long id;
    private OrderStatus status;

    public void cancel() {
        // 주문 취소 규칙
    }
}
```

### 핵심 정리

> 도메인 엔티티는 비즈니스 개념, 식별자, 상태, 행동과 규칙을 표현하는 객체이다.

---

## 5. DB 엔티티

DB 엔티티는 데이터베이스에서 저장하고 관리해야 하는 대상을 의미한다.

예를 들면 다음과 같다.

* 회원
* 주문
* 상품

관계형 데이터베이스에서는 이러한 엔티티가 보통 테이블로 구현된다.

```text
회원 엔티티 → member 테이블
주문 엔티티 → orders 테이블
상품 엔티티 → product 테이블
```

하지만 엔티티와 테이블은 완전히 같은 개념은 아니다.

엔티티는 저장하고 관리할 대상에 대한 개념이고, 테이블은 그 개념을 관계형 데이터베이스에 구현한 결과에 가깝다.

---

## 6. 영속성 객체

영속성 객체는 메모리에만 존재하는 것이 아니라, DB에 저장되고 다시 조회될 수 있는 객체이다.

JPA에서는 보통 `@Entity`가 붙은 클래스를 영속성 객체라고 부른다.

```java
@Entity
@Table(name = "orders")
public class OrderJpaEntity {

    @Id
    private Long id;

    private String status;
}
```

구조는 다음과 같다.

```text
OrderJpaEntity
      ↕
JPA / Hibernate
      ↕
orders 테이블
```

### 핵심 정리

> 영속성 객체는 애플리케이션의 데이터를 DB에 저장하고 조회하기 위해 사용하는 객체이다.

---

## 7. 도메인 엔티티에 `@Entity`를 붙여도 되는가?

도메인 엔티티에 JPA의 `@Entity`를 붙이는 것은 잘못이 아니다.

실무에서는 하나의 클래스가 도메인 엔티티와 JPA 엔티티 역할을 함께 담당하는 경우가 많다.

```java
@Entity
public class Order {

    @Id
    private Long id;

    private OrderStatus status;

    public void cancel() {
        if (status == OrderStatus.SHIPPED) {
            throw new IllegalStateException();
        }

        status = OrderStatus.CANCELED;
    }
}
```

이 `Order`는 다음 두 역할을 모두 한다.

* 비즈니스 규칙을 가진 도메인 엔티티
* DB 테이블과 매핑되는 JPA 엔티티

### 일부에서 반대하는 이유

DDD나 클린 아키텍처를 엄격하게 적용하는 사람들은 도메인 엔티티에 `@Entity`를 붙이는 것을 반대하기도 한다.

이유는 도메인 객체가 JPA라는 저장 기술에 의존하게 되기 때문이다.

```java
@Entity
@Table(name = "orders")
public class Order {

    @Id
    @GeneratedValue
    @Column(name = "order_id")
    private Long id;
}
```

이제 `Order`는 비즈니스 규칙 외에도 다음 내용을 알게 된다.

* DB 테이블 이름
* 컬럼 이름
* PK 생성 방식
* JPA 기본 생성자 규칙
* 지연 로딩과 연관관계 설정

이것을 도메인과 영속성 기술의 결합이라고 본다.

---

## 8. 도메인 엔티티와 JPA 엔티티를 분리하는 방식

도메인 모델을 순수하게 유지하려면 두 객체를 분리할 수 있다.

### 도메인 엔티티

```java
public class Order {

    private final OrderId id;
    private OrderStatus status;

    public void cancel() {
        if (status == OrderStatus.SHIPPED) {
            throw new IllegalStateException();
        }

        status = OrderStatus.CANCELED;
    }
}
```

### JPA 영속성 객체

```java
@Entity
@Table(name = "orders")
public class OrderJpaEntity {

    @Id
    private Long id;

    private String status;

    protected OrderJpaEntity() {
    }
}
```

### 구조

```text
도메인 엔티티
      ↕ 변환
JPA 영속성 객체
      ↕
DB 테이블
```

이 방식에서는 별도의 변환 코드가 필요하다.

```java
public Order toDomain() {
    return new Order(
            new OrderId(id),
            OrderStatus.valueOf(status)
    );
}
```

### 분리 방식의 장점

* 도메인 코드가 JPA를 몰라도 된다.
* 비즈니스 규칙에 집중할 수 있다.
* DB 구조와 도메인 구조를 독립적으로 설계할 수 있다.

### 분리 방식의 단점

* 클래스 수가 늘어난다.
* 변환 코드가 필요하다.
* 같은 필드가 중복될 수 있다.
* 단순한 CRUD 프로젝트에서는 과한 설계가 될 수 있다.

### 실무적인 기준

단순한 CRUD 프로젝트에서는 도메인 엔티티와 JPA 엔티티를 합쳐서 사용해도 괜찮다.

비즈니스 규칙이 복잡하거나, 도메인과 DB 구조가 크게 다르거나, 클린 아키텍처를 엄격하게 적용한다면 분리를 고려할 수 있다.

### 핵심 정리

> 도메인 엔티티에 `@Entity`를 붙이는 것이 무조건 잘못은 아니다.

> 다만 `@Entity`를 붙이면 도메인 객체가 JPA라는 영속성 기술에 의존하게 된다.

---

## 9. Stub

Stub은 테스트할 때 실제 객체 대신 사용하는 가짜 객체이다.

Stub은 요청을 받으면 미리 정해둔 값을 반환한다.

```java
class PaymentStub implements PaymentClient {

    @Override
    public boolean pay(int amount) {
        return true;
    }
}
```

테스트에서는 실제 결제 API를 호출하지 않고 항상 결제 성공을 반환하도록 만들 수 있다.

```java
PaymentClient paymentClient = new PaymentStub();

boolean result = paymentClient.pay(10_000);
// 항상 true
```

### Stub을 사용하는 이유

실제 외부 시스템을 테스트에서 호출하면 다음과 같은 문제가 생길 수 있다.

* 네트워크 상태에 따라 테스트가 실패할 수 있다.
* 테스트 속도가 느려질 수 있다.
* 실제 결제가 발생할 수 있다.
* 외부 서버가 중단되면 테스트가 실패한다.
* 원하는 성공 또는 실패 상황을 만들기 어렵다.

그래서 테스트 대상이 아닌 외부 의존성은 Stub으로 바꾼다.

```text
주문 서비스 테스트
      ↓
실제 결제 API 호출하지 않음
      ↓
PaymentStub이 미리 정한 결과 반환
```

예를 들어 다음과 같이 성공 Stub과 실패 Stub을 만들 수 있다.

```java
class PaymentSuccessStub implements PaymentClient {

    @Override
    public boolean pay(int amount) {
        return true;
    }
}
```

```java
class PaymentFailStub implements PaymentClient {

    @Override
    public boolean pay(int amount) {
        return false;
    }
}
```

### Stub 처리했다는 의미

“결제 API를 Stub 처리했다”는 말은 다음을 의미한다.

> 실제 결제 API를 호출하지 않고, 테스트를 위해 성공이나 실패 응답을 미리 만들어 사용했다.

### 핵심 정리

> Stub은 테스트에서 실제 객체 대신 사용하며, 호출되면 미리 정해둔 결과를 반환하는 가짜 객체이다.

---

# 전체 요약

## 빌더 패턴

객체를 생성할 때 필드 이름을 지정하면서 값을 설정하는 방식이다.

```java
Member.builder()
        .name("오영수")
        .age(40)
        .build();
```

가독성은 좋아지지만 필수값 누락을 컴파일러가 잡지 못할 수 있다.

## 도메인 엔티티

비즈니스에서 중요한 대상을 표현하며, 식별자와 비즈니스 규칙을 가진 객체이다.

```java
order.cancel();
```

## JPA 엔티티

DB 테이블과 매핑하기 위해 `@Entity`가 붙은 객체이다.

```java
@Entity
public class OrderJpaEntity {
}
```

## 도메인 엔티티와 JPA 엔티티

하나의 클래스로 합쳐서 사용할 수도 있고, 각각 분리할 수도 있다.

```text
단순한 프로젝트 → 합쳐서 사용 가능
복잡한 도메인 → 분리 고려
```

## Stub

테스트에서 실제 객체 대신 사용하며, 미리 정해둔 값을 반환하는 가짜 객체이다.

```java
whenPaymentRequestedReturnSuccess();
```
