# Testcontainers

Reach for this for integration tests that hit real infrastructure - no mocked persistence.
If you mock the DB, the test is lying.

## Base setup (`@SpringBootTest` + `@Testcontainers`)

```java
@SpringBootTest(webEnvironment = RANDOM_PORT)
@Testcontainers
class OrderServiceIntegrationTest {

    @Container
    @ServiceConnection
    static PostgreSQLContainer postgres =
        new PostgreSQLContainer(DockerImageName.parse("postgres:17-alpine"));

    @Autowired OrderRepository orderRepository;

    @Test
    void should_persist_order_when_saved() {
        var saved = orderRepository.save(new Order("cust-1", TEN));
        assertThat(orderRepository.findById(saved.getId())).isPresent();
    }
}
```

- `@Testcontainers` = the JUnit 5 extension that starts/stops the `@Container`.
- `@ServiceConnection` = Boot auto-wires the datasource from the container. No properties file.
- **Static** container = one instance shared across all tests in the class (fast).

## `@ServiceConnection` (Boot 4 - no manual properties)

```java
// Before: manual wiring with @DynamicPropertySource (avoid now)
// Now: just annotate the container
@Container @ServiceConnection
static PostgreSQLContainer postgres = new PostgreSQLContainer(DockerImageName.parse("postgres:17-alpine"));
```

Boot detects the container type and configures `spring.datasource.*` (or Kafka/Redis/etc.) for you. Works for the common containers out of the box.

## Testcontainers 2.x imports (module packages, not `containers.*`)

```java
// Correct (TC 2.x) - each module has its own package
import org.testcontainers.postgresql.PostgreSQLContainer;
import org.testcontainers.rabbitmq.RabbitMQContainer;
import org.testcontainers.kafka.KafkaContainer;

// Deprecated (TC 1.x)
// import org.testcontainers.containers.PostgreSQLContainer;   // deprecated
```

New classes aren't self-generic (no `<?>`). Construct with `DockerImageName.parse(...)`, **not** the `String` constructor.

```xml
<dependency><groupId>org.testcontainers</groupId>
  <artifactId>postgresql</artifactId><scope>test</scope></dependency>
```

## Common containers

```java
@Container @ServiceConnection
static PostgreSQLContainer postgres =
    new PostgreSQLContainer(DockerImageName.parse("postgres:17-alpine"));

@Container @ServiceConnection
static RabbitMQContainer rabbit =
    new RabbitMQContainer(DockerImageName.parse("rabbitmq:4-management-alpine"));

@Container @ServiceConnection
static KafkaContainer kafka =
    new KafkaContainer(DockerImageName.parse("apache/kafka:3.9.0"));

@Container @ServiceConnection
static GenericContainer<?> redis =
    new GenericContainer<>(DockerImageName.parse("redis:7-alpine")).withExposedPorts(6379);
```

## Shared container across many test classes

```java
// A base class other IT classes extend - container starts once for the suite
@Testcontainers
public abstract class AbstractIntegrationTest {
    @Container @ServiceConnection
    static PostgreSQLContainer postgres =
        new PostgreSQLContainer(DockerImageName.parse("postgres:17-alpine"));
}

class OrderServiceIntegrationTest extends AbstractIntegrationTest { ... }
```

Static container in a base class + JVM reuse = the container isn't restarted per class.

## `@Sql` for test data (not `data.sql` / `import.sql`)

```java
@Test
@Sql("/test-data/orders.sql")                 // runs before the test
@Sql(scripts = "/test-data/cleanup.sql", executionPhase = AFTER_TEST_METHOD)
void should_return_orders_for_customer() {
    assertThat(orderRepository.findByCustomerId("cust-1")).hasSize(2);
}
```

## Full request-response flow (`TestRestTemplate` / `MockMvc`)

```java
@Autowired TestRestTemplate rest;

@Test
void should_return_201_when_creating_order() {
    var request = new CreateOrderRequest("cust-1", TEN, "a@b.com", List.of());
    var response = rest.postForEntity("/api/orders", request, OrderResponse.class);
    assertThat(response.getStatusCode()).isEqualTo(HttpStatus.CREATED);
    assertThat(response.getBody().id()).isNotNull();
}
```

## Integration vs unit test split

| Gets an integration test | Gets a unit test |
|---|---|
| Anything with a `@Repository` | Pure service logic, no IO |
| Kafka / Redis / S3 / RabbitMQ | Business rules, calculations |
| Full request→response flow | Mapping / validation logic |

Unit test: construct the service with plain constructor args, no Spring context, no `@MockBean` for persistence.

## Test naming - `should_<expected>_when_<condition>`

```java
@Test void should_throw_when_order_not_found() { ... }
@Test void should_apply_discount_when_customer_is_premium() { ... }
```

One assertion concept per test. Class naming: `*IntegrationTest` (integration), `*Test` (unit).

## Gotchas / things I always forget

- Import from the **module package** (`org.testcontainers.postgresql.*`), not the deprecated `org.testcontainers.containers.*`.
- Construct with `DockerImageName.parse("...")` - the `String` constructor is gone in TC 2.x.
- `@ServiceConnection` replaces `@DynamicPropertySource` for supported containers - don't do both.
- Make the container **static**, or it restarts for every test method (slow) and loses class-level sharing.
- Never mock the persistence layer (`@MockBean` on a repository) - the test stops verifying anything real.
- `@Sql` runs per method; use `AFTER_TEST_METHOD` cleanup or a fresh transaction rollback to keep tests isolated.
- Pin image tags (`postgres:17-alpine`), never `:latest` - reproducible tests.
- Ryuk (the TC reaper) needs Docker socket access; in CI make sure the runner exposes it.

## Quick reference

| Rule | Value |
|---|---|
| Extension | `@Testcontainers` |
| Auto-wire connection | `@ServiceConnection` |
| Container field | `static` |
| Construct | `new XContainer(DockerImageName.parse("..."))` |
| Import from | `org.testcontainers.<module>` |
| Test data | `@Sql` |
| Class naming (IT) | `*IntegrationTest` |
| Class naming (unit) | `*Test` |
| Method naming | `should_<expected>_when_<condition>` |
