# Java Spring Boot Microservice Coding Standard

Status: mandatory for every person and every AI agent that writes code for our Spring Boot microservices.
Baseline: Java 25, Spring Boot 4.1.x (4.1.2 when this file was written), imperative Spring MVC, PostgreSQL.
This file is the only coding standard. A service may record a temporary exception in its own README under "Standard exceptions", with the reason and the review date. Agents do not invent exceptions.

Copy this file to the root of each microservice, and copy `.cursor/rules/java-spring-boot-standard.mdc` with it. Agents read this file before generating or modifying code.

## 1. How to use this file

Humans review against the checklist in section 23.
Agents follow this order:

1. Section 2, then the Boot 4 trap table in section 3.
2. Section 5, and place every new file before writing it. Name it with section 7, and keep the member order from section 5. Section 20 is the only layout.
3. Section 8 before introducing a class. Copy an accepted snippet. For any HTTP endpoint, read section 11 before choosing the path, method, status, or headers. Read section 17 before authentication, authorization, secrets, or a deploy file, and section 18 before adding a log line. Then read the layer section for the code you are about to write (sections 10–15).
4. Section 19, and write the tests with the production code.
5. Section 23 before you finish.

If two rules seem to conflict, the dependency rule (section 6) and the file-grouping rules (section 5) win. Prefer the smaller design.

Examples use `com.company.orderservice`. Replace `company` with the organization's namespace and `orderservice` with the existing service root. Never introduce a second root package.

## 2. Non-negotiables

- Package by feature, then by layer: `{feature}/api`, `application`, `domain`, `infrastructure`.
- Domain code imports neither Spring, nor Jakarta Persistence, nor Jackson, nor the Servlet API.
- A controller calls an application service. It does not call a repository, a JPA entity, or another infrastructure type.
- An application service orchestrates. Business invariants live in the domain model.
- Infrastructure implements ports. It contains no business rules.
- Constructor injection only. Dependencies are `private final` and are set in the constructor.
- Requests, responses, commands, views, identifiers, and value objects are Java records.
- Aggregates are classes. Create them through a factory. Change them through an intention-revealing method.
- Bean Validation guards HTTP input. The domain model still rejects an invalid state in its constructor or method.
- Every error response is a Problem Details body. A 500 body has a generic detail and no exception text. A failure is never HTTP 200.
- HTTP follows section 11. Paths are plural nouns. Methods keep their HTTP meaning. JSON representations include a `self` link. Every response sets `Cache-Control`. The server stores no client session.
- Schema changes are Flyway migrations. `ddl-auto` is `validate`. `open-in-view` is `false`.
- `@Transactional` sits on the application service. Query methods are `readOnly = true`.
- Persistence tests run against PostgreSQL in Testcontainers. H2 is not a dependency.
- No Lombok, no `@Value`, no `@MockBean`, no `@SpyBean`, no `I` prefix, no `Impl` suffix, no `util` or `helper` or `common` package.
- Mapping between layers is MapStruct. Injected JSON mapping uses the Boot auto-configured Jackson 3 `JsonMapper`.
- A behavior change ships with the test type in section 19. The ArchUnit test stays green.
- Outbound HTTP has a connect timeout and a read timeout.
- The service is a stateless OAuth2 resource server. It validates the bearer JWT itself. The gateway and the mesh are not a substitute. Authenticate every route except the health and info probes.
- Logs are ECS JSON on stdout. They contain the aggregate id and the Micrometer `traceId`. They never contain secrets, tokens, cards, national identifiers, or emails.
- The image runs as a non-root user. A new service ships `catalog-info.yaml` and a default-deny `NetworkPolicy`. CI publishes a CycloneDX SBOM and fails a HIGH or CRITICAL vulnerability that has a fix.
- Design choices match the accepted snippets in section 8. A principle is not a reason to add a service interface, a `Utils` class, a setter-only model, or a base class.
- Names use the four shapes in section 7. Layout is Palantir Java Format: four-space indent, 120-character lines, explicit imports. Google's two-space style and Oracle's 80-column style are not used.
- A new library must be on the approved list in section 21, or the change must say why and a human must accept it.

## 3. Technology baseline

| Choice | Standard |
| --- | --- |
| Language | Java 25 |
| Framework | Spring Boot 4.1.x, latest patch, imported through `spring-boot-starter-parent` |
| Web | Imperative MVC, `spring-boot-starter-webmvc`. One service is either MVC or WebFlux. New services are MVC. |
| API style | JSON over HTTP, OpenAPI 3 via springdoc 3.2.x |
| Bean Validation | `spring-boot-starter-validation` |
| Persistence | Spring Data JPA, Hibernate, PostgreSQL, Flyway |
| Mapping | MapStruct |
| Security | `spring-boot-starter-security` and `spring-boot-starter-security-oauth2-resource-server` |
| Operations | Actuator, Micrometer, ECS JSON logs on stdout, OpenTelemetry traces over OTLP |
| Tests | JUnit Jupiter (the Boot 4.1 BOM manages Jupiter 6), AssertJ, Mockito, Testcontainers, ArchUnit |
| Build | Maven. A service that already uses Gradle keeps Gradle and the same package layout. A new service uses Maven. Do not convert a build tool inside a feature change. |

Parent:

```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>4.1.2</version>
</parent>

<properties>
    <java.version>25</java.version>
</properties>
```

Bump the parent to the latest 4.1 patch in a dedicated change. Do not stay on Boot 3.

### Boot 4 traps

Agents trained on Boot 3 generate code that does not compile here. Use the right column.

| Boot 3 habit | What we write |
| --- | --- |
| `spring-boot-starter-web` | `spring-boot-starter-webmvc` |
| `spring-boot-starter-oauth2-resource-server` | `spring-boot-starter-security-oauth2-resource-server` |
| `spring-boot-starter-oauth2-client` | `spring-boot-starter-security-oauth2-client` |
| `@MockBean`, `@SpyBean` | `@MockitoBean`, `@MockitoSpyBean` in `org.springframework.test.context.bean.override.mockito` |
| `com.fasterxml.jackson.databind.ObjectMapper` | `tools.jackson.databind.json.JsonMapper` |
| `@JsonComponent` | `@JacksonComponent` |
| `org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest` | `org.springframework.boot.webmvc.test.autoconfigure.WebMvcTest` |
| `org.springframework.boot.test.autoconfigure.orm.jpa.DataJpaTest` | `org.springframework.boot.data.jpa.test.autoconfigure.DataJpaTest` |
| `org.springframework.boot.test.autoconfigure.jdbc.AutoConfigureTestDatabase` | `org.springframework.boot.jdbc.test.autoconfigure.AutoConfigureTestDatabase` |
| `org.springframework.boot.web.client.RestTemplateBuilder` | Do not add new `RestTemplate` clients. Use `RestClient`. |
| `spring.jackson.serialization.write-dates-as-timestamps` | `spring.jackson.datatype.datetime.write-dates-as-timestamps` |
| Flyway core alone | `spring-boot-starter-flyway` plus `org.flywaydb:flyway-database-postgresql` |
| `org.testcontainers:postgresql` | `org.testcontainers:testcontainers-postgresql` (Testcontainers 2, version from the Boot BOM) |
| Field `@Autowired` | Constructor injection |
| `javax.*` | `jakarta.*` |

Jackson annotations (`@JsonProperty`, `@JsonIgnore`) stay in `com.fasterxml.jackson.annotation`.

## 4. Repository layout

One deployable service is one Maven module. Do not split the service into Maven modules. ArchUnit enforces the boundaries.

```text
order-service/
├── pom.xml
├── CODING_STANDARD.md
├── README.md
├── catalog-info.yaml
├── Dockerfile
├── .editorconfig
├── deploy/
│   └── network-policy.yaml
├── src/main/java/com/company/orderservice/
│   ├── OrderServiceApplication.java
│   ├── config/
│   ├── shared/
│   │   ├── error/
│   │   └── web/
│   └── order/
│       ├── package-info.java
│       ├── api/
│       ├── application/
│       ├── domain/
│       └── infrastructure/
│           ├── persistence/
│           ├── client/
│           └── messaging/
├── src/main/resources/
│   ├── application.yml
│   ├── application-local.yml.example
│   ├── application-test.yml
│   └── db/migration/
├── src/test/java/com/company/orderservice/
│   ├── architecture/
│   └── order/                         # mirrors main
└── src/test/resources/
```

Names:

- Maven `artifactId` is kebab-case and ends in `-service`: `order-service`.
- The root package is `com.{org}.{artifact}` with the hyphen and the `-service` suffix removed, then the word `service` appended when it is needed to keep the feature name free: artifact `order-service` → `com.company.orderservice`.
- The application class is `{Service}Application` and lives in the root package: `OrderServiceApplication`. Component scan starts there. Do not move it into a subpackage, and do not add a second `@SpringBootApplication`.
- `src/test/java` mirrors `src/main/java`.

`.editorconfig`:

```ini
root = true

[*]
charset = utf-8
end_of_line = lf
insert_final_newline = true
trim_trailing_whitespace = true

[*.java]
indent_style = space
indent_size = 4
```

`indent_size = 4` matches Palantir. Leave it at 4. `.gitignore` includes `application-local.yml`, `.env`, and IDE metadata. Secrets never go in git.

## 5. Package layout and file grouping

A feature is one business capability, named with a singular noun: `order`, `payment`, `shipment`. The feature package owns everything required to change that capability. Technical layering happens inside the feature, not at the service root.

```text
order/
├── package-info.java          # one sentence: what this capability is responsible for
├── api/                       # HTTP in and HTTP out of this feature
├── application/               # use cases, commands, views, outbound ports that are not repositories
├── domain/                    # aggregates, value objects, domain exceptions, repository ports
└── infrastructure/
    ├── persistence/           # JPA entities, Spring Data, persistence adapter, persistence mapper
    ├── client/                # outbound HTTP adapters
    └── messaging/             # consumers, publishers, outbox relay
```

Feature-specific `@Configuration` and `@ConfigurationProperties` live in `{feature}/config` when they exist. Application-wide beans (security filter chain, `Clock`, problem-detail handler) live in the root `config` or in `shared`.

### Grouping algorithm

1. Name the capability with one singular noun. That is the feature package. Reuse it when it already exists.
2. HTTP types and the controller go in `{feature}/api`.
3. Use-case orchestration, commands, and read models go in `{feature}/application`.
4. A rule that must remain true for the business object goes in `{feature}/domain`.
5. JDBC, JPA, HTTP clients, and brokers go in `{feature}/infrastructure`. The interface they implement stays in `domain` (a repository) or `application` (every other port).
6. A type moves to `shared/{bucket}` only when it is technical and a second feature already needs it, or when it is on the starter list below.
7. One primary type per file. A record used by only one parent stays nested in that parent.
8. Stop. Do not create a package for a pattern name, a layer name at the root, or a hypothetical future feature.

### Order inside a source file

The algorithm above chooses the file. This list is the order inside it. Palantir (section 20) formats whitespace and sorts imports. It does not reorder members, so the author writes this order.

1. A license header only when this service already has one. Do not invent a copyright banner.
2. The `package` statement.
3. Imports, one type per line. The formatter's order is the standard: static imports, one blank line, then every other import in ASCII order (`com` before `java`). No star import (`java.util.*`) and no `import module`.
4. Exactly one top-level type. A record, exception, or builder used only by that type stays nested in it.

Class members, top to bottom:

1. `static final` constants, on the type that uses them.
2. Instance fields. Collaborators are `private final` and are not assigned again after the constructor.
3. Constructors.
4. The type's behavior. Each public method is followed by the private helpers that exist only for it.
5. Nested types.

Group methods by the job they do. A file-wide sort of every `public` method, then every `private` method, separates a use case from the helper that only it calls.

| Type | Method order |
| --- | --- |
| Controller | `POST` collection, `GET` collection, `GET` item, `PUT`, `PATCH`, action `POST`s, `DELETE` |
| Aggregate | static factories, then mutators, then queries |
| Application service | constructor, the one use-case method, then its private helpers |
| Test | the success path, then each rejection |

Accepted. One top-level type, imports in ASCII order, then fields and the constructor:

```java
package com.company.orderservice.order.application;

import com.company.orderservice.order.domain.OrderRepository;
import java.util.Objects;

public class CancelOrderService {
    private final OrderRepository orders;

    public CancelOrderService(OrderRepository orders) {
        this.orders = Objects.requireNonNull(orders);
    }
}
```

The use-case method follows the constructor, and a private helper follows the method that uses it. The `cancel` body itself is the one in section 10.

Rejected. A star import, a static import below the other imports, two top-level types, and a constant holder:

```java
import java.util.*;

import static java.util.Objects.requireNonNull;

class CancelOrderService {}

class OrderConstants {
    static final int MAX_ATTEMPTS = 3;
}
```

### Where a new file goes

