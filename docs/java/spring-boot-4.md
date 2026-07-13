# Spring Boot 4 / Spring Framework 7

Reach for this to use the modern Spring APIs - no Boot 2.x/3.x habits, no `RestTemplate`,
no hand-rolled error POJOs.

## Constructor injection (never field/setter)

```java
@Service
public class OrderService {
    private final OrderRepository orderRepository;
    private final PaymentClient paymentClient;

    public OrderService(OrderRepository orderRepository, PaymentClient paymentClient) {
        this.orderRepository = orderRepository;
        this.paymentClient = paymentClient;
    }
}
```

Single constructor → no `@Autowired` needed. Fields are `final`. More than 4 params = design smell.

## `RestClient` (imperative HTTP - never `RestTemplate`)

```java
@Bean
RestClient paymentRestClient(RestClient.Builder builder) {
    return builder.baseUrl("https://payments.example.com").build();
}

// Use it
PaymentResult result = restClient.post()
    .uri("/charges")
    .body(new ChargeRequest(amount, currency))
    .retrieve()
    .body(PaymentResult.class);

// Error handling
restClient.get()
    .uri("/charges/{id}", id)
    .retrieve()
    .onStatus(HttpStatusCode::is4xxClientError, (req, res) -> {
        throw new PaymentNotFoundException(id);
    })
    .body(PaymentResult.class);
```

## `@HttpExchange` (declarative HTTP interfaces)

```java
@HttpExchange(url = "/api", accept = "application/json")
public interface PaymentApi {

    @GetExchange("/charges/{id}")
    PaymentResult getCharge(@PathVariable String id);

    @PostExchange("/charges")
    PaymentResult charge(@RequestBody ChargeRequest request);

    @GetExchange(value = "/charges", version = "2")   // SF7 API versioning on the client
    List<PaymentResult> list();
}

@Bean
PaymentApi paymentApi(RestClient.Builder builder) {
    RestClient client = builder.baseUrl("https://payments.example.com").build();
    return HttpServiceProxyFactory
        .builderFor(RestClientAdapter.create(client))
        .build()
        .createClient(PaymentApi.class);
}
```

Declarative interface, Spring generates the implementation. Prefer this over hand-written clients.

## Controllers - `ResponseEntity`, `@Valid`, DTOs

```java
@RestController
@RequestMapping("/api/orders")
public class OrderController {
    private final OrderService orderService;

    public OrderController(OrderService orderService) {
        this.orderService = orderService;
    }

    @PostMapping
    public ResponseEntity<OrderResponse> create(@Valid @RequestBody CreateOrderRequest request) {
        var response = orderService.create(request);
        return ResponseEntity.status(HttpStatus.CREATED).body(response);
    }

    @GetMapping("/{id}")
    public ResponseEntity<OrderResponse> get(@PathVariable Long id) {
        return ResponseEntity.ok(orderService.get(id));
    }
}
```

Controllers do HTTP only: validate, map to/from DTOs, set explicit status. No business logic, no try/catch.

## Request/response DTOs (records + validation on components)

```java
public record CreateOrderRequest(
    @NotBlank String customerId,
    @NotNull @Positive BigDecimal amount,
    @Email String contactEmail,
    @Size(min = 1) List<@Valid OrderLine> lines
) {}

public record OrderResponse(Long id, String customerId, BigDecimal amount, OrderStatus status) {}
```

Separate request and response records. Validation annotations go **on the components** - not a separate validator, not re-checked in the service.

## `ProblemDetail` (RFC 9457 error responses)

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(OrderNotFoundException.class)
    public ProblemDetail handleNotFound(OrderNotFoundException ex) {
        var problem = ProblemDetail.forStatusAndDetail(HttpStatus.NOT_FOUND, ex.getMessage());
        problem.setTitle("Order not found");
        problem.setType(URI.create("https://api.example.com/errors/order-not-found"));
        problem.setProperty("orderId", ex.getOrderId());
        return problem;
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ProblemDetail handleValidation(MethodArgumentNotValidException ex) {
        var problem = ProblemDetail.forStatusAndDetail(HttpStatus.BAD_REQUEST, "Validation failed");
        problem.setProperty("errors", ex.getBindingResult().getFieldErrors().stream()
            .collect(Collectors.toMap(FieldError::getField, FieldError::getDefaultMessage)));
        return problem;
    }
}
```

One `@RestControllerAdvice` for all exceptions, consistent shape. No custom error POJOs.

## Resilience (SF7 - no Resilience4j for basic cases)

```java
@Configuration
@EnableResilientMethods
class ResilienceConfig {}

