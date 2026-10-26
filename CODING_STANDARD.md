# Java Spring Boot Microservice Coding Standard

Status: mandatory for every person and every AI agent that writes code for our Spring Boot microservices.
Baseline: Java 25, Spring Boot 4.1.x (4.1.2 when this file was written), imperative Spring MVC, PostgreSQL.
This file is the only coding standard. A service may record a temporary exception in its own README under "Standard exceptions", with the reason and the review date. Agents do not invent exceptions.

Copy this file to the root of each microservice, and copy `.cursor/rules/java-spring-boot-standard.mdc` with it. Agents read this file before generating or modifying code.

## 1. How to use this file

Humans review against the checklist in section 23.
Agents follow this order:

1. Section 2, then the Boot 4 trap table in section 3.
2. Section 5, and place every new file before writing it.
3. The layer section for the code you are about to write (sections 10–15). For any HTTP endpoint, read section 11 before choosing the path, method, status, or headers.
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
- The service is a stateless OAuth2 resource server. Authenticate every route except the health and info probes.
- Logs are structured. They contain the aggregate id and the trace id. They never contain secrets, tokens, or raw credentials.
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
| Operations | Actuator, Micrometer, structured JSON logs, OpenTelemetry when the platform traces |
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
├── .editorconfig
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

`.gitignore` includes `application-local.yml`, `.env`, and IDE metadata. Secrets never go in git.

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

### Packages and files

- Packages are lowercase singular nouns. No underscores, no acronyms in mixed case.
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

Method name: `should{Expected}When{Condition}`.

```text
shouldRejectCancellationWhenOrderIsShipped
shouldReturn404WhenOrderDoesNotExist
```

## 8. Design principles

These are rules for where code goes, not slogans.

**Dependency rule.** Inner layers do not know outer layers. Section 6 is the enforceable form.

**Single responsibility.** A class changes for one reason. A type that validates HTTP, applies a price rule, and writes SQL is three types in three layers. A method that needs a comment to separate two phases is two methods.

**Open/closed, in practice.** Add a new strategy type when a third variant appears. A two-branch `if` stays an `if`. Do not introduce a strategy, a factory, and an interface for two cases.

**Liskov.** Every implementation of a port honors the same failures. If `OrderRepository.findById` returns an empty `Optional` when the row is missing, no adapter throws a vendor exception for that case instead.

**Interface segregation.** A repository port exposes the methods the use cases need, typically `save` and `findById`, plus a specific finder when a use case needs it. Query screens that are not aggregate edits use a query port that returns a view. They do not add reporting methods to the aggregate repository.

**Dependency inversion.** Domain and application own the interfaces. Infrastructure implements them. Spring injecting a concrete adapter into the service is the inversion. `ApplicationContext.getBean` is not.

**Fail fast.** An aggregate, a value object, and an identifier cannot be constructed in an invalid state. Check at the constructor. Do not repair bad data deep in a call chain.

**Command-query separation.** A command method changes state and returns the aggregate or its id. A query method does not change state. Writes and reads are different application services when both exist. One service with a single `create` method does not also grow a `list`.

**Tell, don't ask.** The application service tells the aggregate `order.cancel(reason)`. It does not copy the status out, decide, and copy a new status in. Getters exist for mapping and for tests, not for exporting decisions.

**Law of Demeter.** A caller uses the object it has. It does not walk `order.customer().address().country()` to make a policy decision. That decision is a method on the aggregate or a domain policy that receives the values it needs.

**Composition over inheritance.** Do not inherit to reuse code. No `BaseService`, no `AbstractAdapter`. Extend a type only for a closed domain hierarchy that is a true is-a, and keep it to one level. Prefer a port plus an adapter.

**YAGNI.** Do not add a layer, a cache, a message, an interface, or a feature package for a need that is not in the current change. The repository interface and the client interface are required by the dependency rule. An interface for `CreateOrderService` is not.

**DRY, after the third copy.** Duplicate a small mapping or a small value object once rather than create the wrong abstraction. On the third copy, extract it to the owning feature or to `shared` using the admission test.

**Explicit dependencies.** Everything a class needs is a constructor parameter. No static service lookup, no field injection, no hidden `new` of an outbound client inside a method.

**Least surprise.** Status codes match section 12. Names match section 7. A reader should predict the package from the type name.