| You are adding | Package and name |
| --- | --- |
| A new business capability | `{feature}/` plus `package-info.java` |
| An endpoint on an existing resource | `{feature}/api/{Resource}Controller.java` |
| An HTTP body | `{feature}/api/{Action}{Resource}Request.java` or `{Resource}Response.java` |
| A write use case | `{feature}/application/{Action}{Resource}Service.java` and `{Action}{Resource}Command.java` |
| A read use case | `{feature}/application/{Resource}QueryService.java` and `{Resource}View.java` |
| A business invariant or state change | a method on the aggregate in `{feature}/domain` |
| A value that has rules of its own | `{feature}/domain/{Name}.java` as a record |
| Loading and saving an aggregate | `{feature}/domain/{Aggregate}Repository.java` and `{feature}/infrastructure/persistence/{Aggregate}PersistenceAdapter.java` |
| A table or column | `src/main/resources/db/migration/V{n}__{description}.sql` and the JPA entity in the same feature |
| A call to another service | port in `{feature}/application`, adapter in `{feature}/infrastructure/client` |
| A message published with the database change | port in `{feature}/application`, outbox adapter in `{feature}/infrastructure/messaging` |
| A business failure | `{feature}/domain/{Reason}Exception.java`, extending a type from `shared/error` |
| A type used by two features that is still a business concept of one of them | keep it in the owning feature; the other feature calls that feature's application service |
| A technical type already needed by a second feature | `shared/error` or `shared/web` |
| A scheduled job | `{feature}/infrastructure/{Job}Job.java` calling an application service |
| A cross-cutting servlet filter | `shared/web` |
| The service catalog entry | `catalog-info.yaml` at the repository root |
| Who may open a connection to this process | `deploy/network-policy.yaml` |
| The image user and the jar entrypoint | `Dockerfile` at the repository root |

### Starter types that exist in every service

Create these with the service, not on demand inside a feature:

| Type | Location |
| --- | --- |
| `DomainException`, `NotFoundException`, `ConflictException`, `BusinessRuleException`, `BadRequestException`, `PreconditionRequiredException`, `PreconditionFailedException` | `shared/error` |
| `ApiExceptionHandler` | `shared/error` |
| `OffsetPage`, `CursorPage` | `shared/paging` |
| `Link`, `ApiLinks` | `shared/web` |
| `SecurityConfig`, `TimeConfig` | `config` |
| `LayeringArchTest` | `src/test/java/.../architecture` |

### What is never a package

`controller`, `service`, `repository`, `entity`, `model`, `dto`, `util`, `utils`, `helper`, `helpers`, `common`, `base`, `generic`, `pattern`, `patterns`.

`shared` is the only cross-feature package, and it has named buckets. A class named `*Utils`, `*Helper`, `*Constants`, or `*Manager` is the wrong shape: put the behavior on the type that owns the data, or name the type for the job it does (`OrderNumberFormatter` inside the feature that owns order numbers).

### How features talk to each other

- Feature A may call feature B's public application service.
- Feature A does not import feature B's infrastructure, JPA entities, or repository.
- Feature A does not reach into feature B's aggregate and change it.
- A cycle between application services is a design error. Replace the back-edge with a domain event published by the writer and handled by the other feature after commit.

### `shared` admission test

Move a type into `shared` only when all of these are true:

1. It has no business meaning that belongs to one feature.
2. A second feature uses it now, or it is on the starter list.
3. It does not pull Spring, JPA, or HTTP into `shared/error`.

Two copies of a ten-line value object are acceptable. The third use is the moment to extract.

## 6. Dependency rule

Source dependencies point inward.

```text
api  →  application  →  domain
              ▲
infrastructure ┘
```

| Package | May depend on | Must not depend on |
| --- | --- | --- |
| `domain` | Java, `shared.error` | Spring, JPA, Jackson, Servlet, `api`, `application`, `infrastructure`, `shared.web`, `shared.paging` |
| `application` | `domain`, `shared.error`, `shared.paging`, Java | `api`, `infrastructure`, `shared.web`, JPA, Servlet |
| `api` | `application`, `domain` (to map), `shared.web`, `shared.paging`, `shared.error` | `infrastructure`, JPA entities |
| `infrastructure` | the port it implements, JPA, HTTP clients, brokers | `api`, business decisions |
| `shared.error` | Java | Spring, JPA, Jackson |
| `config` | Spring, and the types it wires | feature internals it does not configure |

The domain is persistence-ignorant. The JPA entity is not the aggregate. The controller return type is not the aggregate and not the entity.

Ports:

| Port | Defined in | Implemented in | Name |
| --- | --- | --- | --- |
| Load and save an aggregate | `domain` | `infrastructure/persistence` | `{Aggregate}Repository` |
| Call another system | `application` | `infrastructure/client` | `{System}Client` |
| Publish an integration event | `application` | `infrastructure/messaging` | `{Event}Publisher` |

An application service is a concrete class. It gets an interface only when a second implementation exists in this codebase. Repository and client ports are interfaces because the dependency rule needs them, which is a real reason.

## 7. Naming

Four shapes cover every name we invent. An acronym inside one of those names is a word: `OrderId`, `ApiLinks`, `HttpHeaders`, `JsonBody`, `Url`. `OrderID`, `APILinks`, `HTTPHeaders`, and `JSONBody` are the wrong shape. Types from the JDK and from libraries keep the spelling they were published with (`UUID`, `URI`, `HttpClient`).

| Kind | Shape | Accepted | Rejected |
| --- | --- | --- | --- |
| Class, interface, record, enum | UpperCamelCase noun or noun phrase | `OrderProcessor`, `CreateOrderService` | `orderProcessor`, `Order_Processor` |
| Method, parameter, field, local | lowerCamelCase | `calculateTotal()`, `findUserById()` | `CalculateTotal()`, `find_user_by_id()` |
| Constant (`static final`, enum constant) | UPPER_SNAKE_CASE | `MAX_RETRY_ATTEMPTS`, `DEFAULT_TIMEOUT` | `maxRetryAttempts` |
| Package | lowercase, dot-separated, no underscores | `com.company.orderservice.order` | `com.company.orderService`, `com.company.orderservice.utility` |

A method starts with a verb. A type is a noun. A constant is `static final` on the type that uses the value. There is no `Constants` class, and `utility` is not a package (section 5).

### Packages and files

- Packages are lowercase singular nouns. No underscores, no mixed-case acronyms.
- The file name is the public type name. One primary type per file.
- `package-info.java` on the feature root states the capability in one sentence.

### Types

| Kind | Pattern | Example |
| --- | --- | --- |
| Application class | `{Service}Application` | `OrderServiceApplication` |
| Controller | `{Resource}Controller` | `OrderController` |
| HTTP request | `{Action}{Resource}Request` | `CreateOrderRequest` |
| HTTP response | `{Resource}Response` | `OrderResponse` |
| Write service | `{Action}{Resource}Service` | `CreateOrderService` |
| Query service | `{Resource}QueryService` | `OrderQueryService` |
| Command | `{Action}{Resource}Command` | `CreateOrderCommand` |
| Read model | `{Resource}View` | `OrderView` |
| Aggregate | singular business noun | `Order` |
| Identifier | `{Aggregate}Id` | `OrderId` |
| Value object | business noun | `Money`, `Sku` |
| Domain exception | `{Reason}Exception` | `OrderNotFoundException` |
| Repository port | `{Aggregate}Repository` | `OrderRepository` |
| Persistence adapter | `{Aggregate}PersistenceAdapter` | `OrderPersistenceAdapter` |
| JPA entity | `{Aggregate}JpaEntity` | `OrderJpaEntity` |
| Spring Data interface | `{Aggregate}JpaRepository` | `OrderJpaRepository` |
| API mapper | `{Aggregate}ApiMapper` | `OrderApiMapper` |
| Persistence mapper | `{Aggregate}PersistenceMapper` | `OrderPersistenceMapper` |
| Outbound port | `{System}Client` or `{Event}Publisher` | `PaymentClient` |
| Outbound adapter | `{System}ClientAdapter` | `PaymentClientAdapter` |
| Properties | `{Concern}Properties` | `PaymentClientProperties` |
| Configuration | `{Concern}Config` | `SecurityConfig` |
| Domain event | past tense | `OrderPlaced` |
| Unit test | `{Type}Test` | `OrderTest` |
| Integration test | `{Type}IT` | `OrderPersistenceAdapterIT` |

Forbidden type names: `IOrderRepository`, `OrderRepositoryImpl`, `OrderServiceImpl`, `OrderDTO`, `OrderVO`, `OrderManager`, `OrderHelper`, `OrderUtils`, `BaseController`, `AbstractService`, `CommonService`.

`{Name}Impl` is allowed only when two implementations live side by side, and even then the name should say how it is implemented (`OrderRedisCache`, not `OrderCacheImpl`) when the technology is the reason it exists.

### Methods and fields

- Command methods are verbs: `create`, `cancel`, `place`, `save`.
- A lookup that throws when the row is missing is `get`. A lookup that may be absent is `find` and returns `Optional`. A collection lookup is `list`.
- Booleans read as predicates: `isCancelled`, `hasLines`, `canCancel`. The field does not repeat `flag`.
- Methods and parameters are camelCase. No Hungarian notation.
- Collections are never null. Return `List.of()` or an empty page. Do not return null from a domain or application method.
- `Optional` is a return type for a single absent value. It is not a field, not a parameter, and not a collection wrapper.

### HTTP

- Base path: `/api/v1/{resources}`. Resources are plural kebab-case: `/api/v1/purchase-orders`.
- The path names the resource. It does not contain a verb. Section 11 is the full URL and method profile.
- A state change that is not CRUD is its own resource: `POST /api/v1/orders/{orderId}/cancellations`.
- Path variables match the Java name: `{orderId}`.
- JSON fields are camelCase.
- Offset queries use `?page=0&size=20&sort=createdAt,desc`. Cursor queries use `?limit=20&cursor={opaque}`. An endpoint uses one of those styles, chosen in section 11. Do not invent `page_desc` or `sort=price_desc`.

### Persistence

| Object | Style | Example |
| --- | --- | --- |
| Table | snake_case plural | `orders`, `order_lines` |
| Column | snake_case | `created_at`, `order_id` |
| Primary key | `id` | `id uuid` |
| Foreign key column | `{singular}_id` | `order_id` |
| Index | `idx_{table}_{columns}` | `idx_orders_created_at` |
| Unique constraint | `uk_{table}_{columns}` | `uk_orders_number` |
| Foreign key constraint | `fk_{table}_{column}` | `fk_order_lines_order_id` |
| Enum column | string, the enum constant name | `PLACED` |

### Configuration and operations

- Property prefix: `app.{feature}.{concern}` in kebab-case inside YAML.
- Profile names: `local`, `test`, `prod`.
- Metric names: lowercase dot-separated, without the service name: `orders.placed`.
- Log messages are sentences with placeholders: `log.info("Order placed. orderId={}", orderId)`.
- Migration files: `V{n}__{snake_description}.sql`. `{n}` is a zero-padded integer that only grows: `V001__create_orders.sql`. Do not use timestamps as versions.

### Tests

Method name: `should{Expected}When{Condition}`. Inside the class, the success path comes first and each rejection follows it (section 5).

```text
shouldRejectCancellationWhenOrderIsShipped
shouldReturn404WhenOrderDoesNotExist
```

## 8. Design principles

Principles decide where a behavior lives. They are not permission to add a class. Patterns in section 9 are the blueprints. This section is the judgment.

When a principle and a slogan from a blog disagree with an accepted snippet here, the snippet wins. In particular:

- "Program to an interface" does not mean an interface for every service.
- "Open/closed" and "prefer polymorphism" do not mean a strategy for two branches.
- "DRY" does not mean a `Utils` or `Helper` class.
- "Encapsulation" does not mean a public setter for every field.
- "Inheritance for reuse" does not mean `BaseService` or `AbstractAdapter`.
- A command in an application service may return the aggregate or its id. It does not return an HTTP response, and a query does not save.

Before introducing a type, name the principle you are applying and match its accepted snippet. If the new type is a service interface with one implementation, a `*Utils` class, a base class, or a setter-only aggregate, do not generate it.

### 8.1 Single responsibility

A class changes for one reason. HTTP mapping, a business rule, and SQL are three types in three layers.

Accepted. The controller maps, the service orchestrates, the aggregate protects the invariant, the adapter stores.

```java
public Order create(CreateOrderCommand command) {
    Order order = Order.place(new OrderId(UUID.randomUUID()), command.lines(), clock.instant());
    return orders.save(order);
}
```

Rejected. One class validates the body, computes a total, and calls JPA. It has three reasons to change.

```java
@RestController
class OrderController {
    @PostMapping
    OrderResponse create(CreateOrderRequest request) {
        if (request.lines().isEmpty()) {
            throw new BadRequestException("bad-request", "Order requires a line");
        }
        var total = request.lines().stream().map(Line::price).reduce(Money.ZERO, Money::add);
        return new OrderResponse(jpa.save(new OrderJpaEntity(total)).getId(), total);
    }
}
```

### 8.2 Open/closed

Extend behavior by adding a type when a third variant appears. Until then, a branch stays a branch. Closed means you do not edit the existing carriers to add a new one. It does not mean you may never edit a class that has a bug.

Accepted while there are two carriers:

```java
Money quote(Order order, Carrier carrier) {
    return switch (carrier) {
        case ROAD -> roadRate(order);
        case AIR -> airRate(order);
    };
}
```

Accepted once a third carrier is required. The interface lives in `domain` when the rate is a function of the order, and in `application` when it calls a carrier. Implementations are Spring beans outside `domain`. A new carrier is a new class. The other carriers stay untouched. `UnsupportedCarrierException` extends `BusinessRuleException`.

```java
public interface ShippingQuote {
    boolean supports(Carrier carrier);
    Money quote(Order order);
}

public Money quote(Order order, Carrier carrier) {
    return quotes.stream()
            .filter(quote -> quote.supports(carrier))
            .findFirst()
            .orElseThrow(() -> new UnsupportedCarrierException(carrier))
            .quote(order);
}
```