@Service
class InventoryService {

    @Retryable(maxAttempts = 3, delay = 200, multiplier = 2)   // exponential backoff
    public Stock check(String sku) {
        return inventoryClient.get(sku);
    }

    @ConcurrencyLimit(10)                     // cap concurrent executions
    public void reindex() { ... }

    @Recover
    public Stock fallback(RemoteAccessException ex, String sku) {
        return Stock.unknown(sku);            // called when retries are exhausted
    }
}
```

## REST API versioning (first-class in SF7)

```java
@GetMapping(value = "/orders", version = "1")
public List<OrderV1> listV1() { ... }

@GetMapping(value = "/orders", version = "2")
public List<OrderV2> listV2() { ... }
```
```properties
# Choose the strategy (path / header / query param / media type)
spring.mvc.apiversion.use.header=X-API-Version
```

Don't hand-roll versioning - configure the strategy and annotate with `version`.

## Repositories - `ListCrudRepository`

```java
public interface OrderRepository extends ListCrudRepository<Order, Long> {
    List<Order> findByCustomerId(String customerId);   // returns List, not Iterable
    Optional<Order> findByReference(String reference);
}
```

`ListCrudRepository` returns `List` where `CrudRepository` returns `Iterable`. Interfaces only - no query logic in service/controller.

## Flyway + `ddl-auto: validate`

```
src/main/resources/db/migration/
  V1__create_orders.sql
  V2__add_status_to_orders.sql
```
```yaml
spring:
  jpa:
    hibernate:
      ddl-auto: validate     # Hibernate validates against the schema, never changes it
  flyway:
    enabled: true
```

See the [Flyway sheet](../data/flyway.md) for conventions.

## Observability - Actuator + Micrometer over OpenTelemetry

```xml
<dependency><groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-actuator</artifactId></dependency>
<dependency><groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-opentelemetry</artifactId></dependency>
```
```yaml
management:
  endpoints.web.exposure.include: health,info,metrics,prometheus
  endpoint.health.probes.enabled: true      # liveness/readiness for k8s
  otlp.metrics.export.url: http://collector:4318/v1/metrics
  tracing.sampling.probability: 1.0
```

Micrometer signals exported via OTLP; auto-instruments HTTP/JDBC/logs. Every project gets this.

## Config properties (type-safe)

```java
@ConfigurationProperties(prefix = "payment")
public record PaymentProperties(String baseUrl, Duration timeout, int maxRetries) {}
```
```java
@EnableConfigurationProperties(PaymentProperties.class)   // on a @Configuration class
```

## Gotchas / things I always forget

- **`RestTemplate` is legacy** - `RestClient` for imperative, `WebClient` only if you're reactive.
- Entities must never leave the service layer - always map to a DTO first.
- `@RestControllerAdvice` = `@ControllerAdvice` + `@ResponseBody`; returning `ProblemDetail` gives you `application/problem+json` automatically.
- Validation on a record component only fires with `@Valid` on the controller param - Spring doesn't re-validate in the service.
- `@Retryable` needs `@EnableResilientMethods` and a proxied bean - self-invocation (calling the method from within the same class) bypasses the proxy and the retry.
- `ddl-auto: validate` fails fast at startup if the schema doesn't match the entities - that's the point; fix the migration, not the entity.
- Nested DTO validation needs `@Valid` on the field/element (`List<@Valid OrderLine>`), or nested constraints are skipped.
- One constructor → no `@Autowired`; add a second constructor and Spring won't know which to use without it.

## Quick reference

| Need | Use |
|---|---|
| Imperative HTTP | `RestClient` |
| Declarative HTTP client | `@HttpExchange` interface |
| Error response | `ProblemDetail` (RFC 9457) |
| Retry / limit | `@Retryable` / `@ConcurrencyLimit` + `@EnableResilientMethods` |
| Fallback | `@Recover` |
| API version | `@GetMapping(version = "2")` |
| List-returning repo | `ListCrudRepository` |
| Schema | Flyway + `ddl-auto: validate` |
| Type-safe config | `@ConfigurationProperties` record |
| Observability | actuator + opentelemetry starter |
