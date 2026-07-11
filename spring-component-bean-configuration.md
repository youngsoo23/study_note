# Spring `@Component`, `@Bean`, `@Configuration` 쉽게 이해하기

## 1. 먼저 스프링 빈이란?

스프링 빈은 쉽게 말해:

> 스프링이 대신 만들어서 보관하고 관리하는 객체

입니다.

일반 자바에서는 직접 객체를 생성합니다.

```java
PaymentService paymentService = new PaymentService();
```

하지만 스프링에서는 클래스에 어노테이션을 붙이면 스프링이 객체를 생성하고 관리합니다.

```java
@Service
public class PaymentService {
}
```

그리고 필요한 곳에 생성자 주입으로 전달합니다.

```java
@RestController
public class PaymentController {

    private final PaymentService paymentService;

    public PaymentController(PaymentService paymentService) {
        this.paymentService = paymentService;
    }
}
```

여기서 스프링이 생성하고 관리하는 `PaymentService` 객체가 **스프링 빈**입니다.

---

## 2. `@Component`란?

```java
@Component
public class JwtTokenProvider {
}
```

`@Component`는 스프링에게 다음과 같이 말하는 것입니다.

> 이 클래스를 발견하면 객체로 만들어서 관리해줘.

스프링이 내부적으로 다음과 비슷한 일을 한다고 생각하면 됩니다.

```java
JwtTokenProvider jwtTokenProvider = new JwtTokenProvider();
```

즉, `@Component`는 **클래스 자체를 빈으로 등록하는 방식**입니다.

### `@Component` 계열 어노테이션

다음 어노테이션들도 모두 `@Component`의 역할을 포함합니다.

- `@Service`: 비즈니스 로직
- `@Repository`: 데이터 접근
- `@Controller`: 화면 요청 처리
- `@RestController`: REST API 요청 처리
- `@Component`: 그 외 일반 컴포넌트

예시:

```java
@Service
public class PaymentService {
}
```

```java
@Repository
public class PaymentRepository {
}
```

```java
@Component
public class JwtTokenProvider {
}
```

---

## 3. `@Bean`이란?

`@Bean`은 메서드가 반환한 객체를 스프링 빈으로 등록합니다.

```java
@Bean
public PasswordEncoder passwordEncoder() {
    return new BCryptPasswordEncoder();
}
```

이 코드는 스프링에게 다음과 같이 말합니다.

> 이 메서드를 실행해서 반환된 객체를 스프링이 관리해줘.

즉, 다음 객체가 빈으로 등록됩니다.

```java
new BCryptPasswordEncoder()
```

### 차이

```text
@Component
→ 클래스에 붙인다.

@Bean
→ 메서드에 붙인다.
```

---

## 4. `@Configuration`이란?

`@Configuration`은 보통 `@Bean` 메서드를 모아두는 설정 클래스에 붙입니다.

```java
@Configuration
public class SecurityConfig {

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
}
```

의미는 다음과 같습니다.

> 이 클래스 안에는 스프링 빈을 생성하고 등록하는 설정이 들어있다.

구조는 보통 다음과 같습니다.

```java
@Configuration
public class 설정클래스 {

    @Bean
    public 빈타입 빈이름() {
        return new 객체();
    }
}
```

---

## 5. 같은 객체를 등록하는 두 가지 방법

예를 들어 다음 클래스가 있다고 가정합니다.

```java
public class TossPaymentProcessor {
}
```

### 방법 1: `@Component` 사용

```java
@Component
public class TossPaymentProcessor {
}
```

스프링이 클래스를 찾아 자동으로 객체를 생성합니다.

### 방법 2: `@Configuration`과 `@Bean` 사용

```java
public class TossPaymentProcessor {
}
```

```java
@Configuration
public class PaymentConfig {

    @Bean
    public TossPaymentProcessor tossPaymentProcessor() {
        return new TossPaymentProcessor();
    }
}
```

이번에는 개발자가 직접 객체 생성 코드를 작성합니다.

둘 다 결과적으로 `TossPaymentProcessor` 빈이 등록됩니다.

```text
@Component
→ 스프링이 클래스를 찾아 자동 생성

@Bean
→ 개발자가 객체 생성 코드를 작성하고,
   스프링이 그 결과를 관리
```

---

## 6. `@Configuration`을 써야 할 때 `@Component`를 쓰면 안 되나?

반드시 안 되는 것은 아닙니다.