Rejected. A subclass, or a strategy plus a factory, introduced for the second carrier.

```java
class AirCreateOrderService extends CreateOrderService { }
```

### 8.3 Liskov substitution

Anything that claims to be a port or a parent type must honor that contract, including the failures. Callers substitute it without a special case.

Accepted. Every `OrderRepository` returns an empty `Optional` when the id is absent. `OrderNotFoundException` is a `NotFoundException`, so the handler still answers 404.

```java
public Optional<Order> findById(OrderId id) {
    return jpa.findById(id.value()).map(mapper::toDomain);
}
```

Rejected. One adapter throws a vendor "row not found" exception for the case the port defines as empty. Callers grow an `if (adapter instanceof ...)` check, and the parent contract is a lie.

```java
public Optional<Order> findById(OrderId id) {
    OrderJpaEntity entity = jpa.findById(id.value()).orElseThrow(EntityNotFoundException::new);
    return Optional.of(mapper.toDomain(entity));
}
```

### 8.4 Interface segregation

A client depends on the methods it calls. The aggregate repository stays small. A screen that is not editing the aggregate gets its own query port.

Accepted.

```java
public interface OrderRepository {
    Order save(Order order);
    Optional<Order> findById(OrderId id);
}

public interface OrderViewQuery {
    OffsetPage<OrderView> list(OrderSearch search, PageRequest page);
}
```

Rejected. One fat port. The write service now depends on reporting and export methods it must not call.

```java
public interface OrderRepository {
    Order save(Order order);
    Optional<Order> findById(OrderId id);
    List<Order> findByCustomer(CustomerId id);
    Money revenue(Year year);
    byte[] exportCsv();
    void deleteAll();
}
```

### 8.5 Dependency inversion

High-level code owns the abstraction. Low-level code implements it. Spring injecting the adapter into the service is the inversion.

Accepted. `CreateOrderService` asks for `OrderRepository`. The JPA adapter is the only class that sees `OrderJpaRepository`.

```java
public CreateOrderService(OrderRepository orders, Clock clock) {
    this.orders = orders;
    this.clock = clock;
}
```

Rejected. The use case constructs or declares the persistence adapter. The domain rule now depends on JDBC.

```java
public CreateOrderService(OrderJpaRepository jpa) {
    this.jpa = jpa;
}
```

`ApplicationContext.getBean` and a static `OrderRepositoryHolder` are the same mistake.

### 8.6 Program to an interface

Use an abstract type at the boundary that has more than one implementation, or that exists to keep the dependency rule. Use `List`, not `ArrayList`, in a signature. The application service itself stays a concrete class.

Accepted.

```java
public final class Order {
    public static Order place(OrderId id, List<OrderLine> lines, Instant now) {
        return new Order(id, lines, now, OrderStatus.PLACED);
    }
}
```

Rejected. An interface with one implementation, and a concrete collection type in the API.

```java
public interface CreateOrderService {
    Order create(CreateOrderCommand command);
}

public class CreateOrderServiceImpl implements CreateOrderService { }

void place(ArrayList<OrderLine> lines) { }
```

### 8.7 DRY, after the third copy

A rule lives in one place. Two copies of a ten-line value object may wait. The third copy moves to the type that owns the concept, or to `shared` when the admission test in section 5 passes.

Accepted. "A shipped order cannot be cancelled" is only inside `Order.cancel`. The controller, the mapper, and the tests call that method. They do not restate the rule.

Rejected. A helper that exists because two lines looked similar, and a second copy of the business rule in the controller.

```java
public final class OrderUtils {
    public static boolean canCancel(String status) {
        return "PLACED".equals(status);
    }
}
```

### 8.8 KISS and YAGNI

The smallest design that satisfies the current use case is the one to generate. A repository port and a client port are required by section 6. A cache, a message bus, a generic pipeline, and an interface "for later" are not.

Accepted. `Order.place` checks its invariant and returns. No interceptor chain, no factory registry.

Rejected. `CachingOrderRepositoryDecorator`, `OrderCreatedEvent`, and `CreateOrderService` as an interface, added in the same change as the first `POST /orders` because a later feature might want them.

### 8.9 Composition over inheritance

A collaborator is a field. Inheritance is for a true is-a that stays substitutable (section 8.3), one level deep. The exception hierarchy is that case. Reuse is not.

Accepted.

```java
public class CancelOrderService {
    private final OrderRepository orders;

    public CancelOrderService(OrderRepository orders) {
        this.orders = orders;
    }
}

public final class OrderNotFoundException extends NotFoundException {
    public OrderNotFoundException(OrderId id) {
        super("order-not-found", "Order " + id.value() + " was not found");
    }
}
```

Rejected.

```java
public abstract class AbstractCrudService<T> { }

public class CancelOrderService extends CreateOrderService { }
```

### 8.10 Law of Demeter

A method talks to its fields, its parameters, and the objects it creates. It does not walk a graph to make a decision that belongs on the aggregate.

Accepted. The service tells the aggregate. The rule stays in `Order.cancel`, the method in section 10.

```java
public Order cancel(CancelOrderCommand command) {
    Order order = orders.findById(command.orderId())
            .orElseThrow(() -> new OrderNotFoundException(command.orderId()));
    order.cancel(command.reason());
    return orders.save(order);
}
```

Rejected.

```java
if (!order.customer().shippingAddress().country().allows(order.lines().getFirst().sku())) {
    throw new OrderCancellationRejectedException(order.id());
}
```

If the decision needs data from two aggregates, pass those values into a `CancellationPolicy` in `domain`. Do not reach through both graphs from the service.

### 8.11 Encapsulate what changes

What varies gets a name and a boundary. What is stable stays a straight method. The payment provider varies. The use case that places an order does not.

Accepted. `PaymentClient` is the port. `AdyenPaymentClient` and `StripePaymentClient` are the adapters. `CreateOrderService` calls `payments.authorize(...)`.

Rejected. `CreateOrderService` contains the HTTP details of both providers, and a new provider edits that service.

### 8.12 Cohesion and coupling

A feature package is cohesive: everything in `order` is about orders. Coupling between features is a call to the other feature's application service.

Accepted. `shipment` calls `CancelOrderService` or `OrderQueryService`. It does not import `OrderJpaEntity`.

Rejected. `ShipmentController` injects `OrderJpaRepository`, or one `CommerceService` places orders, ships them, and issues invoices.

### 8.13 Command-query separation

A command changes state. A query returns data and does not change state. They are different application services when both exist.

A command may return the aggregate or its id. The caller needs the id, the version, and the new state for the HTTP representation, and a second query inside the same request is not required. A command does not return `OrderResponse`. A query does not call `save`.

Accepted.

```java
@Transactional
public Order create(CreateOrderCommand command) {
    return orders.save(Order.place(new OrderId(UUID.randomUUID()), command.lines(), clock.instant()));
}

@Transactional(readOnly = true)
public Order get(OrderId id) {
    return orders.findById(id).orElseThrow(() -> new OrderNotFoundException(id));
}
```

Rejected. `getOrCreate`, a query method that updates a "last viewed" column as a side effect, and `create` returning `ResponseEntity<OrderResponse>`.

### 8.14 Fail fast

An invalid value is rejected at the boundary that first sees it. The domain constructor rejects an illegal object. The request record rejects a malformed body. Nothing downstream repairs a null id.

Accepted. `OrderId` and `Order.place` throw before `save` is reached.

Rejected. `Order.place` returns null when `lines` is empty, and the service null-checks three calls later.

### 8.15 The four object-oriented tools

Use each tool for the job below. The textbook reason to inherit, "so we can reuse fields," is the rejected use.

| Tool | Where we use it | Accepted shape | Rejected shape |
| --- | --- | --- | --- |
| Abstraction | Hide SQL, HTTP, and brokers behind a port | `OrderRepository.findById` | A controller that builds a JPQL string |
| Encapsulation | Keep state private and change it through a business method | `order.cancel(reason)` with private fields | Public setters, or a public field `status` |
| Inheritance | A substitutable is-a, one level | `OrderNotFoundException extends NotFoundException` | `CancelOrderService extends CreateOrderService` to reuse a field |
| Polymorphism | Several implementations of one port, once three variants exist | `ShippingQuote` implementations chosen by `supports` | A growing `switch` that is edited for every new carrier after the third, or a subclass per HTTP endpoint |

Encapsulation still allows a getter used by a mapper. The getter returns an immutable value (`List.copyOf`, a record, an enum). It is not a way for the service to reach in and change `status`.

### 8.16 What the agent checks before generating a class

1. Which single reason will make this class change? If there are two, split it along the layers in section 5.
2. Is this a port with a real second implementation, or a dependency-rule boundary? If neither, it is a concrete class.
3. Does a behavior already exist on the aggregate? Call it. Do not copy the `if` into the controller.
4. Are you adding this type because the current use case needs it? If the reason is "later," stop.
5. Does the new type's name end in `Utils`, `Helper`, `Manager`, `Impl`, or `Base`? Rename it to the job, or delete it and move the method onto the type that owns the data.

## 9. Design patterns

Use a pattern when the problem it solves is present. Name the type for the business or the technology, not for the pattern (`PricingPolicy`, not `StrategyPattern`).

| Pattern | Use it when | Shape in this codebase |
| --- | --- | --- |
| Ports and adapters | At every feature boundary. This is the default structure, not an optional extra. | Sections 5 and 6 |
| Repository | An aggregate is loaded and saved | `OrderRepository` in `domain`, adapter in `infrastructure/persistence` |
| Adapter | The service calls a database, a broker, or another HTTP API | `PaymentClientAdapter` |
| Factory method | Construction has invariants or a meaning | `Order.place(...)`. A separate factory class only when construction needs collaborators. |
| Policy | A business rule does not naturally live on one entity | `CancellationPolicy` in `domain`, called by the aggregate or the application service |
| Strategy | Three or more interchangeable algorithms, chosen at runtime | A small interface in the feature that owns the variation, one type per algorithm |
| Domain event | Another feature, or another service, must react, and a direct call would couple them or cycle | Record in `domain`, published by the application service |
| Transactional outbox | A message must commit atomically with the database write | Table and relay in `infrastructure/messaging` |
| Anti-corruption | An external payload must not become our domain language | Mapping inside the client adapter. Our ports speak our types. |
| Specification | A query predicate is reused by several use cases and has business meaning | A domain type that the persistence adapter translates. One repository method is enough for a single query. |
| Builder | A value has many optional parts and a compact constructor has become unreadable | A builder on that type. Four fields or fewer stay a record constructor. |

Patterns we do not use:

| Pattern | Why it is out |
| --- | --- |
| Service locator, `ApplicationContext.getBean` in feature code | Hides dependencies |
| Hand-written singleton | The container creates beans |
| Active Record (the aggregate is the `@Entity`) | Breaks the dependency rule |
| Generic CRUD base controller or base service | Erases the use case and invites unscoped `deleteAll` |
| `IName` / `NameImpl` pairs | Noise. The port has the plain name. |
| Layer-first root packages | Spread one change across the tree and defeat the grouping rules |
| Domain events for a private call inside one use case | A method call is clearer |
| Repository exposed as `JpaRepository` outside infrastructure | Leaks `deleteAll`, flush, and vendor types |

CQRS, in the form we use, means separate command and query services inside the same feature and the same database. It does not mean event sourcing, a second database, or a framework. Add those only through a change to this standard.

## 10. Layer rules

### API

The controller translates HTTP into a command or a query and translates the result into a response. It contains no business rule and no conditional that encodes a business rule.

```java
@RestController
@RequestMapping("/api/v1/orders")
public class OrderController {

    private final CreateOrderService createOrderService;
    private final OrderQueryService orderQueryService;
    private final OrderApiMapper mapper;

    public OrderController(CreateOrderService createOrderService,
                           OrderQueryService orderQueryService,
                           OrderApiMapper mapper) {
        this.createOrderService = createOrderService;
        this.orderQueryService = orderQueryService;
        this.mapper = mapper;
    }

    @PostMapping
    public ResponseEntity<OrderResponse> create(@Valid @RequestBody CreateOrderRequest request) {
        Order order = createOrderService.create(mapper.toCommand(request));
        OrderResponse body = mapper.toResponse(order);
        URI location = URI.create("/api/v1/orders/" + body.id());
        return ResponseEntity.created(location).body(body);
    }

    @GetMapping("/{orderId}")
    public OrderResponse get(@PathVariable UUID orderId) {
        return mapper.toResponse(orderQueryService.get(new OrderId(orderId)));
    }
}
```

- `@RestController` and `@RequestMapping` carry the resource path. Method mappings add only the suffix.
- `@Valid` on every `@RequestBody` and on every query object that has constraints.
- Return `ResponseEntity` when the status or a header varies, including `201 Created` with `Location`. Return the body directly for a plain `200`.
- Do not catch domain exceptions in the controller. The shared handler maps them.
- Do not return a domain object, a JPA entity, `Optional`, or a Spring `Page`.
- Do not inject `OrderRepository`, `OrderJpaRepository`, or an `EntityManager`.
- The snippet below shows the call flow. Production controllers also set the links, `Location`, `ETag`, and `Cache-Control` required by section 11. Do not ship the relative `URI.create` string; build it with `ApiLinks`.

Request records carry Jakarta Validation constraints. Nested parts that belong only to that request stay nested.

```java
public record CreateOrderRequest(
        @NotEmpty List<@Valid LineRequest> lines
) {
    public record LineRequest(
            @NotBlank @Size(max = 64) String sku,
            @Positive int quantity
    ) {
    }
}
```

