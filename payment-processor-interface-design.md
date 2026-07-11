# 결제 처리 인터페이스 설계 정리

## 1. 구현체를 먼저 만들 때 생길 수 있는 문제

구현체를 먼저 만드는 것 자체가 항상 잘못은 아니다.  
다만 구현체의 세부사항을 기준으로 전체 구조가 만들어지면, 사용하는 쪽이 특정 구현에 강하게 의존할 수 있다.

예를 들어 주문 서비스가 토스 결제 구현체를 직접 사용한다고 해보자.

```java
@Service
public class OrderService {

    private final TossPaymentService tossPaymentService;

    public OrderService(TossPaymentService tossPaymentService) {
        this.tossPaymentService = tossPaymentService;
    }

    public void order(Long orderId, int amount) {
        tossPaymentService.tossPay(orderId, amount);
    }
}
```

이 구조에서 `OrderService`는 다음 내용을 알고 있다.

- 결제 수단이 토스라는 것
- 토스 결제 메서드 이름이 `tossPay`라는 것
- 토스 구현체가 요구하는 파라미터 구조

카카오페이나 다른 결제 수단이 추가되면 `OrderService`까지 수정해야 한다.

---

## 2. 역할을 인터페이스로 정의하기

주문 서비스가 실제로 필요한 것은 특정 결제사가 아니라, 다음과 같은 결제 기능이다.

> 결제를 요청하고 그 결과를 받는다.

이를 인터페이스로 표현할 수 있다.

```java
public interface PaymentProcessor {

    PaymentMethod supports();

    PaymentResult pay(PaymentCommand command);
}
```

결제 수단은 enum으로 정의한다.

```java
public enum PaymentMethod {
    TOSS,
    KAKAO_PAY,
    NAVER_PAY
}
```

공통으로 필요한 결제 요청 값도 내부 모델로 정의한다.

```java
public record PaymentCommand(
    Long orderId,
    int amount,
    String orderName,
    String customerId
) {
}
```

```java
public record PaymentResult(
    boolean success,
    String transactionId
) {
}
```

---

## 3. 토스 결제 구현체

```java
@Component
public class TossPaymentProcessor implements PaymentProcessor {

    private final TossPaymentClient tossPaymentClient;

    public TossPaymentProcessor(
        TossPaymentClient tossPaymentClient
    ) {
        this.tossPaymentClient = tossPaymentClient;
    }

    @Override
    public PaymentMethod supports() {
        return PaymentMethod.TOSS;
    }

    @Override
    public PaymentResult pay(PaymentCommand command) {
        TossPaymentRequest request = new TossPaymentRequest(
            command.orderId().toString(),
            command.amount(),
            command.orderName()
        );

        TossPaymentResponse response =
            tossPaymentClient.pay(request);

        return new PaymentResult(
            response.isSuccess(),
            response.paymentKey()
        );
    }
}
```

토스 API의 요청·응답 객체는 `TossPaymentProcessor` 내부에서만 사용한다.

---

## 4. 카카오페이 구현체 추가

카카오페이가 추가되면 같은 인터페이스를 구현하는 클래스를 하나 더 만든다.

```java
@Component
public class KakaoPaymentProcessor implements PaymentProcessor {

    private final KakaoPaymentClient kakaoPaymentClient;

    public KakaoPaymentProcessor(
        KakaoPaymentClient kakaoPaymentClient
    ) {
        this.kakaoPaymentClient = kakaoPaymentClient;
    }

    @Override
    public PaymentMethod supports() {
        return PaymentMethod.KAKAO_PAY;
    }

    @Override
    public PaymentResult pay(PaymentCommand command) {
        KakaoReadyRequest request = new KakaoReadyRequest(
            command.customerId(),
            command.orderId().toString(),
            command.orderName(),
            command.amount()
        );

        KakaoReadyResponse response =
            kakaoPaymentClient.ready(request);

        return new PaymentResult(
            true,
            response.tid()
        );
    }
}
```

토스와 카카오페이의 실제 API 요청·응답 구조가 달라도 괜찮다.

각 구현체 내부에서 공통 모델인 `PaymentCommand`를 외부 결제사의 요청 객체로 변환하고, 외부 응답을 `PaymentResult`로 변환하면 된다.

---

## 5. 결제 구현체 선택하기

구현체가 여러 개가 되면 결제 수단에 맞는 구현체를 선택하는 객체가 필요하다.

```java
@Component
public class PaymentProcessorResolver {

    private final Map<PaymentMethod, PaymentProcessor> processors;

    public PaymentProcessorResolver(
        List<PaymentProcessor> paymentProcessors
    ) {
        this.processors = paymentProcessors.stream()
            .collect(Collectors.toUnmodifiableMap(
                PaymentProcessor::supports,
                Function.identity()
            ));
    }

    public PaymentProcessor resolve(
        PaymentMethod paymentMethod
    ) {
        PaymentProcessor processor =
            processors.get(paymentMethod);

        if (processor == null) {
            throw new IllegalArgumentException(
                "지원하지 않는 결제수단입니다: "
                    + paymentMethod
            );
        }

        return processor;
    }
}
```

Spring이 모든 `PaymentProcessor` 구현체를 `List<PaymentProcessor>`로 주입한다.

생성자에서는 이를 다음과 같은 Map 구조로 변환한다.

```text
TOSS       -> TossPaymentProcessor
KAKAO_PAY  -> KakaoPaymentProcessor
NAVER_PAY  -> NaverPaymentProcessor
```

---

