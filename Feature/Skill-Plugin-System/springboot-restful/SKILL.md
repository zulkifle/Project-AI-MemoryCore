---
name: springboot-restful
description: >
  Use this skill for EVERY task involving Spring Boot, Java REST APIs, OOP design patterns,
  or any Java backend development. Trigger when the user asks to create, scaffold, refactor,
  review, or explain any Spring Boot project, REST endpoint, Java class, service layer,
  repository, entity, DTO, exception handler, or test. Also trigger for Docker/CI-CD tasks
  related to a Spring Boot app. This skill enforces industry best practices, clean layered
  architecture, and teaches design patterns (Builder, Factory, Strategy, Decorator) inline
  with every code sample produced.
---

# Spring Boot RESTful API — Best Practices Skill

This skill governs how Claude writes, scaffolds, reviews, and teaches Spring Boot REST APIs
for a developer who is learning Java and wants to improve their OOP and design pattern skills.

## Core Philosophy

- **Teach while doing.** Every code block must include a short inline comment explaining the
  design decision (e.g., why Builder is used here, what interface segregation achieves).
- **Layered architecture always.** Never mix concerns. Controller → Service → Repository → Entity.
- **Design patterns are not optional.** Apply them naturally in context — Builder for entities,
  Factory for services, Strategy for interchangeable logic, Decorator for cross-cutting concerns.
- **Best-practice naming and structure** from the start so the user builds muscle memory.
- **DevOps-ready.** Always produce Dockerfile + docker-compose.yml when scaffolding a project.

---

## Project Structure

Every Spring Boot project must follow this package structure:

```
src/main/java/com/{company}/{project}/
├── controller/       # REST layer — handles HTTP only, no business logic
├── service/
│   ├── impl/         # Concrete service implementations
│   └── {Name}Service.java  # Interface (enables Strategy + DI)
├── repository/       # Spring Data JPA interfaces
├── entity/           # JPA entities (use Builder pattern)
├── dto/
│   ├── request/      # Input DTOs (what the API receives)
│   └── response/     # Output DTOs (what the API returns — never expose entities directly)
├── exception/        # Custom exceptions + global handler
├── config/           # Spring configuration classes
└── mapper/           # Entity <-> DTO conversion (use MapStruct or manual)
```

---

## Layer Rules

### Controller
- Annotate with `@RestController` and `@RequestMapping("/api/v1/resource")`
- **Only** handles: request validation, calling service, returning ResponseEntity
- Never contains business logic or database calls
- Always return `ResponseEntity<ApiResponse<T>>` (wrap in ApiResponse)

```java
// Pattern: Controller as thin HTTP adapter
@RestController
@RequestMapping("/api/v1/products")
@RequiredArgsConstructor  // Lombok - constructor injection (best practice over @Autowired)
public class ProductController {

    private final ProductService productService;  // injected by interface, not impl

    @GetMapping("/{id}")
    public ResponseEntity<ApiResponse<ProductResponse>> getById(@PathVariable Long id) {
        return ResponseEntity.ok(ApiResponse.success(productService.findById(id)));
    }

    @PostMapping
    public ResponseEntity<ApiResponse<ProductResponse>> create(
            @Valid @RequestBody CreateProductRequest request) {
        return ResponseEntity.status(HttpStatus.CREATED)
                .body(ApiResponse.success(productService.create(request)));
    }
}
```

### Service Interface + Implementation
- Always define a **Java interface** for every service (enables Strategy pattern + testability)
- Put implementation in `service/impl/`

```java
// Interface: defines the contract (Strategy pattern principle)
public interface ProductService {
    ProductResponse findById(Long id);
    ProductResponse create(CreateProductRequest request);
    Page<ProductResponse> findAll(Pageable pageable);
    ProductResponse update(Long id, UpdateProductRequest request);
    void delete(Long id);
}

// Implementation: concrete strategy
@Service
@RequiredArgsConstructor
@Transactional  // default transaction boundary at service level
public class ProductServiceImpl implements ProductService {

    private final ProductRepository productRepository;
    private final ProductMapper productMapper;

    @Override
    @Transactional(readOnly = true)  // read-only for queries — best practice for performance
    public ProductResponse findById(Long id) {
        Product product = productRepository.findById(id)
                .orElseThrow(() -> new ResourceNotFoundException("Product", "id", id));
        return productMapper.toResponse(product);
    }

    @Override
    public ProductResponse create(CreateProductRequest request) {
        // Builder pattern: readable object construction, avoids telescoping constructors
        Product product = Product.builder()
                .name(request.getName())
                .price(request.getPrice())
                .stock(request.getStock())
                .build();
        return productMapper.toResponse(productRepository.save(product));
    }
}
```