### Application

One write method, one transaction, one class when the class would otherwise mix concerns. The service:

1. Loads aggregates through repository ports.
2. Calls domain methods.
3. Saves aggregates.
4. Publishes domain events after the aggregate returns them, inside the same transaction only as an outbox write or as an in-memory event published before commit and handled after commit.
5. Returns the aggregate or a view. It does not return a response record and it does not accept a request record.

The command is its own file. It carries domain types, never a request record.

```java
public record CreateOrderCommand(List<OrderLine> lines) {
}
```

```java
@Service
public class CreateOrderService {

    private final OrderRepository orders;
    private final Clock clock;

    public CreateOrderService(OrderRepository orders, Clock clock) {
        this.orders = orders;
        this.clock = clock;
    }

    @Transactional
    public Order create(CreateOrderCommand command) {
        Order order = Order.place(new OrderId(UUID.randomUUID()), command.lines(), clock.instant());
        return orders.save(order);
    }
}
```

```java
@Service
@Transactional(readOnly = true)
public class OrderQueryService {

    private final OrderRepository orders;

    public OrderQueryService(OrderRepository orders) {
        this.orders = orders;
    }

    public Order get(OrderId id) {
        return orders.findById(id).orElseThrow(() -> new OrderNotFoundException(id));
    }
}
```

A query that would hydrate an aggregate only to flatten it for a screen uses a query port in `application` (`OrderViewQuery`) implemented in infrastructure. The port returns `OrderView`, `OffsetPage<OrderView>`, or `CursorPage<OrderView>`. It does not return a JPA entity, and it does not build hyperlinks. The API mapper adds links.

`@Transactional` is on the application service, on a public method or on the class for a query service. It is not on the controller, not on the domain, and not on the Spring Data interface. The service class is public and not `final`: Boot applies the transaction with a class proxy, and a `final` class or a non-public method silently runs with no transaction.

Self-invocation does not start a new transaction. `this.place(...)` from another method of the same class skips the proxy. If the second method needs its own transaction, move it to another bean and inject that bean. `Propagation.REQUIRES_NEW` is that case, and only that case. Two writes that must commit or roll back together stay in one public method with the default `REQUIRED`.

Do not catch a domain exception and return null. The rollback happens only when the exception leaves the service. Unchecked exceptions roll back. Business failures are unchecked. Do not declare checked exceptions on a service method.

A query service is `@Transactional(readOnly = true)` on the class. It has no method that calls `save`. A command service does not use `readOnly`.

Accepted. One public method, one transaction, absence handled before the next step.

```java
@Transactional
public Order cancel(CancelOrderCommand command) {
    Order order = orders.findById(command.orderId())
            .orElseThrow(() -> new OrderNotFoundException(command.orderId()));
    order.cancel(command.reason());
    return orders.save(order);
}
```

Rejected. Field injection, a Lombok constructor, a JPA type in the signature, and a transaction annotation the proxy will ignore.

```java
@Service
@RequiredArgsConstructor
public final class OrderProcessingService {
    @Autowired
    private OrderJpaRepository orderRepository;

    public OrderResponseDTO processOrder(Long orderId) {
        OrderJpaEntity entity = orderRepository.findById(orderId).orElse(null);
        this.audit(entity);
        return new OrderResponseDTO(entity.getId());
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    void audit(OrderJpaEntity entity) {
    }
}
```

What that snippet gets wrong, and the replacement:

| Blog habit | What we generate |
| --- | --- |
| `@RequiredArgsConstructor` | The constructor written in the class. Lombok stays banned. |
| A service interface plus `OrderServiceImpl` for one implementation | The concrete `CancelOrderService`. An interface appears only for a port, or for a strategy that already has a second implementation. |
| The service accepts `CreateOrderRequest` or returns `OrderResponse` | It accepts a command and returns the aggregate or a view. The controller maps HTTP. |
| The service returns `OrderJpaEntity` | The entity never leaves `infrastructure`. `open-in-view` is false, so a lazy load in the controller would throw, not hide. |
| One fat `OrderService` for every operation | One use case class. The invariant stays on the aggregate. "Logic in the service" means orchestration, not a second copy of the rule. |
| `@Transactional` on a private, package-private, or `final` method, or on `this.audit()` | A public method on a non-final bean. A different boundary is a different bean. |
| `orElse(null)` and a later null check | `orElseThrow` with the domain exception before any further work. |
| `SQLException` or `DataAccessException` thrown to the controller | The adapter translates a constraint failure to `ConflictException`. The service throws domain exceptions. `ApiExceptionHandler` maps them. |

The service is stateless. No field is assigned after the constructor. A request-scoped value is a parameter or a local variable, never a field on the `@Service`.

Do not call another service over HTTP while holding a database transaction. Persist the local state, commit, then call. If the remote side must be consistent with the local commit, write an outbox row in the same transaction and let the relay call.

### Domain

The aggregate is a class. Fields are private. The constructor is private. Factories are the public way in. Mutators are business verbs and check the current state.

```java
public final class Order {

    private final OrderId id;
    private final List<OrderLine> lines;
    private final Instant placedAt;
    private OrderStatus status;

    private Order(OrderId id, List<OrderLine> lines, Instant placedAt, OrderStatus status) {
        this.id = Objects.requireNonNull(id, "id");
        if (lines == null || lines.isEmpty()) {
            throw new IllegalArgumentException("Order requires at least one line");
        }
        this.lines = List.copyOf(lines);
        this.placedAt = Objects.requireNonNull(placedAt, "placedAt");
        this.status = Objects.requireNonNull(status, "status");
    }

    public OrderId id() {
        return id;
    }

    public static Order place(OrderId id, List<OrderLine> lines, Instant now) {
        return new Order(id, lines, now, OrderStatus.PLACED);
    }

    public void cancel(String reason) {
        if (reason == null || reason.isBlank()) {
            throw new IllegalArgumentException("Cancellation requires a reason");
        }
        if (status == OrderStatus.CANCELLED) {
            return;
        }
        if (status != OrderStatus.PLACED) {
            throw new OrderCancellationRejectedException(id);
        }
        this.status = OrderStatus.CANCELLED;
    }
}
```

- A rehydrate factory (`Order.restore(...)`) exists for the persistence adapter. It does not re-run creation rules that were true at creation time and are not invariants of every stored row. It still rejects corrupt data.
- Collections exposed from the aggregate are unmodifiable copies.
- The domain calls `Instant.now()`, `UUID.randomUUID()`, or `new RestClient()` in no method. Time and ids arrive as arguments. I/O happens outside.
- A thin CRUD feature with no invariant stays a small domain object. Do not invent a rich model that only forwards setters. The moment a rule appears ("cannot cancel a shipped order"), the rule moves into the aggregate.
- Domain services are rare. Prefer a method on the aggregate. Add a `{Capability}Policy` in `domain` when the rule needs more than one aggregate or a calculation the entity should not own.
- Domain exceptions extend `NotFoundException`, `ConflictException`, or `BusinessRuleException`. Their messages are safe to return to an API client. They carry the id. They do not carry an HTTP status.

Value objects and ids are records with a compact constructor.

```java
public record OrderId(UUID value) {
    public OrderId {
        Objects.requireNonNull(value, "value");
    }
}
```

Enums represent closed sets. Persist them by name. Do not use ordinals.

### Infrastructure

The adapter translates between the port and the technology. It catches vendor exceptions and throws a port-level or domain exception that the caller already expects (`ConflictException` for a unique-key violation the use case handles). It does not decide business policy.

```java
@Component
class OrderPersistenceAdapter implements OrderRepository {

    private final OrderJpaRepository jpa;
    private final OrderPersistenceMapper mapper;

    OrderPersistenceAdapter(OrderJpaRepository jpa, OrderPersistenceMapper mapper) {
        this.jpa = jpa;
        this.mapper = mapper;
    }

    @Override
    public Order save(Order order) {
        return mapper.toDomain(jpa.save(mapper.toEntity(order)));
    }

    @Override
    public Optional<Order> findById(OrderId id) {
        return jpa.findById(id.value()).map(mapper::toDomain);
    }
}
```

Spring Data interfaces extend `Repository`, not `JpaRepository` and not `CrudRepository`. Declare the methods you actually call. This keeps `deleteAll` and bulk updates out of the feature.

```java
interface OrderJpaRepository extends Repository<OrderJpaEntity, UUID> {

    Optional<OrderJpaEntity> findById(UUID id);

    OrderJpaEntity save(OrderJpaEntity entity);
}
```

JPA entities are package-private, live next to the adapter, and are a storage model. `FetchType.LAZY` is the default. `FetchType.EAGER` is not used. `@Version` is a `long` on any aggregate that can be updated. `@Enumerated(EnumType.STRING)`. No `CascadeType.ALL` unless the children are part of the same aggregate and are saved only through the root.

Equality of a JPA entity is by id, and only for a non-null id. Do not put entities in a `HashSet` before they have ids.

### Mapping

MapStruct does the mapping. `componentModel = "spring"`. `unmappedTargetPolicy = ReportingPolicy.ERROR`.

- `OrderApiMapper` in `api` maps request ↔ command and domain ↔ response.
- `OrderPersistenceMapper` in `infrastructure/persistence` maps domain ↔ JPA entity.
- The two mappers do not call each other.
- A default method on the mapper may unwrap an `OrderId`. Do not hand-write a second mapping style beside MapStruct.
- MapStruct names its generated class `{Mapper}Impl`. That generated name is the one exception to the no-`Impl` rule. Source types still use the names in section 7.

### Visibility

- Public: application services, domain types, ports, controllers, properties, shared error types, the application class.
- Package-private: JPA entities, Spring Data interfaces, adapters, MapStruct mappers.
- Private: aggregate constructors, fields, and methods that are not the model's behavior.

### Logging inside layers

Log a business rejection once, at WARN, in the application service, without a stack trace.
Log an unexpected exception once, at ERROR, in `ApiExceptionHandler`, with the stack trace.
Log an outbound failure once, at ERROR, in the adapter, then throw a client-safe exception.
Do not log the same failure in the controller and the service and the handler.

## 11. REST API

Every controller follows this section. It is the team's REST profile: Fielding's architectural constraints, applied with the method and status rules in RFC 9110, caching in RFC 9111, Problem Details in RFC 9457, partial update in RFC 5789 and RFC 7396, and `Sunset` in RFC 8594. A blog summary does not override this section.

Agents do not copy these shortcuts:

| Shortcut | What we write instead |
| --- | --- |
| `POST /createNewUser`, `GET /getUserDetails?id=5` | `POST /api/v1/users`, `GET /api/v1/users/{userId}` |
| A singular collection, `/product` | Plural collections, `/products` |
| `/users/5/orders/12/items/3/reviews` | At most two identifiers; otherwise a filtered collection |
| `sort=price_desc` or `?page=2&limit=50` on an offset endpoint | `sort=price,desc`. Offset uses `page` and `size`. Cursor uses `limit` and `cursor` |
| HTTP 200 with `{ "success": false }` | The status from section 11.8 and a Problem Details body |
| Spring HATEOAS `EntityModel` | A `links` object built by `ApiLinks` |
| `null` in PATCH means "skip this field" | JSON Merge Patch: omitted means unchanged, `null` clears |
| PUT creates a resource when our ids are server-generated | `POST` creates. `PUT` on a missing id is 404 |
| JavaScript, HTML, or a screen label in the body | A JSON resource representation |

### 11.1 The six constraints

| Constraint | What the agent writes | What the agent does not write |
| --- | --- | --- |
| Uniform interface | One resource shape for the whole service: plural noun URIs, HTTP methods with the meanings below, JSON representations, and links to the next valid actions | A one-off RPC path, a second error format, or a response the client can understand only from a wiki page |
| Client-server | The body is the resource. Clients render it | HTML, view-model labels (`statusLabel`), or a field that exists only to feed one screen |
| Stateless | Each request carries its own `Authorization` and parameters. `SessionCreationPolicy.STATELESS` | `HttpSession`, sticky sessions, or a singleton map of "the current user" |
| Cacheable | Every response sets `Cache-Control`. A GET of a resource also sets `ETag` | A GET that is silent about caching, or `Cache-Control: public` on an authenticated body |
| Layered system | `Location` and links use `app.api.public-base-url`. The client cannot see pod hostnames | A link built from the internal listen address, or a problem body that names an internal service host |
| Code on demand | Not used. This optional constraint is rejected | Executable script, HTML with script, or a body the client is expected to eval |

`server.forward-headers-strategy: framework` is set so a gateway's `Forwarded` headers are visible. The public base URL still wins over those headers when it is configured, which it is in every deployed environment.

### 11.2 URLs and resources

- Path shape: `/api/v1/{collection}` and `/api/v1/{collection}/{id}`. Collections are plural kebab-case nouns.
- The gateway may prefix the service. The service path still starts at `/api`.
- No verb segment: `create`, `get`, `update`, `delete`, `list`, `search`, `cancel`, `process`.
- No trailing slash, no `.json` suffix, no file extension.
- A non-CRUD action is a noun sub-resource: `POST /api/v1/orders/{orderId}/cancellations`. The action name is not the path (`/cancel` is rejected).
- Nesting stops at two identifiers. `/api/v1/orders/{orderId}/lines/{lineId}` is the deepest legal path. A third identifier becomes a query on a collection: `GET /api/v1/reviews?lineId={lineId}`.
- A true singleton under a parent may be singular: `GET /api/v1/customers/{customerId}/profile`. A collection is never singular.
- Identifiers in the path are the resource id, not a search field. Filters go in the query string.
- A collection that does not exist as a route returns 404 from the framework. An empty collection that does exist returns 200 and an empty `items` array, not 404.