**Small use cases.** One application-service method is one transaction and one reason for the user to call the service. Multi-step workflows call several use cases or a domain method. They do not become a 400-line service.

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

`@Transactional` is on the application service, on the public method or on the class for a query service. It is not on the controller, not on the domain, and not on the Spring Data interface. Self-invocation does not start a transaction; do not call a `@Transactional` method of `this` and expect a new boundary.

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
- Map a remote 404 to our `NotFoundException` only when absence is part of our use case. Map a timeout or a 5xx to a dedicated exception that the handler turns into `502` or `504`. Do not wrap it as a 500 with the remote body attached.
- WebClient and declarative HTTP interfaces are for a WebFlux service. An MVC service does not add them.

Messaging, when the feature publishes:

- Write an outbox row in the same transaction as the aggregate.
- A relay in `infrastructure/messaging` publishes after commit and marks the row sent.
- Delivery is at least once. Consumers are idempotent.
- Do not publish to a broker and then commit the database, and do not commit and then publish in the request thread with no outbox.
- Until a feature publishes, do not add an outbox "for later".

In-process reactions use a domain event record. The aggregate collects events. The application service pulls them and publishes with `ApplicationEventPublisher`. The listener is `@TransactionalEventListener(phase = AFTER_COMMIT)` on an application service of the consuming feature.

## 16. Configuration

- `application.yml` is production-safe and committed.
- `application-local.yml.example` is committed, with obvious local placeholders.
- `application-local.yml` is gitignored.
- Profiles are `local`, `test`, and `prod`. The platform sets `prod`. The jar does not default itself to `local`.
- Secrets are environment variables with no default: `${DB_PASSWORD}`. A missing secret fails startup.
- Grouped settings are `@ConfigurationProperties` records scanned from the application class. `@Value` is not used. `@ConstructorBinding` is not added; a record already has one constructor.

```java
@ConfigurationProperties(prefix = "app.payment")
public record PaymentClientProperties(
        URI baseUrl,
        Duration connectTimeout,
        Duration readTimeout
) {
}
```

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

## 17. Security

- Every service ships `SecurityFilterChain`. Sessions are stateless. CSRF is disabled because the API is a token API with no cookie session.
- The resource server validates JWTs. Authorization rules that are per use case use `@PreAuthorize` on the application service, with method security enabled.
- `permitAll` is limited to `/actuator/health/**` and `/actuator/info`.
- Actuator's business port is `8081`. `env`, `beans`, `heapdump`, and `mappings` are not exposed.
- CORS, when a browser calls the service directly, lists explicit origins. It is not `*`.
- Passwords, tokens, card numbers, national identifiers, and connection strings are not logged, not put in exceptions, and not put in Problem Details.
- SQL goes through JPA or Spring's parameter binding. Do not concatenate a request value into SQL, JPQL, or a native query.
- Dependencies are reviewed when added. Do not add a parser for XML from external users without saying so in the change.

## 18. Logging and observability

- SLF4J, one logger per class: `private static final Logger log = LoggerFactory.getLogger(CreateOrderService.class);`
- Parameterized messages. No string concatenation in the call.
- INFO for a state change that an operator would search for. DEBUG for diagnostic detail. WARN for a rejected business action. ERROR for a failed operation a human must notice.
- Include the aggregate id. Include a stable event phrase at the start of the message (`Order placed.`).
- Production logs are ECS JSON via `logging.structured.format.console=ecs`. Trace and span ids come from Micrometer Tracing / OpenTelemetry. The problem body echoes `traceId` when MDC has it.
- Custom metrics use Micrometer and a lowercase dot name: `orders.placed`, with a `result` tag of `success` or `failure`. Do not put high-cardinality values (raw ids, emails) in tags.
- Health distinguishes liveness and readiness. Readiness fails when the database is unreachable.

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
- No raw types, no wildcard imports, no star static imports.
- Prefer the JDK to a utility library. `String.isBlank`, `List.copyOf`, `Objects.requireNonNull`, and `Optional` cover most needs.
- Comments explain a business or regulatory constraint that the code cannot say. They do not narrate the next line. Commented-out code is deleted.
- Javadoc is for a port, a public domain method whose contract is not obvious, and a properties type. It is not required on every getter.
- Format with the Spotless Maven plugin and Palantir Java Format. Run `mvn spotless:apply` before committing. CI runs `mvn spotless:check`. Do not hand-wrap code to fight the formatter, and do not mix a repo-wide format into a behavior change.
- Encoding is UTF-8. Line endings are LF. Indent is four spaces. Files end with a newline.