### Entity
- Always use `@Builder` (Lombok) on entities — Builder pattern for object construction
- Always extend a `BaseEntity` (audit fields: createdAt, updatedAt, createdBy)
- Never expose entity directly through API — always map to DTO

```java
@Entity
@Table(name = "products")
@Getter
@Builder
@NoArgsConstructor
@AllArgsConstructor
@EntityListeners(AuditingEntityListener.class)
public class Product extends BaseEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, length = 255)
    private String name;

    @Column(nullable = false, precision = 10, scale = 2)
    private BigDecimal price;

    @Column(nullable = false)
    private Integer stock;
}

// BaseEntity for audit fields — DRY principle
@MappedSuperclass
@EntityListeners(AuditingEntityListener.class)
@Getter
public abstract class BaseEntity {

    @CreatedDate
    @Column(updatable = false)
    private LocalDateTime createdAt;

    @LastModifiedDate
    private LocalDateTime updatedAt;
}
```

### DTO (Data Transfer Object)
- Request DTOs: include Bean Validation annotations (`@NotNull`, `@Size`, etc.)
- Response DTOs: use `record` (Java 16+) or `@Value` (Lombok) — immutable

```java
// Request DTO — validates input before it reaches service
@Getter
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class CreateProductRequest {

    @NotBlank(message = "Product name is required")
    @Size(min = 2, max = 255, message = "Name must be between 2 and 255 characters")
    private String name;

    @NotNull(message = "Price is required")
    @Positive(message = "Price must be positive")
    private BigDecimal price;

    @NotNull(message = "Stock is required")
    @Min(value = 0, message = "Stock cannot be negative")
    private Integer stock;
}

// Response DTO — Java record (immutable, concise)
public record ProductResponse(
    Long id,
    String name,
    BigDecimal price,
    Integer stock,
    LocalDateTime createdAt
) {}
```

### Exception Handling
- Always create a global exception handler with `@RestControllerAdvice`
- Create typed custom exceptions (e.g., `ResourceNotFoundException`, `BusinessException`)
- Always return consistent `ApiResponse<Void>` on error

```java
// Custom exception
public class ResourceNotFoundException extends RuntimeException {
    public ResourceNotFoundException(String resource, String field, Object value) {
        super(String.format("%s not found with %s: '%s'", resource, field, value));
    }
}

// Global handler — Decorator pattern: wraps all exceptions uniformly
@RestControllerAdvice
@Slf4j
public class GlobalExceptionHandler {

    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ApiResponse<Void>> handleNotFound(ResourceNotFoundException ex) {
        log.warn("Resource not found: {}", ex.getMessage());
        return ResponseEntity.status(HttpStatus.NOT_FOUND)
                .body(ApiResponse.error(ex.getMessage()));
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ApiResponse<Void>> handleValidation(
            MethodArgumentNotValidException ex) {
        String message = ex.getBindingResult().getFieldErrors().stream()
                .map(e -> e.getField() + ": " + e.getDefaultMessage())
                .collect(Collectors.joining(", "));
        return ResponseEntity.status(HttpStatus.BAD_REQUEST)
                .body(ApiResponse.error(message));
    }
}

// Standardised API response wrapper — used on ALL endpoints
@Getter
@Builder
public class ApiResponse<T> {
    private boolean success;
    private String message;
    private T data;
    private LocalDateTime timestamp;

    public static <T> ApiResponse<T> success(T data) {
        return ApiResponse.<T>builder()
                .success(true).data(data)
                .timestamp(LocalDateTime.now()).build();
    }

    public static ApiResponse<Void> error(String message) {
        return ApiResponse.<Void>builder()
                .success(false).message(message)
                .timestamp(LocalDateTime.now()).build();
    }
}
```

---

## Design Patterns — How to Apply in Spring Boot

| Pattern | Where to apply | Spring Boot example |
|---------|---------------|---------------------|
| **Builder** | Entity construction, DTO creation, ApiResponse | `Product.builder().name(...).build()` |
| **Factory** | Creating different implementations dynamically | `PaymentProcessorFactory.getProcessor(type)` |
| **Strategy** | Interchangeable algorithms / behaviours | `DiscountStrategy` interface, multiple impls |
| **Decorator** | Logging, caching, security wrappers | `@Cacheable`, AOP `@Around`, filter chain |
| **Singleton** | Spring beans (default scope is singleton) | `@Service`, `@Repository`, `@Component` |
| **Proxy** | Spring AOP, `@Transactional`, `@Cacheable` | Spring wraps your bean automatically |
| **Template Method** | Abstract service with reusable steps | Abstract base service class |
| **Observer** | Application events | `ApplicationEventPublisher` + `@EventListener` |