## 6. 주문 서비스에서 사용하기

```java
@Service
public class OrderService {

    private final PaymentProcessorResolver processorResolver;

    public OrderService(
        PaymentProcessorResolver processorResolver
    ) {
        this.processorResolver = processorResolver;
    }

    public PaymentResult order(
        Long orderId,
        int amount,
        String orderName,
        String customerId,
        PaymentMethod paymentMethod
    ) {
        PaymentProcessor processor =
            processorResolver.resolve(paymentMethod);

        PaymentCommand command = new PaymentCommand(
            orderId,
            amount,
            orderName,
            customerId
        );

        return processor.pay(command);
    }
}
```

이제 `OrderService`는 다음 내용을 모른다.

- 토스 API의 요청 형식
- 카카오페이 API의 요청 형식
- 결제사별 인증 방식
- 결제사별 응답 필드
- 각 결제 구현체의 클래스 이름

주문 서비스는 단지 결제수단을 전달하고, 공통 인터페이스로 결제를 요청한다.

---

## 7. 네이버페이가 추가되는 경우

네이버페이가 추가되더라도 새 구현체만 만들면 된다.

```java
@Component
public class NaverPaymentProcessor implements PaymentProcessor {

    @Override
    public PaymentMethod supports() {
        return PaymentMethod.NAVER_PAY;
    }

    @Override
    public PaymentResult pay(PaymentCommand command) {
        // 네이버페이 API 호출

        return new PaymentResult(
            true,
            "naver-transaction-id"
        );
    }
}
```

`OrderService`와 `PaymentProcessorResolver`는 수정할 필요가 없다.

단, `PaymentMethod` enum에는 새로운 결제수단을 추가해야 한다.

```java
public enum PaymentMethod {
    TOSS,
    KAKAO_PAY,
    NAVER_PAY
}
```

---

## 8. 전체 구조

```text
OrderService
    |
    v
PaymentProcessorResolver
    |
    +-- TOSS -------> TossPaymentProcessor
    |
    +-- KAKAO_PAY --> KakaoPaymentProcessor
    |
    +-- NAVER_PAY --> NaverPaymentProcessor
```

실제 외부 API까지 포함하면 다음과 같다.

```text
OrderService
    |
    v
PaymentProcessor
    |
    +-- TossPaymentProcessor
    |       |
    |       v
    |   TossPaymentClient
    |       |
    |       v
    |   Toss Payments API
    |
    +-- KakaoPaymentProcessor
            |
            v
        KakaoPaymentClient
            |
            v
        KakaoPay API
```

---

## 9. 이 구조의 장점

### 주문 서비스가 결제사 구현을 모른다

`OrderService`는 `TossPaymentProcessor`나 `KakaoPaymentProcessor`에 직접 의존하지 않는다.

### 새로운 결제수단을 추가하기 쉽다

새로운 `PaymentProcessor` 구현체를 추가하면 된다.

### 외부 API 변경의 영향 범위가 작다

토스 API 응답 구조가 변경되면 주로 `TossPaymentProcessor`와 `TossPaymentClient`만 수정한다.

### 테스트하기 쉽다

테스트용 구현체를 만들어 실제 외부 결제를 호출하지 않고 주문 로직을 검증할 수 있다.

```java
public class FakePaymentProcessor
    implements PaymentProcessor {

    @Override
    public PaymentMethod supports() {
        return PaymentMethod.TOSS;
    }

    @Override
    public PaymentResult pay(PaymentCommand command) {
        return new PaymentResult(
            true,
            "fake-transaction-id"
        );
    }
}
```

### 각 구현체가 자신의 변환 책임을 가진다

토스 요청 객체와 카카오 요청 객체의 구조가 달라도 각 구현체 내부에서 처리한다.

---

## 10. 주의할 점

모든 서비스에 인터페이스를 만들 필요는 없다.

다음처럼 구현체가 하나뿐이고 대체 가능성이나 명확한 경계가 없다면, 인터페이스가 불필요할 수 있다.

```java
public interface MemberService {
    Member find(Long memberId);
}

@Service
public class MemberServiceImpl
    implements MemberService {
}
```

단순히 `Service`와 `ServiceImpl`로 나누는 것은 좋은 추상화가 아니다.

인터페이스는 주로 다음 상황에서 유용하다.

- 외부 API 연동
- 구현 전략이 여러 개인 경우
- 런타임에 구현체를 선택해야 하는 경우
- 테스트에서 구현체를 대체해야 하는 경우
- 도메인과 인프라의 경계를 분리하는 경우

---

## 핵심 정리

인터페이스를 먼저 생각한다는 것은 Java의 `interface` 파일을 무조건 먼저 작성한다는 뜻이 아니다.

핵심은 다음 질문을 구현 전에 고민하는 것이다.

> 이 기능을 사용하는 쪽은 구체적인 구현체가 아니라 어떤 역할과 계약을 필요로 하는가?

결제 예제에서는 주문 서비스가 원하는 것이 토스 결제나 카카오페이 결제가 아니라, 공통적인 결제 처리 기능이다.

따라서 다음처럼 역할을 분리한다.

```text
PaymentProcessor         : 결제 처리 계약
TossPaymentProcessor     : 토스 결제 구현
KakaoPaymentProcessor    : 카카오페이 구현
PaymentProcessorResolver : 결제수단에 맞는 구현체 선택
OrderService             : 결제 기능을 사용하는 애플리케이션 서비스
```

이 구조는 전략 패턴과 의존성 역전 원칙이 적용된 형태로 볼 수 있다.