내가 만든 단순한 클래스라면 두 방식 모두 가능합니다.

```java
@Component
public class TossPaymentProcessor {
}
```

또는:

```java
@Configuration
public class PaymentConfig {

    @Bean
    public TossPaymentProcessor tossPaymentProcessor() {
        return new TossPaymentProcessor();
    }
}
```

다만 실무에서는 역할에 맞는 더 자연스러운 방식을 선택합니다.

---

## 7. `@Component` 계열을 쓰는 경우

클래스 자체가 실제 업무를 수행하는 객체라면 보통 `@Component` 계열을 사용합니다.

```java
@Service
public class PaymentService {
}
```

```java
@Repository
public class PaymentRepository {
}
```

```java
@Component
public class JwtTokenProvider {
}
```

이 클래스들은 특별한 생성 설정 없이 스프링이 바로 객체로 만들면 됩니다.

---

## 8. `@Configuration + @Bean`을 쓰는 경우

### 8.1 외부 라이브러리 객체를 빈으로 등록할 때

다음 클래스들은 내가 만든 클래스가 아닙니다.

- `ObjectMapper`
- `WebClient`
- `BCryptPasswordEncoder`

외부 라이브러리 클래스 소스에 직접 `@Component`를 붙일 수 없습니다.

그래서 설정 클래스에서 객체를 생성하고 빈으로 등록합니다.

```java
@Configuration
public class SecurityConfig {

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
}
```

### 8.2 객체를 특별한 설정과 함께 생성할 때

예를 들어 결제 API용 `WebClient`를 만든다고 가정합니다.

```java
@Configuration
public class WebClientConfig {

    @Bean
    public WebClient paymentWebClient() {
        return WebClient.builder()
                .baseUrl("https://payment.example.com")
                .defaultHeader("Authorization", "secret-key")
                .build();
    }
}
```

여기서 의미는 다음과 같습니다.

- 어떤 객체: `WebClient`
- 어떻게 생성: URL과 헤더를 설정해서 생성
- 등록: 완성된 객체를 스프링 빈으로 등록

이것이 바로:

> 어떤 객체를 어떻게 생성해서 등록할지 설정한다.

라는 뜻입니다.

---

## 9. 판단 기준

### 클래스 자체가 일을 한다면

```java
@Service
public class PaymentService {
}
```

```java
@Repository
public class PaymentRepository {
}
```

```java
@Component
public class JwtTokenProvider {
}
```

보통 `@Component` 계열을 사용합니다.

### 객체를 만드는 방법을 직접 작성해야 한다면

```java
@Configuration
public class AppConfig {

    @Bean
    public ObjectMapper objectMapper() {
        return new ObjectMapper()
                .registerModule(new JavaTimeModule());
    }
}
```

보통 `@Configuration + @Bean`을 사용합니다.

---

## 10. 중요한 점

`@Component`와 `@Configuration`은 완전히 반대되는 개념이 아닙니다.

`@Configuration`도 내부적으로 `@Component`의 성격을 가집니다.

```java
@Configuration
public class AppConfig {
}
```

`AppConfig` 자체도 스프링 빈으로 등록됩니다.

다만 목적이 다릅니다.

```text
@Component
→ 이 클래스 자체를 빈으로 등록

@Configuration
→ 다른 빈들을 생성하고 등록하는 설정 클래스
```

---

## 11. 중복 등록 주의

다음처럼 같은 클래스를 두 방식으로 동시에 등록하면 빈이 두 개 생길 수 있습니다.

```java
@Component
public class TossPaymentProcessor {
}
```

```java
@Configuration
public class PaymentConfig {

    @Bean
    public TossPaymentProcessor tossPaymentProcessor() {
        return new TossPaymentProcessor();
    }
}
```

특별한 이유가 없다면 한 가지 방식만 사용하는 것이 좋습니다.

---

## 12. 최종 정리

```text
@Component
= 이 클래스의 객체를 스프링이 만들어줘

@Bean
= 이 메서드가 반환한 객체를 스프링이 관리해줘

@Configuration
= @Bean 메서드들을 모아놓은 설정 클래스
```

실무 기준:

```text
내가 만든 Service, Repository, Provider
→ @Service, @Repository, @Component

ObjectMapper, WebClient, PasswordEncoder처럼
직접 생성하거나 설정해서 등록해야 하는 객체
→ @Configuration + @Bean
```