---

## Standard Dependencies (pom.xml)

Always include these for a production-ready REST API:

```xml
<!-- Core -->
<dependency>spring-boot-starter-web</dependency>
<dependency>spring-boot-starter-data-jpa</dependency>
<dependency>spring-boot-starter-validation</dependency>
<dependency>spring-boot-starter-actuator</dependency>

<!-- Database -->
<dependency>postgresql</dependency>  <!-- or h2 for dev/test -->
<dependency>flyway-core</dependency>  <!-- DB migrations — never use ddl-auto=create in prod -->

<!-- Developer productivity -->
<dependency>lombok</dependency>

<!-- Testing -->
<dependency>spring-boot-starter-test</dependency>  <!-- includes JUnit 5 + Mockito -->
<dependency>testcontainers</dependency>  <!-- real DB in integration tests -->

<!-- API docs -->
<dependency>springdoc-openapi-starter-webmvc-ui</dependency>
```

---

## application.yml Best Practices

```yaml
spring:
  datasource:
    url: ${DB_URL:jdbc:postgresql://localhost:5432/mydb}
    username: ${DB_USERNAME:postgres}
    password: ${DB_PASSWORD:postgres}
  jpa:
    hibernate:
      ddl-auto: validate   # NEVER 'create' or 'update' in production — use Flyway
    show-sql: false          # disable in production
    properties:
      hibernate:
        format_sql: true
  flyway:
    enabled: true
    locations: classpath:db/migration

server:
  port: 8080

management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics
```

---

## Testing Standards

Every service and controller must have tests. Minimum coverage expectations:

```java
// Unit test — test service in isolation with Mockito
@ExtendWith(MockitoExtension.class)
class ProductServiceTest {

    @Mock
    private ProductRepository productRepository;

    @Mock
    private ProductMapper productMapper;

    @InjectMocks
    private ProductServiceImpl productService;

    @Test
    void findById_whenProductExists_returnsResponse() {
        // Arrange — Given
        Product product = Product.builder().id(1L).name("Laptop").build();
        ProductResponse expected = new ProductResponse(1L, "Laptop", null, null, null);
        when(productRepository.findById(1L)).thenReturn(Optional.of(product));
        when(productMapper.toResponse(product)).thenReturn(expected);

        // Act — When
        ProductResponse result = productService.findById(1L);

        // Assert — Then
        assertThat(result.name()).isEqualTo("Laptop");
        verify(productRepository).findById(1L);
    }
}

// Integration test — test full HTTP layer with MockMvc
@WebMvcTest(ProductController.class)
class ProductControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @MockBean
    private ProductService productService;

    @Test
    void getById_returnsProduct() throws Exception {
        when(productService.findById(1L))
                .thenReturn(new ProductResponse(1L, "Laptop", BigDecimal.TEN, 5, null));

        mockMvc.perform(get("/api/v1/products/1"))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.data.name").value("Laptop"));
    }
}
```

---

## Docker Setup (always include when scaffolding)

```dockerfile
# Dockerfile — multi-stage build for smaller image
FROM eclipse-temurin:21-jdk-alpine AS builder
WORKDIR /app
COPY mvnw pom.xml ./
COPY .mvn .mvn
RUN ./mvnw dependency:go-offline
COPY src src
RUN ./mvnw package -DskipTests

FROM eclipse-temurin:21-jre-alpine
WORKDIR /app
COPY --from=builder /app/target/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

```yaml
# docker-compose.yml
services:
  app:
    build: .
    ports:
      - "8080:8080"
    environment:
      DB_URL: jdbc:postgresql://db:5432/mydb
      DB_USERNAME: postgres
      DB_PASSWORD: postgres
    depends_on:
      db:
        condition: service_healthy

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: mydb
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 5s
      retries: 5
    volumes:
      - postgres_data:/var/lib/postgresql/data

volumes:
  postgres_data:
```

---

## What Claude Should Do When Using This Skill

1. **When asked to scaffold a project**: produce full folder structure + all base classes + Dockerfile + docker-compose.yml in one go as files the user can download.

2. **When writing any Java class**: apply relevant design patterns automatically and annotate every non-obvious decision with a `// Pattern:` or `// Why:` comment.

3. **When the user asks about a pattern**: demonstrate it inside a real Spring Boot context, not a generic textbook example.

4. **When reviewing code**: flag violations of layered architecture, missing validation, entity exposure in API, or missing tests.

5. **When the user is stuck**: explain the concept simply first, then show the code — never code-dump without context.

6. **Always prefer interfaces over concrete classes** for service injection. This is the foundation of testable, maintainable Spring Boot code.