### 11.3 Method semantics

Safe means the method does not change server state. Idempotent means repeating a successful request leaves the same server state, even if a later response status differs.

| Method | Use | Safe | Idempotent | Success |
| --- | --- | --- | --- | --- |
| `GET` | Read one resource or one page | Yes | Yes | 200, or 304 when `If-None-Match` hits |
| `POST` | Create a resource, or run an action that is not a pure replace | No | No, unless `Idempotency-Key` makes the create retry-safe | 201 for a create, 200 for an action that returns a body, 202 only when the work is actually queued |
| `PUT` | Replace the whole representation of an existing resource | No | Yes | 200 and the new representation |
| `PATCH` | Partial update with JSON Merge Patch | No | Yes for our media type. Generic PATCH is not | 200 and the new representation |
| `DELETE` | Remove the resource from the API | No | Yes | 204 |

Narrowings agents must keep:

- `POST` creates. Our ids are generated by the service, so `PUT` does not upsert. `PUT` against an unknown id is 404.
- `PUT` sends every editable field. A missing field on `PUT` clears it or fails validation. It is not a partial update.
- `PATCH` consumes `application/merge-patch+json`. Omitted members stay as they are. `null` clears a nullable member. An array in the patch replaces the whole array. Applying the same patch twice yields the same state.
- A state change with its own rules (`cancel`, `submit`, `refund`) is a `POST` to an action collection, not a `PATCH` that hides the rule in a status string.
- `DELETE` means a later `GET` returns 404. History may remain in the database as a terminal state, but the public resource is gone. A second `DELETE` returns 404. The server state does not change again.
- Do not use `GET` with a body, and do not use `GET` to change state.
- 202 is only for work that is accepted onto a queue and not finished in the request. The response then includes a link to the status resource. Do not use 202 as a vague success.

`POST`, `PUT`, and `PATCH` require a body and `Content-Type`. `GET` and `DELETE` reject a JSON body with 400 if one is sent.

### 11.4 Representations

- Success bodies are JSON objects, `Content-Type: application/json`.
- `POST`, `PUT` consume `application/json`. `PATCH` consumes `application/merge-patch+json`.
- A client that only accepts XML or another type receives 406. A write with the wrong `Content-Type` receives 415.
- Unknown JSON members are ignored so a client can send a newer field. Required members are Bean Validation constraints.
- Dates and times are ISO-8601 strings. Instants are UTC. Epoch numbers are not used.
- Enum values are stable uppercase names. Clients must tolerate an unknown enum value. Adding a value is compatible only because of that rule; document it on the enum in OpenAPI.
- The body does not expose table names, foreign-key column names, JPA annotations, or SQL. Optimistic concurrency is the `ETag`, not a `version` field the client edits by hand.
- Do not return `null` for an empty collection. Return `[]`.

### 11.5 Hypermedia

Every resource representation includes `links`. `self` is always present and is a `GET` of that resource. Other relations appear only when that action is valid for the current state. The domain method decides, for example `order.canCancel()`. The mapper does not reimplement the rule.

```json
{
  "id": "6f1b0c3e-9a2d-4f4a-8e1a-0d0c2b6a1001",
  "status": "PLACED",
  "links": {
    "self": { "href": "https://api.company.com/api/v1/orders/6f1b0c3e-9a2d-4f4a-8e1a-0d0c2b6a1001", "method": "GET" },
    "cancel": { "href": "https://api.company.com/api/v1/orders/6f1b0c3e-9a2d-4f4a-8e1a-0d0c2b6a1001/cancellations", "method": "POST" }
  }
}
```

```java
public record Link(String href, String method) {
}
```

`ApiLinks` in `shared/web` builds absolute URIs from `app.api.public-base-url`. Relation names are the map keys (`self`, `cancel`, `next`). Do not add Spring HATEOAS.

A collection page includes `self` and, when another page exists, `next`. Offset pages also include `prev` when `page > 0`.

### 11.6 Pagination, filtering, and sorting

A collection endpoint is paginated. Returning every row is a defect.

Cursor pagination is the default. Use it for any collection that grows, is append-heavy, or whose total count is expensive.

| | Cursor | Offset |
| --- | --- | --- |
| When | Default for new collections | Admin lists that need a page number and a total, and stay small |
| Request | `?limit=20&cursor={opaque}` | `?page=0&size=20&sort=createdAt,desc` |
| `limit` / `size` | Default 20, maximum 100 | Default 20, maximum 100 |
| Response type inside the service | `CursorPage<T>` (`items`, `nextCursor`) | `OffsetPage<T>` (`items`, `page`, `size`, `totalElements`, `totalPages`) |
| End of data | `nextCursor` is null and `next` is absent | `page` is the last page and `next` is absent |

- `page` is zero-based. The first page is `0`, not `1`.
- The cursor is opaque. Clients send it back unchanged. It is not an offset integer and not a raw timestamp the client may edit.
- One endpoint uses one style. Do not accept both `page` and `cursor`.
- Offset queries add a unique tie-break, the id, after the requested sort. `createdAt` alone is not a stable order.
- Changing `sort` starts a new cursor or a new offset sequence. The client does not reuse the old cursor.
- Filters are camelCase field names: `GET /api/v1/orders?status=PLACED&customerId={uuid}`. Repeating one parameter means OR for that field: `?status=PLACED&status=CANCELLED`.
- An unknown filter or sort field returns 400, problem code `unknown-query-parameter`. Do not ignore it. A typo must not silently return an unfiltered collection.
- There is no generic query language (`filter=status:PLACED`, RSQL, or a SQL fragment) unless this standard is changed.

`spring.data.web.pageable.max-page-size` is 100 and `default-page-size` is 20. A cursor endpoint enforces the same maximum in the controller. A `limit` or `size` above the maximum is 400, not a silent clamp.

### 11.7 Caching and conditional requests

| Response | `Cache-Control` | Validators |
| --- | --- | --- |
| Authenticated `GET` | `private, no-cache` | Strong `ETag` |
| `POST`, `PUT`, `PATCH`, `DELETE` | `no-store` | `ETag` of the new state on 200 and 201 |
| Problem Details | `no-store` | None |
| `304` | The same cache header as the GET, no body | The current `ETag` |

- The `ETag` is the aggregate version in quotes, for example `"3"`. `ResponseEntity.eTag` supplies the quotes. Weak ETags are not used.
- After `save`, the returned aggregate exposes `version()`. The JPA `@Version` column is the persistence of that number. The domain type does not import JPA.
- `GET` calls `WebRequest.checkNotModified(etag)` before writing the body. A match returns 304.
- `PUT` and `PATCH` require `If-Match`. A missing header is 428 (`PreconditionRequiredException`). A stale value is 412 (`PreconditionFailedException`). A lost race that still reaches the database maps `ObjectOptimisticLockingFailureException` to 412 as well.
- Authenticated bodies are never `public`. Do not set a long `max-age` on private data.
- `Vary: Authorization` is set on responses that depend on the caller.

### 11.8 Status codes

| Status | When |
| --- | --- |
| 200 | GET, PUT, PATCH, or an action POST that returns a representation |
| 201 | POST created a resource. `Location` is the new `self` URI |
| 202 | Work was queued. The body links to the status resource |
| 204 | DELETE succeeded. No body |
| 304 | GET and `If-None-Match` matches. No body |
| 400 | Malformed JSON, Bean Validation, unknown query parameter, or a body on GET or DELETE |
| 401 | Missing or invalid credentials. Include `WWW-Authenticate: Bearer` |
| 403 | Authenticated, not allowed |
| 404 | The resource is not in the API, including after DELETE |
| 405 | Method not mapped. Include `Allow` |
| 406 | `Accept` does not allow JSON or Problem Details |
| 409 | Unique-key conflict, or the same `Idempotency-Key` reused with a different body |
| 412 | `If-Match` does not match the current version |
| 415 | `Content-Type` is not the type this method consumes |
| 422 | The representation is understood and a business rule rejects it |
| 428 | `PUT` or `PATCH` without `If-Match` |
| 429 | Rate limited. Include `Retry-After` in seconds |
| 500 | Unexpected failure. Generic detail |
| 502 / 504 | A downstream call failed or timed out |
| 503 | This service is unavailable. Include `Retry-After` when the pause is known |

### 11.9 Idempotency-Key

A `POST` that creates a resource and can be retried requires `Idempotency-Key`. The same key and the same body return the original status and body. The same key and a different body return 409. The key is stored in the same transaction as the aggregate. One implementation serves the whole service. Features do not each invent a store.

### 11.10 Versioning

- The version is a path prefix, `/api/v1`, from the first release. Do not version with a query parameter or a custom header.
- A breaking change is a new prefix, `/api/v2`, in package `api/v2`, sharing the same application and domain. A breaking change removes or renames a field, changes a type or a status meaning, changes a URI, or makes an optional field required.
- Adding an optional JSON member, adding an endpoint, and adding an enum value that clients already tolerate are compatible. They stay on v1.
- Shipped v1 behavior is not "fixed" by changing it in place. A defect fix that changes a documented status or field is a new version.
- While v1 is being retired, its responses include `Sunset` (RFC 8594) and `Link: <{successor}>; rel="successor-version"`.
- Do not fork the domain per version.

### 11.11 Documentation

Controllers carry `@Tag`. Each method carries `@Operation(summary = "...")`. springdoc publishes the description. Do not hand-maintain `openapi.yaml`. The dependency is `springdoc-openapi-starter-webmvc-ui`. Swagger UI is enabled only in the `local` profile. Outside `local`, `/v3/api-docs/**` and `/swagger-ui/**` require authentication.

Document the method, success status, problem statuses, required headers (`Authorization`, `Content-Type`, `Idempotency-Key`, `If-Match`), and whether the collection is cursor or offset.

## 12. Validation and errors

Bean Validation checks shape: present, length, range, format. The domain checks business invariants. Do not copy every business rule into an annotation, and do not skip the annotation for obvious shape checks.

Every error body uses `ProblemDetail` (`application/problem+json`).

| Situation | Status | Exception |
| --- | --- | --- |
| Malformed JSON, failed Bean Validation, unknown query parameter, body on GET or DELETE | 400 | framework types, or `BadRequestException` |
| Missing or invalid credentials | 401 | Spring Security, with `WWW-Authenticate: Bearer` |
| Authenticated but not allowed | 403 | Spring Security |
| Missing route or missing aggregate | 404 | `NotFoundException` |
| Method not mapped | 405 | framework type, with `Allow` |
| `Accept` cannot be satisfied | 406 | `HttpMediaTypeNotAcceptableException` |
| Unique-key clash, or idempotency key reused with a different body | 409 | `ConflictException` |
| `If-Match` does not match, including a lost `@Version` update | 412 | `PreconditionFailedException` |
| Wrong `Content-Type` | 415 | `HttpMediaTypeNotSupportedException` |
| Well-formed request rejected by a business rule | 422 | `BusinessRuleException` |
| `PUT` or `PATCH` without `If-Match` | 428 | `PreconditionRequiredException` |
| Rate limited | 429 | handler, with `Retry-After` |
| Remote system returned an error, or the call timed out | 502 or 504 | exception thrown by the client adapter |
| Unexpected failure | 500 | handler, generic detail |

```java
public abstract class DomainException extends RuntimeException {

    private final String problemCode;

    protected DomainException(String problemCode, String message) {
        super(message);
        this.problemCode = problemCode;
    }

    public String problemCode() {
        return problemCode;
    }

    public Map<String, Object> properties() {
        return Map.of();
    }
}
```

`NotFoundException`, `ConflictException`, and `BusinessRuleException` extend `DomainException` and fix nothing about HTTP. Each one exposes a constructor of `(String problemCode, String message)`. The handler chooses the status. A feature exception passes a stable kebab-case code.

`BadRequestException`, `PreconditionRequiredException`, and `PreconditionFailedException` also extend `DomainException`, with default codes `bad-request`, `precondition-required`, and `precondition-failed`. The API layer raises them for protocol failures. Aggregates do not. Pass a more specific code when the failure has one, for example `unknown-query-parameter`. Do not add a subclass per query parameter.

```java
public final class OrderNotFoundException extends NotFoundException {

    public OrderNotFoundException(OrderId id) {
        super("order-not-found", "Order " + id.value() + " was not found");
    }
}
```

The handler:

- Sets `type` to `https://api.company.com/problems/{problemCode}`.
- Sets `title` from the status.
- Sets `detail` from the exception message.
- Sets `instance` from the request path.
- Copies `properties()` onto the problem.
- Adds `traceId` from MDC when present.
- For validation, adds an `errors` array of `{field, message}`.
- For a 500, sets `detail` to `Unexpected error` and logs the exception. The response does not contain the exception message, the SQL, or a class name.
- Sets `Cache-Control: no-store` on every problem response.
- Adds `WWW-Authenticate`, `Allow`, or `Retry-After` when section 11.8 requires that header.
- Maps the framework exceptions in the table above so Spring's default error body is never returned.

One `@RestControllerAdvice` exists: `ApiExceptionHandler`. A feature adds an exception type. It does not add another advice.

## 13. Persistence and transactions