The Maven Enforcer plugin bans these dependencies: `org.projectlombok:lombok`, `com.h2database:h2`, `org.modelmapper:modelmapper`, `junit:junit`, and `org.springframework.boot:spring-boot-starter-web`.

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
| `spring-boot-starter-opentelemetry` | When the platform traces |
| `spring-boot-starter-restclient` | Outbound HTTP |
| `org.springdoc:springdoc-openapi-starter-webmvc-ui` 3.2.x | OpenAPI. Not managed by the Boot BOM. |
| `org.mapstruct:mapstruct` and `mapstruct-processor` | Current stable release that supports Java 25. Not managed by the Boot BOM. |
| `spring-boot-starter-webmvc-test` | Controller tests. Scope test. |
| `spring-boot-starter-data-jpa-test` | Persistence slice. Scope test. |
| `spring-boot-starter-security-test` | Security assertions. Scope test. |
| `org.springframework.boot:spring-boot-testcontainers` | Scope test |
| `org.testcontainers:testcontainers-junit-jupiter` and `testcontainers-postgresql` | Scope test, versions from the BOM |
| `com.tngtech.archunit:archunit` | Scope test. Current release that reads Java 25 bytecode. |

Not approved:

- Lombok, ModelMapper, Dozer, MapStruct's old `mapstruct-jdk8` artifact.
- H2.
- `RestTemplate` for new clients. An existing client may stay until a change touches it, and that change converts it to `RestClient`.
- WebFlux in an MVC service.
- A second JSON library.
- Spring HATEOAS (`spring-boot-starter-hateoas`). Links are the `Link` record in section 11.5.
- A CRUD generator, a generic repository framework, or a base-controller library.
- `spring-boot-properties-migrator` in a service that was born on Boot 4. It is a migration aid, not a runtime dependency.

A library outside this list is a visible decision in the change description. The agent does not add it silently.

## 22. Agent workflow

Before writing code:

1. Read section 2 and section 5. If the change adds or alters an endpoint, read section 11 before writing the controller.
2. Find the feature package. Name it with the grouping algorithm. Do not create a parallel tree.
3. List the files you will add, each with its layer, and check them against the table in section 5. Choose the path, method, status, cache header, and link relations from section 11. Do not invent a second URL or error style.
4. Reuse the starter types in `shared.error` and `shared.web`. Do not create a second exception hierarchy or a second page wrapper.
5. Use the Boot 4 column of the trap table for every import and dependency.

While writing code:

- Match the canonical shapes in section 10. Same annotations, same constructor injection, same mapping split.
- Do not add an interface to an application service.
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
- [ ] Invariants are enforced in the aggregate, not only in annotations.
- [ ] JPA entities and Spring Data interfaces are package-private and stay in `infrastructure/persistence`.
- [ ] Spring Data extends `Repository` and declares the methods it uses.
- [ ] `@Transactional` is on the application service. Queries are read-only.
- [ ] No HTTP call inside an open database transaction without an outbox.
- [ ] Errors go through `ApiExceptionHandler` and Problem Details. No failure is returned as HTTP 200.
- [ ] Paths, methods, statuses, pagination, `ETag`, `Cache-Control`, and `links` match section 11. `PUT` and `PATCH` require `If-Match`.
- [ ] Resource URIs come from `ApiLinks` and the public base URL. The body has no view-model labels and no executable content.
- [ ] A schema change is a new immutable Flyway migration, and `ddl-auto` remains `validate`.
- [ ] Names match section 7. Source types have no `Impl`, no `I` prefix, no `DTO`, and no `Utils`. MapStruct's generated `{Mapper}Impl` is the only exception.
- [ ] Constructor injection. No `@Value`, no field injection, no Lombok.
- [ ] Time comes from `Clock`. Money is `BigDecimal`. Ids are value objects.
- [ ] The required test from section 19 exists and fails if the new behavior is removed.
- [ ] Outbound clients have timeouts. Logs have no secrets.
- [ ] Boot 4 artifact and import names are the ones in section 3.
- [ ] `mvn spotless:check`, `mvn test`, and the relevant `*IT` were run for the change.

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