- Database: PostgreSQL. A new service does not start on another engine.
- Ids are UUID, generated by the application service and passed into the factory. Columns are `uuid`. No `@GeneratedValue` on an aggregate id.
- Timestamps are `timestamptz`. Hibernate uses UTC (`spring.jpa.properties.hibernate.jdbc.time_zone: UTC`).
- Money is `numeric(19, 4)` plus `char(3)` when the feature is multi-currency. A single-currency feature stores the amount and keeps the currency in the value object as a named constant.
- Child tables of an aggregate use a foreign key and an index on that foreign key.
- `spring.jpa.hibernate.ddl-auto=validate` and `spring.jpa.open-in-view=false` in every profile.
- Flyway owns the schema. Migrations live in `src/main/resources/db/migration`.
- A merged migration is immutable. Repair a mistake with a new migration.
- One migration, one concern.
- A breaking column change follows expand then contract: add the new column, backfill, switch the code, drop the old column in a later release.
- Optimistic locking is the default for updates. The API expresses it as `ETag` and `If-Match` (section 11.7). A lost update maps `ObjectOptimisticLockingFailureException` to 412. Pessimistic locks require a comment in the adapter naming the contention they prevent.
- The transaction boundary is the application-service method. Default propagation is `REQUIRED`.

## 14. Time, money, and identifiers

- A `Clock` bean of `Clock.systemUTC()` lives in `TimeConfig`. Application services call `clock.instant()` and pass the `Instant` into the domain.
- Tests that care about time pass `Clock.fixed(...)`.
- The domain does not call `LocalDateTime.now()` or `new Date()`.
- `Instant` is a moment. `LocalDate` is a civil date. `LocalDateTime` is not a timestamp and is not stored as one.
- Money is a `Money` record of `BigDecimal` plus `Currency`. Never `double` or `float`. Rounding happens in a named domain method with an explicit `RoundingMode`.
- Identifiers are records. The raw `UUID` appears at the HTTP boundary and in the JPA entity, not as the field type of an aggregate.

## 15. Calls to other services

- Use `RestClient`, built from the injected `RestClient.Builder`, one bean per remote system, in the feature's client package.
- Base URL and both timeouts come from `@ConfigurationProperties`. No URL is hard-coded.
- The adapter maps the remote payload into our types (anti-corruption). A remote DTO does not leak into `application` or `domain`.
- Timeouts fail the call. Retries are allowed for idempotent calls only, with a small fixed number and a bounded delay. A create is not retried unless the idempotency contract is in place.
- Build the client from the injected `RestClient.Builder`. That builder sends the current W3C `traceparent`. `RestClient.builder()` and `RestClient.create()` drop it.
- The remote URL is `https`. TLS certificate checks stay on. An all-trusting client is a defect.
- Map a remote 404 to our `NotFoundException` only when absence is part of our use case. Map a timeout or a 5xx to a dedicated exception that the handler turns into `502` or `504`. Do not wrap it as a 500 with the remote body attached.
- WebClient and declarative HTTP interfaces are for a WebFlux service. An MVC service does not add them.

Messaging, when the feature publishes:

- Write an outbox row in the same transaction as the aggregate.
- A relay in `infrastructure/messaging` publishes after commit and marks the row sent.
- Delivery is at least once. Consumers are idempotent.
- The relay copies the current Micrometer trace onto the published message. The consumer continues that trace. A hand-rolled `X-Correlation-ID` is not a second trace.
- Do not publish to a broker and then commit the database, and do not commit and then publish in the request thread with no outbox.
- Until a feature publishes, do not add an outbox "for later".

In-process reactions use a domain event record. The aggregate collects events. The application service pulls them and publishes with `ApplicationEventPublisher`. The listener is `@TransactionalEventListener(phase = AFTER_COMMIT)` on an application service of the consuming feature.

## 16. Configuration

Configuration is typed, validated, and free of secrets in source. Business code reads a properties record. It does not call `System.getenv`, and it does not use `@Value`.

- `application.yml` is the only committed format. Do not also add `application.properties`.
- The file is production-safe. Profiles are `local`, `test`, and `prod`. The platform sets `prod`. The jar does not set `spring.profiles.active` and does not default itself to `local`.
- `application-local.yml.example` is committed. `application-local.yml` is gitignored. A profile file overrides the keys that differ. It does not copy the whole base file.
- A secret is an environment variable with no default: `${DB_PASSWORD}`. A missing secret fails startup. Do not put a secret in a properties record. A record's `toString` would print it. Boot's datasource password stays on `spring.datasource.password`.
- Non-secret settings for one concern are one `@ConfigurationProperties` record. The prefix is `app.{feature}.{concern}`. `@ConstructorBinding` is not added. A record already has one constructor.
- `@Validated` and `ignoreUnknownFields = false` are set. A typo in a key fails startup. A missing required value fails startup. Durations and URIs use `Duration` and `URI`, not strings that the service parses.
- `@ConfigurationPropertiesScan` is on the application class. Do not register every properties type with `@EnableConfigurationProperties`.
- Feature properties and the `@Bean` methods that use them live in `{feature}/config`. Application-wide beans stay in the root `config`.
- `@Configuration(proxyBeanMethods = false)` is the default. `@Bean` methods take their dependencies as parameters. They do not call each other, and they do not use `@Autowired` or `@Value`.
- Do not log a properties object.

```java
@SpringBootApplication
@ConfigurationPropertiesScan
public class OrderServiceApplication {
    public static void main(String[] args) {
        SpringApplication.run(OrderServiceApplication.class, args);
    }
}
```

```java
@Validated
@ConfigurationProperties(prefix = "app.payment", ignoreUnknownFields = false)
public record PaymentClientProperties(
        @NotNull URI baseUrl,
        @NotNull Duration connectTimeout,
        @NotNull Duration readTimeout
) {
    public PaymentClientProperties {
        if (connectTimeout != null && (connectTimeout.isZero() || connectTimeout.isNegative())) {
            throw new IllegalArgumentException("connectTimeout must be positive");
        }
        if (readTimeout != null && (readTimeout.isZero() || readTimeout.isNegative())) {
            throw new IllegalArgumentException("readTimeout must be positive");
        }
    }
}
```

```java
@Configuration(proxyBeanMethods = false)
class PaymentClientConfig {

    @Bean
    RestClient paymentRestClient(RestClient.Builder builder, PaymentClientProperties properties) {
        JdkClientHttpRequestFactory requestFactory = new JdkClientHttpRequestFactory(
                HttpClient.newBuilder().connectTimeout(properties.connectTimeout()).build());
        requestFactory.setReadTimeout(properties.readTimeout());
        return builder.baseUrl(properties.baseUrl().toString()).requestFactory(requestFactory).build();
    }
}
```

Rejected. A scattered `@Value`, a stringly-typed URL, and a default password.

```java
@Bean
RestClient paymentRestClient(@Value("${app.payment.base-url:http://localhost}") String baseUrl,
                             @Value("${PAYMENT_PASSWORD:secret}") String password) {
    return RestClient.builder().baseUrl(baseUrl).build();
}
```

A value rule on the record is tested by constructing it. A binding rule is tested with `ApplicationContextRunner`. Neither test is a `@SpringBootTest`.

```yaml
spring:
  application:
    name: order-service
  threads:
    virtual:
      enabled: true
  datasource:
    url: jdbc:postgresql://localhost:5432/order_service
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}
  jpa:
    open-in-view: false
    hibernate:
      ddl-auto: validate
    properties:
      hibernate.jdbc.time_zone: UTC
  flyway:
    enabled: true
  jackson:
    datatype:
      datetime:
        write-dates-as-timestamps: false
  data:
    web:
      pageable:
        max-page-size: 100
        default-page-size: 20
  http:
    clients:
      connect-timeout: 2s
      read-timeout: 5s

server:
  port: 8080
  shutdown: graceful
  forward-headers-strategy: framework

app:
  api:
    public-base-url: https://api.company.com
  payment:
    base-url: https://payments.internal
    connect-timeout: 2s
    read-timeout: 5s

management:
  server:
    port: 8081
  endpoints:
    web:
      exposure:
        include: health,info,prometheus
  endpoint:
    health:
      probes:
        enabled: true
      show-details: never
  tracing:
    sampling:
      probability: 0.1

logging:
  structured:
    format:
      console: ecs

springdoc:
  swagger-ui:
    enabled: false
```

`application-local.yml` turns the UI back on with `springdoc.swagger-ui.enabled: true`. The `local` profile may permit `/swagger-ui/**` and `/v3/api-docs/**`. Other profiles do not.

Virtual threads handle request work. Do not create a platform-thread pool for ordinary HTTP. A CPU-bound batch uses a named bounded executor. Do not pool virtual threads.

`build-info` is produced by the Spring Boot Maven plugin so `/actuator/info` shows the version.

## 17. Security and compliance

Zero trust applies to this service. The gateway, the mesh, and the vault are platform controls. This repository still validates the caller, omits secrets from source, and ships the evidence an audit can read. A blog list of products does not add a second gateway, a second mesh, or a second policy engine.

### Edge and the caller

The API gateway terminates external TLS, rate-limits, and routes. It is not a module of this service. Do not add Spring Cloud Gateway or Kong configuration here.

The gateway forwards the caller's bearer JWT. This service validates that JWT on every business request: signature, issuer, audience, and expiry. A request that reached the pod without a gateway hop is still unauthenticated.

Accepted. Stateless token API, method security on, probes open, everything else authenticated.

```java
@Configuration(proxyBeanMethods = false)
@EnableMethodSecurity
class SecurityConfig {

    @Bean
    SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        return http
                .csrf(csrf -> csrf.disable())
                .sessionManagement(sessions -> sessions.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
                .authorizeHttpRequests(auth -> auth
                        .requestMatchers("/actuator/health/**", "/actuator/info").permitAll()
                        .anyRequest().authenticated())
                .oauth2ResourceServer(oauth -> oauth.jwt(Customizer.withDefaults()))
                .build();
    }
}
```

`permitAll` stops at those two matchers. The `local` profile may also permit the springdoc paths from section 16. Actuator listens on `8081`. `env`, `beans`, `heapdump`, and `mappings` stay unexposed. `server.forward-headers-strategy` stays `framework`, and the network policy below is what keeps that port reachable only from the gateway.

Rejected. The service trusts the gateway and stops checking tokens.

```java
.authorizeHttpRequests(auth -> auth.anyRequest().permitAll())
```

### East-west traffic

The cluster mesh encrypts service-to-service traffic with mutual TLS and gives the workload an identity. The service does not embed Istio or Linkerd, and it does not load mesh certificates in Java.

The caller's authority is still the JWT. A mesh identity is not a substitute for it. Outbound URLs use `https`, and certificate checks stay on (section 15).

### Authorization

Two different questions stay in two places.

| Question | Where it is answered |
| --- | --- |
| May this caller invoke the use case? | `@PreAuthorize` on the public application-service method, checking one authority from the JWT (`SCOPE_orders.write`) |
| Is this state change legal for the aggregate? | The aggregate method (section 8 and section 10) |

`@PreAuthorize` runs because the controller calls the Spring bean. A self-call on the same instance skips it, the same way it skips `@Transactional`.

Accepted. This is the section 10 method plus the authority check.

```java
@PreAuthorize("hasAuthority('SCOPE_orders.write')")
@Transactional
public Order cancel(CancelOrderCommand command) {
    Order order = orders.findById(command.orderId())
            .orElseThrow(() -> new OrderNotFoundException(command.orderId()));
    order.cancel(command.reason());
    return orders.save(order);
}
```

A policy engine is not part of a feature change. Open Policy Agent is the only engine, and only after the same authorization decision is already required by a third service. It is then one port in `application` and one adapter in `infrastructure`. Oso is not added. The aggregate method stays even when a policy engine exists.

Rejected. The domain rule is moved into a policy call, and the aggregate is updated with a setter.

```java
if (!policy.allow(jwt, "order.cancel", orderId)) {
    throw new ResponseStatusException(HttpStatus.FORBIDDEN);
}
order.setStatus(Status.CANCELLED);
```

### Secrets

The platform vault injects secrets as environment variables with no default (section 16). The service does not add a HashiCorp Vault client or an AWS Secrets Manager client, does not store a secret in a static field, and does not sign JWTs. Validation uses the issuer's JWKS. Rotation happens in the vault; the process restarts with the new value.

Passwords, tokens, card numbers, national identifiers, and connection strings are not logged, not put in exceptions, and not put in Problem Details. SQL goes through JPA or Spring's parameter binding. Do not concatenate a request value into SQL, JPQL, or a native query. Do not add a parser for XML from external users without saying so in the change.

### Image, network, and catalog

These files exist so a service cannot be deployed as an undocumented root process on a flat network. They are evidence for SOC 2, ISO 27001, PCI-DSS, and GDPR reviews. The service does not claim a certification in code, comments, or the catalog.

The image is a current Eclipse Temurin 25 JRE. Production replaces the tag with a digest. The process user is `10001:10001`. The image contains the jar, not the source, not a secret, and not a package manager invocation. Maven `finalName` is `app`, so the copy has one source file.

```dockerfile
FROM eclipse-temurin:25-jre
WORKDIR /app
RUN groupadd --system --gid 10001 app \
    && useradd --system --uid 10001 --gid 10001 --no-create-home app
COPY target/app.jar app.jar
USER 10001:10001
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "/app/app.jar"]
```

`deploy/network-policy.yaml` is default deny. Ingress `8080` comes only from the gateway namespace. Ingress `8081` comes only from the monitoring namespace. Egress is DNS, PostgreSQL, the named downstreams, the broker when this feature publishes, and the OTLP collector. The database policy, owned with the database, accepts `5432` only from this service. Replace the placeholder label with the platform's namespace label. Do not also add a Cilium policy.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: order-service
spec:
  podSelector:
    matchLabels:
      app: order-service
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              policy.company/name: gateway
      ports:
        - protocol: TCP
          port: 8080
    - from:
        - namespaceSelector:
            matchLabels:
              policy.company/name: monitoring
      ports:
        - protocol: TCP
          port: 8081
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              policy.company/name: data
      ports:
        - protocol: TCP
          port: 5432
    - ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53
    - to:
        - namespaceSelector:
            matchLabels:
              policy.company/name: payments
      ports:
        - protocol: TCP
          port: 443
    - to:
        - namespaceSelector:
            matchLabels:
              policy.company/name: telemetry
      ports:
        - protocol: TCP
          port: 4318
```

`catalog-info.yaml` registers the service before it is reachable. Do not add a Backstage plugin to the Java app.

```yaml
apiVersion: backstage.io/v1alpha1
kind: Component
metadata:
  name: order-service
  description: Takes and cancels orders
spec:
  type: service
  lifecycle: production
  owner: team-orders
  system: commerce
```

### Supply chain

CI produces a CycloneDX SBOM with `org.cyclonedx:cyclonedx-maven-plugin` and scans the image with Trivy. A HIGH or CRITICAL finding that has a fixed version fails the pipeline. The bump is its own change. Do not also add Snyk or OWASP Dependency-Check. Dependencies are still reviewed when added (section 21).

| Other write-up | What we generate |
| --- | --- |
| The gateway already authenticated the user | `SecurityConfig` above. The service validates the JWT |
| Istio or Linkerd resources inside this repo | The platform mesh. This repo ships a `NetworkPolicy` |
| OPA or Oso for the first authorization rule | `@PreAuthorize` for the authority, the aggregate for the business rule |
| A Vault or Secrets Manager SDK | Environment variables injected by the platform vault |
| Snyk, Trivy, and Dependency-Check together | Trivy plus one CycloneDX SBOM |
| A Cilium policy and a `NetworkPolicy` | One `NetworkPolicy` |
| A Backstage plugin in the service | `catalog-info.yaml` |

## 18. Logging and observability

Logs go to stdout and stderr only. The platform collector reads them. There is no log file in the image and no vendor log API in the service.

- SLF4J, one logger per class: `private static final Logger log = LoggerFactory.getLogger(CreateOrderService.class);`
- Parameterized messages. No string concatenation in the call.
- INFO for a state change that an operator would search for. DEBUG for diagnostic detail. WARN for a rejected business action. ERROR for a failed operation a human must notice.
- Include the aggregate id. Include a stable event phrase at the start of the message (`Order placed.`).
- Where each failure is logged is section 10. A business rejection is WARN without a stack. An unexpected exception is ERROR with a stack, once, in `ApiExceptionHandler`. The HTTP body still has no exception text (section 12).

Production logs are ECS JSON via `logging.structured.format.console=ecs`. `service.name` comes from `spring.application.name`. Micrometer Tracing puts `traceId` and `spanId` on the MDC, and the ECS formatter writes those MDC keys into the JSON. The problem body echoes `traceId` when the MDC has it.

The correlation id is that `traceId`. The gateway propagates W3C `traceparent`. The injected `RestClient.Builder` and the message relay continue the same trace (section 15). Do not generate `X-Correlation-ID`, and do not add a filter whose job is to invent one.

Accepted.

```java
log.info("Order placed. orderId={}", order.id().value());
```

Rejected. A concatenated message, a token, a card, an email, and a body dump.

```java
log.error("Payment failed for " + email + " token " + authorization + " card " + cardNumber);
log.info("Request body {}", request);
```

Leave these values out of the message, the MDC, and the exception text: `Authorization`, `Cookie`, `Set-Cookie`, passwords, access tokens, card numbers, national identifiers, emails, and raw request or response bodies. An audit line may include the JWT `sub` as `enduser.id`. It does not include the email. A masking utility that regex-scans log lines is the wrong shape. Do not pass the secret in the first place.

Do not log a properties object (section 16).

| Hand-rolled field | ECS field we already emit |
| --- | --- |
| `timestamp` | `@timestamp` |
| `level` | `log.level` |
| `service_name` | `service.name` |
| `message` | `message` |
| `correlation_id` | `traceId` from the MDC |
| `exception` | the stack on the single ERROR log for an unexpected failure |

Traces use `spring-boot-starter-opentelemetry` and export OTLP. The platform sets `management.opentelemetry.tracing.export.otlp.endpoint`. The jar does not hardcode that URL. Sampling in the jar stays `0.1`. Do not add Zipkin, Jaeger, Sleuth, a vendor SDK, or a Logback OpenTelemetry appender. Logs stay on stdout.

Custom metrics use Micrometer and a lowercase dot name: `orders.placed`, with a `result` tag of `success` or `failure`. Do not put high-cardinality values (raw ids, emails) in tags. Prometheus is scraped from `8081`. Health distinguishes liveness and readiness. Readiness fails when the database is unreachable.

A custom span uses `ObservationRegistry` in the application service. The observation name is lowercase and dotted, the same way as a metric. The domain does not take the registry.

## 19. Testing

Tests follow the production package. They use JUnit Jupiter, AssertJ, and Mockito. They do not use JUnit 4.

| Production change | Test |
| --- | --- |
| Domain rule | `{Type}Test`, plain JUnit, no Spring. Cover the failure and the success. |
| Application branching and port interaction | `{Type}Test` with Mockito. No Spring context. |
| HTTP status, validation, JSON shape, `self` link, `ETag`, and `Cache-Control` | `{Controller}Test` with `@WebMvcTest` and `@MockitoBean` for the application services |
| Repository adapter, queries, and migrations | `{Type}IT` with `@DataJpaTest`, Testcontainers PostgreSQL, `@AutoConfigureTestDatabase(replace = Replace.NONE)`, `@ServiceConnection` |
| Security rule on a route | the `@WebMvcTest`, with security enabled for that assertion |
| Layer dependency | `LayeringArchTest`, extended when a new top-level package appears |

`mvn test` runs unit tests, `@WebMvcTest`, and ArchUnit. `*IT` classes run under Failsafe (`mvn verify`) so a missing Docker daemon does not block the unit suite. A bug fix adds the test that would have failed before the fix.

Rules for the tests themselves:

- One behavior per test method.
- Arrange, act, assert, separated by a blank line. No comments that only say "given".
- AssertJ assertions. No `assertEquals` from JUnit except when AssertJ cannot express the check.
- Do not `@SpringBootTest` a domain rule or a controller mapping. `@SpringBootTest` is for a path that needs the real composition of several layers, and then it is an `*IT`.
- `@MockitoBean` is a field on the test class. It is not placed on a `@Configuration` class. `@MockitoSpyBean` follows the same rule.
- Tests do not depend on each other or on execution order.
- Time-sensitive tests inject `Clock.fixed`.
- Persistence tests do not use H2.
- A service test constructs the service with `new`. It does not boot Spring and it does not use `@InjectMocks`. A new constructor parameter then fails compilation instead of remaining null.

Service test shape:

```java
class CancelOrderServiceTest {

    private final OrderRepository orders = mock(OrderRepository.class);
    private final CancelOrderService service = new CancelOrderService(orders);

    @Test
    void shouldRejectWhenTheOrderIsMissing() {
        OrderId id = new OrderId(UUID.randomUUID());
        when(orders.findById(id)).thenReturn(Optional.empty());

        assertThatThrownBy(() -> service.cancel(new CancelOrderCommand(id, "customer changed their mind")))
                .isInstanceOf(OrderNotFoundException.class);

        verify(orders, never()).save(any());
    }
}
```

Static-import `org.mockito.Mockito.mock`, `when`, `verify`, and `never`, and AssertJ `assertThatThrownBy`. `@Mock` and `@InjectMocks` are not used.

Controller test shape:

```java
@WebMvcTest(OrderController.class)
@AutoConfigureMockMvc(addFilters = false)
class OrderControllerTest {

    @Autowired
    private MockMvcTester mvc;

    @MockitoBean
    private CreateOrderService createOrderService;

    @MockitoBean
    private OrderQueryService orderQueryService;

    @MockitoBean
    private OrderApiMapper mapper;

    @Test
    void shouldReturn404WhenOrderDoesNotExist() {
        UUID id = UUID.randomUUID();
        when(orderQueryService.get(new OrderId(id))).thenThrow(new OrderNotFoundException(new OrderId(id)));

        assertThat(mvc.get().uri("/api/v1/orders/{orderId}", id))
                .hasStatus(HttpStatus.NOT_FOUND)
                .hasContentTypeCompatibleWith(MediaType.APPLICATION_PROBLEM_JSON);
    }
}
```

Imports: `org.springframework.boot.webmvc.test.autoconfigure.WebMvcTest`, `org.springframework.test.web.servlet.assertj.MockMvcTester`, `org.springframework.test.context.bean.override.mockito.MockitoBean`. Static-import AssertJ `assertThat`. The request builder is an AssertJ provider, so `assertThat(mvc.get().uri(...))` returns the MVC assertions. The test starter is `spring-boot-starter-webmvc-test`. It brings the base test starter. Do not also declare `spring-boot-starter-test` unless a module truly has no narrower test starter.

`@WebMvcTest` loads `@RestControllerAdvice`, so `ApiExceptionHandler` is in this slice and the domain exception becomes a problem response. If the advice is missing, MockMvcTester treats the exchange as an unresolved failure and `hasStatus` is the wrong assertion.

`@WebMvcTest` also turns security on. A status or validation test disables the filter chain with `@AutoConfigureMockMvc(addFilters = false)` from `org.springframework.boot.webmvc.test.autoconfigure`. A test that asserts 401 or 403 leaves the filters on and uses `spring-boot-starter-security-test`. Without that split, an unauthenticated call returns 401 and the 404 assertion never runs.

Persistence test shape:

```java
@DataJpaTest
@AutoConfigureTestDatabase(replace = Replace.NONE)
@ImportAutoConfiguration(FlywayAutoConfiguration.class)
@Testcontainers
class OrderPersistenceAdapterIT {

    @Container
    @ServiceConnection
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:18");

    @Autowired
    private OrderJpaRepository jpa;

    @Test
    void shouldRoundTripAPlacedOrder() {
        var adapter = new OrderPersistenceAdapter(jpa, new OrderPersistenceMapperImpl());
        var id = new OrderId(UUID.randomUUID());
        var order = Order.place(id, List.of(new OrderLine(new Sku("ABC"), 1)), Instant.parse("2026-01-01T00:00:00Z"));

        adapter.save(order);

        assertThat(adapter.findById(id)).isPresent();
    }
}
```

`@DataJpaTest` does not start Flyway and does not scan the adapter. Import `org.springframework.boot.flyway.autoconfigure.FlywayAutoConfiguration` with `org.springframework.boot.autoconfigure.ImportAutoConfiguration`, and construct the adapter around the injected Spring Data repository. `OrderPersistenceMapperImpl` is the class MapStruct generates.

The image tag is the PostgreSQL major that production runs. `postgres:18` is the default for a new service. Do not use `latest`. Add `spring-boot-testcontainers` and `org.testcontainers:testcontainers-postgresql` and `org.testcontainers:testcontainers-junit-jupiter`, with versions from the Boot BOM.

Architecture test, using ArchUnit core so it stays a plain JUnit test:

```java
class LayeringArchTest {

    private final JavaClasses classes = new ClassFileImporter().importPackages("com.company.orderservice");

    @Test
    void domainDoesNotDependOnFrameworksOrOuterLayers() {
        noClasses()
                .that().resideInAPackage("..domain..")
                .should().dependOnClassesThat().resideInAnyPackage(
                        "org.springframework..",
                        "jakarta.persistence..",
                        "jakarta.servlet..",
                        "tools.jackson..",
                        "com.fasterxml.jackson..",
                        "..api..",
                        "..application..",
                        "..infrastructure..",
                        "..shared.web..",
                        "..shared.paging..",
                        "jakarta.validation..",
                        "org.slf4j..")
                .check(classes);
    }

    @Test
    void applicationDoesNotDependOnApiOrInfrastructure() {
        noClasses()
                .that().resideInAPackage("..application..")
                .should().dependOnClassesThat().resideInAnyPackage(
                        "..api..",
                        "..infrastructure..",
                        "..shared.web..",
                        "jakarta.persistence..")
                .check(classes);
    }

    @Test
    void controllersLiveInApi() {
        classes()
                .that().areAnnotatedWith(RestController.class)
                .should().resideInAPackage("..api..")
                .check(classes);
    }

    @Test
    void infrastructureDoesNotDependOnApi() {
        noClasses()
                .that().resideInAPackage("..infrastructure..")
                .should().dependOnClassesThat().resideInAPackage("..api..")
                .check(classes);
    }

    @Test
    void noFieldInjection() {
        noFields()
                .should().beAnnotatedWith(Autowired.class)
                .check(classes);
        noFields()
                .should().beAnnotatedWith(Value.class)
                .check(classes);
    }
}
```

Keep ArchUnit on a release that can read the class-file version of the service JDK. Fix the library version rather than deleting the test.

## 20. Language and formatting

- Java 25. Records, switch expressions, text blocks, pattern matching, and `var` are welcome. Use `var` when the right-hand side shows the type (`var order = Order.place(...)`). Do not use `var` when the type is a numeric literal or an unclear method result.
- Unnamed variables (`_`) are fine for an unused lambda parameter.
- No raw types, no wildcard imports, no star static imports, no `import module`.
- Prefer the JDK to a utility library. `String.isBlank`, `List.copyOf`, `Objects.requireNonNull`, and `Optional` cover most needs.
- Comments explain a business or regulatory constraint that the code cannot say. They do not narrate the next line. Commented-out code is deleted.
- Javadoc is for a port, a public domain method whose contract is not obvious, and a properties type. It is not required on every getter.
- Encoding is UTF-8. Line endings are LF. Files end with a newline. Indent is four spaces, which is the Palantir block indent below.

The Maven Enforcer plugin bans these dependencies: `org.projectlombok:lombok`, `com.h2database:h2`, `org.modelmapper:modelmapper`, `junit:junit`, and `org.springframework.boot:spring-boot-starter-web`.

### One layout

Palantir Java Format, style `PALANTIR`, is the only layout. Spotless runs it. The style has no settings we change. When this section was written the pair was Spotless Maven plugin 3.10.4 and Palantir 2.102.0. Bump the two together in a dedicated change, the same way section 3 bumps the Boot parent.

| Rule | What we write |
| --- | --- |
| Block indent | 4 spaces. The `.editorconfig` value stays 4. |
| Continuation | 8 more spaces from the start of the statement being continued |
| Line length | 120 characters |
| Call chain | One call per line when the text before the last `.` would pass column 80, even when the whole statement would fit in 120 |
| Braces | Kernighan and Ritchie: `{` on the same line as the statement, `}` alone on the following line, aligned with that statement |
| Control flow | Braces on every `if`, `else`, `for`, `while`, and `do`, including a body of one line. The formatter does not insert a missing pair, so the author writes them |
| Operators | One space on each side of a binary operator, including `=`, `+`, `-`, `*`, `&&`, and `\|\|` |
| Keywords | One space between `if`, `for`, `while`, `switch`, or `catch` and the `(` |
| Wrapping | Break before `.` and before `+`. The break belongs to the operator that continues the expression |
| Strings | A literal past column 120 is split between words, with `+` at the start of the next line |
| Imports | Static imports, one blank line, then every other import in ASCII order. Unused imports are removed. Member order inside the type is section 5, which the formatter leaves alone |
| Javadoc | Left as written. `formatJavadoc` stays false |

Accepted:

```java
if (user.isActive()) {
    int sum = a + b;
    sendNotification(user);
}
```

Rejected. Missing spaces, a tab indent, and a braceless body:

```java
if(user.isActive())
	sendNotification(user);
int sum=a+b;
```

These other layouts are not a second standard:

| Other guide | What an agent must not copy |
| --- | --- |
| Google Java Style | 2-space indent, 100-character lines, `<style>GOOGLE</style>` |
| Oracle Code Conventions | 80-character lines, a second indent width |
| Eclipse / IntelliJ default scheme | Import groups split into `java`, `javax`, `org`, and `com`, or Ctrl+Alt+L / Cmd+Option+L against that scheme |
| AOSP | `<style>AOSP</style>` |
| Checkstyle, google-java-format, Eclipse formatter | A second tool that restates layout. Architecture checks stay in ArchUnit (section 19) |

A Gradle service uses plugin `com.palantir.java-format` at the same Palantir version. The IntelliJ or Eclipse Palantir plugin is that same version. `mvn spotless:apply` is what CI checks. Run it before committing. Do not mix a repo-wide format into a behavior change. When a formatted line reads badly, extract a local or a small method. The next format run keeps that extraction. Hand-wrapping to fight the formatter does not survive `spotless:check`.

```xml
<plugin>
    <groupId>com.diffplug.spotless</groupId>
    <artifactId>spotless-maven-plugin</artifactId>
    <version>3.10.4</version>
    <configuration>
        <java>
            <palantirJavaFormat>
                <version>2.102.0</version>
                <style>PALANTIR</style>
                <formatJavadoc>false</formatJavadoc>
            </palantirJavaFormat>
            <forbidWildcardImports />
            <forbidModuleImports />
        </java>
    </configuration>
    <executions>
        <execution>
            <goals>
                <goal>check</goal>
            </goals>
            <phase>verify</phase>
        </execution>
    </executions>
</plugin>
```

Leave these steps out of the plugin. Each one fights Palantir:

- `<style>GOOGLE</style>` or `<style>AOSP</style>`.
- `<googleJavaFormat>`, `<eclipse>`, or `<idea>`.
- An `<indent>` step. Palantir already indents with 4 spaces, and a later indent step rewrites them to 2.
- `<importOrder>` with `java|javax|org|com`. Palantir already sorts imports.
- A `<licenseHeader>` on a service that does not already have a header.

## 21. Dependencies

Versions that the Spring Boot parent manages are not repeated in the service pom.

Approved for a standard service:

| Dependency | Notes |
| --- | --- |
| `spring-boot-starter-webmvc` | HTTP API |
| `spring-boot-starter-validation` | Bean Validation |
| `spring-boot-starter-data-jpa` | Persistence |
| `spring-boot-starter-flyway` | Migrations |
| `org.flywaydb:flyway-database-postgresql` | Required alongside Flyway core |
| `org.postgresql:postgresql` | Runtime |
| `spring-boot-starter-security` | Security |
| `spring-boot-starter-security-oauth2-resource-server` | JWT |
| `spring-boot-starter-actuator` | Health and metrics |
| `spring-boot-starter-micrometer-metrics` | Metrics |
| `spring-boot-starter-opentelemetry` | OTLP traces. Required. The platform sets the endpoint. |
| `org.cyclonedx:cyclonedx-maven-plugin` | Build plugin. One SBOM. Not a compile dependency. |
| `spring-boot-starter-restclient` | Outbound HTTP |
| `org.springdoc:springdoc-openapi-starter-webmvc-ui` 3.2.x | OpenAPI. Not managed by the Boot BOM. |
| `org.mapstruct:mapstruct` and `mapstruct-processor` | Current stable release that supports Java 25. Not managed by the Boot BOM. |
| `spring-boot-starter-webmvc-test` | Controller tests. Scope test. |
| `spring-boot-starter-data-jpa-test` | Persistence slice. Scope test. |
| `spring-boot-starter-security-test` | Security assertions. Scope test. |
| `org.springframework.boot:spring-boot-testcontainers` | Scope test |
| `org.testcontainers:testcontainers-junit-jupiter` and `testcontainers-postgresql` | Scope test, versions from the BOM |
| `com.tngtech.archunit:archunit` | Scope test. Current release that reads Java 25 bytecode. |
| `com.diffplug.spotless:spotless-maven-plugin` | Build plugin, not a compile dependency. Palantir style, section 20. |

Not approved:

- Lombok, ModelMapper, Dozer, MapStruct's old `mapstruct-jdk8` artifact.
- H2.
- `RestTemplate` for new clients. An existing client may stay until a change touches it, and that change converts it to `RestClient`.
- WebFlux in an MVC service.
- A second JSON library.
- Spring HATEOAS (`spring-boot-starter-hateoas`). Links are the `Link` record in section 11.5.
- A CRUD generator, a generic repository framework, or a base-controller library.
- `spring-boot-properties-migrator` in a service that was born on Boot 4. It is a migration aid, not a runtime dependency.
- google-java-format, the Eclipse Java formatter, or Checkstyle used as a second layout. Spotless with Palantir is the formatter (section 20).
- Spring Cloud Gateway, a Vault or Secrets Manager client, an Open Policy Agent or Oso client on the first authorization rule, `spring-boot-starter-zipkin`, a Jaeger client, and a second log encoder. Section 17 and section 18 name the single control for each.

A library outside this list is a visible decision in the change description. The agent does not add it silently.

## 22. Agent workflow

Before writing code:

1. Read section 2 and section 5. If the change adds or alters an endpoint, read section 11 before writing the controller. Read section 17 before authentication, authorization, secrets, or a deploy file, and section 18 before a log line.
2. Find the feature package. Name it with the grouping algorithm. Do not create a parallel tree.
3. List the files you will add, each with its layer, and check them against the table in section 5. Choose the path, method, status, cache header, and link relations from section 11. Do not invent a second URL or error style.
4. Reuse the starter types in `shared.error` and `shared.web`. Do not create a second exception hierarchy or a second page wrapper.
5. Use the Boot 4 column of the trap table for every import and dependency.

While writing code:

- Match the canonical shapes in section 10 and the accepted snippets in section 8. Same annotations, same constructor injection, same mapping split.
- A service stays concrete, public, not final, and stateless. Copy the accepted transaction method, not the Lombok and `@Autowired` example.
- Names use section 7 (`OrderId`, not `OrderID`). Member order follows section 5. Format with `mvn spotless:apply`. Palantir's `PALANTIR` style is the layout. Google's 2-space style and Oracle's 80-column style are not.
- Configuration is a validated `@ConfigurationProperties` record from section 16. Do not add `@Value`.
- The service validates the JWT. `@PreAuthorize` checks one authority. The aggregate keeps the business rule. Do not add a gateway, a mesh, a vault client, OPA, Oso, Zipkin, or `X-Correlation-ID`.
- Do not add an interface to an application service. Do not add a `Utils`, `Helper`, or base class to satisfy a principle.
- Do not put `@Entity` on a domain type.
- Do not "improve" unrelated code, reformat the repository, or upgrade the framework inside a feature change.
- Do not leave TODOs in place of the required test, the migration, or the error mapping.

When you finish, the change summary names:

- The feature package.
- Each file and its layer.
- The tests.
- The migration id, or "no schema change".
- Any new dependency and the approved-list row that allows it.

If a rule cannot be followed, say which rule and why. Do not quietly start a second style.

## 23. Review checklist

- [ ] The change lives in one feature package, or in `shared` / `config` for a starter concern.
- [ ] No new root package from the forbidden list.
- [ ] Domain files import none of Spring, JPA, Jackson, or Servlet.
- [ ] The controller depends on application services and a mapper only.
- [ ] New types match an accepted snippet in section 8. No one-implementation service interface, no `Utils` or `Helper`, no base class, no setter that skips an invariant.
- [ ] JPA entities and Spring Data interfaces are package-private and stay in `infrastructure/persistence`.
- [ ] Spring Data extends `Repository` and declares the methods it uses.
- [ ] `@Transactional` is on a public method of a non-final application service. Queries are read-only. A second transaction boundary is a second bean.
- [ ] Settings are a validated `@ConfigurationProperties` record. No `@Value`. No secret inside a record. No default password.
- [ ] No HTTP call inside an open database transaction without an outbox.
- [ ] Errors go through `ApiExceptionHandler` and Problem Details. No failure is returned as HTTP 200.
- [ ] Paths, methods, statuses, pagination, `ETag`, `Cache-Control`, and `links` match section 11. `PUT` and `PATCH` require `If-Match`.
- [ ] Resource URIs come from `ApiLinks` and the public base URL. The body has no view-model labels and no executable content.
- [ ] A schema change is a new immutable Flyway migration, and `ddl-auto` remains `validate`.
- [ ] Names match section 7, including the four shapes (`OrderId`, not `OrderID`). Source types have no `Impl`, no `I` prefix, no `DTO`, and no `Utils`. MapStruct's generated `{Mapper}Impl` is the only exception.
- [ ] The file follows section 5: explicit imports, one top-level type, constants, fields, constructor, then behavior. A constant sits on the owning type.
- [ ] Constructor injection. No `@Value`, no field injection, no Lombok.
- [ ] Time comes from `Clock`. Money is `BigDecimal`. Ids are value objects.
- [ ] The required test from section 19 exists and fails if the new behavior is removed.
- [ ] The service validates the bearer JWT. `permitAll` is only health and info, plus springdoc on `local`. `@PreAuthorize` checks an authority. The business rule stays on the aggregate.
- [ ] Secrets are platform-injected environment variables. The image runs as `10001:10001`. `catalog-info.yaml` and a default-deny `NetworkPolicy` exist for a new service.
- [ ] Outbound clients use the injected `RestClient.Builder`, `https`, and timeouts. Certificate checks stay on.
- [ ] Logs are ECS JSON on stdout with the Micrometer `traceId`. No token, card, email, or body is logged.
- [ ] Boot 4 artifact and import names are the ones in section 3.
- [ ] `mvn spotless:check` passes on the Palantir `PALANTIR` profile from section 20 (four spaces, no second formatter). `mvn test` and the relevant `*IT` were run for the change.

## 24. Worked example

Task: add "cancel an order". The `order` feature already exists and already has `OrderRepository.save` and `findById`.

| File | Layer | What changes |
| --- | --- | --- |
| `order/api/OrderController.java` | api | `POST /api/v1/orders/{orderId}/cancellations`. Success is 200 with `Cache-Control: no-store` and no `cancel` link. Rejection is 422 |
| `order/api/CancelOrderRequest.java` | api | record with `@NotBlank String reason` |
| `order/application/CancelOrderService.java` | application | load, `order.cancel(reason)`, save |
| `order/domain/Order.java` | domain | `cancel` enforces the state rule |
| `order/domain/OrderCancellationRejectedException.java` | domain | extends `BusinessRuleException`, code `order-cancellation-rejected` |
| `order/domain/OrderTest.java` | test | shipped order is rejected; placed order becomes cancelled; a second cancel is a no-op |
| `order/api/OrderControllerTest.java` | test | 422 problem body when the service throws the rejection |

No new repository method. No new JPA entity. No new shared type. No `OrderService` that both creates and cancels. The existing handler already maps `BusinessRuleException` to 422.
