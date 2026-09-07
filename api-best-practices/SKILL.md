---
name: api-best-practices
description: Enforces REST API best practices for Java Spring Boot projects — RESTful resource naming, correct HTTP status codes, DTOs (never entities) in controllers, Bean Validation, Controller-Service-Repository layering, pagination, global exception handling, idempotency, resilience (retries/circuit breakers), performance (caching, N+1, pagination), Spring Security/JWT, URL versioning, and OpenAPI docs. Works with Spring Boot 3.x and 4.x — nothing here is version-specific. Use this skill whenever creating or modifying ANY Spring Boot controller, endpoint, REST API, DTO, exception handler, or service class — even if the user just says "add an endpoint", "create a CRUD API", "build a controller", or asks to review/refactor existing Spring Boot API code. Also use when the user asks about idempotency, retries, rate limiting, circuit breakers, or API performance in a Spring Boot context.
---

# Spring Boot REST API Best Practices

Apply these conventions to every REST endpoint created or modified in a Spring Boot project. The most common API defects — leaked entities, inconsistent errors, unbounded queries, missing validation, double-processed retries — are cheap to prevent at write time and expensive to fix later.

When reviewing or refactoring existing code, audit it against the [Verification Checklist](#verification-checklist) at the end and fix violations.

---

## 1. Resource naming

Use plural nouns. Never put verbs or actions in URLs — the HTTP method is the verb.

```java
// Correct
@GetMapping("/api/v1/users")            // list
@GetMapping("/api/v1/users/{id}")       // get one
@PostMapping("/api/v1/users")           // create
@PutMapping("/api/v1/users/{id}")       // full update
@PatchMapping("/api/v1/users/{id}")     // partial update
@DeleteMapping("/api/v1/users/{id}")    // delete

// Wrong — never generate these
@GetMapping("/getAllUsers")
@PostMapping("/createUser")
```

Nest sub-resources hierarchically: `/api/v1/users/{userId}/posts/{postId}`. Nest at most two levels deep; beyond that, promote the sub-resource to a top-level path.

Put the base path (`/api/v1/users`) in `@RequestMapping` on the class, not repeated on every method.

## 2. HTTP status codes

Return the status that reflects what actually happened. Defaults per operation:

| Operation | Success status |
|---|---|
| GET | 200 OK |
| POST (create) | 201 Created |
| PUT / PATCH | 200 OK |
| DELETE | 204 No Content |

`202 Accepted` applies to POST or PUT when the operation is queued for async processing rather than completed synchronously — there's no result body yet.

PUT vs. PATCH: PUT replaces the *whole* resource (client sends every field; omitted fields are cleared) and is naturally idempotent. PATCH applies a *partial* update (client sends only the changed fields) and is idempotent only if the change is expressed as an absolute value rather than a relative one (e.g. "set status = ACTIVE" is idempotent, "increment count by 1" is not).

Error mapping (produced by the global exception handler, not inline in controllers):

- 400 — malformed request or failed validation
- 401 — not authenticated
- 403 — authenticated but not allowed
- 404 — resource not found
- 409 — conflict (e.g., duplicate unique field)
- 422 — syntactically valid but unprocessable business-wise
- 429 — rate limit exceeded (see §11 Rate limiting)
- 500 — unexpected server error (never leak stack traces or internal messages)
- 503 — service temporarily unavailable (downstream dependency down, or maintenance)

The set worth knowing cold: `200, 201, 202, 204, 400, 401, 403, 404, 409, 429, 500, 503`.

```java
@PostMapping
public ResponseEntity<UserResponse> createUser(@RequestBody @Valid CreateUserRequest request) {
    UserResponse created = userService.createUser(request);
    return ResponseEntity.status(HttpStatus.CREATED).body(created);
}

@DeleteMapping("/{id}")
public ResponseEntity<Void> deleteUser(@PathVariable Long id) {
    userService.deleteUser(id);
    return ResponseEntity.noContent().build();
}
```

## 3. DTOs, never entities

Controllers must never accept or return JPA entities. Every request body and response body is a DTO.

- Implement DTOs as Java records.
- Give response DTOs a static `from(Entity)` factory method (or use a dedicated mapper for complex graphs).
- Separate request and response DTOs: `CreateUserRequest`, `UpdateUserRequest`, `UserResponse`. Add a slimmer `UserSummaryResponse` for list endpoints when the detail view is heavy.
- Never expose passwords, internal flags, or fields the client has no business seeing.

```java
public record UserResponse(Long id, String name, String email, LocalDateTime createdAt) {
    public static UserResponse from(User user) {
        return new UserResponse(user.getId(), user.getName(), user.getEmail(), user.getCreatedAt());
    }
}
```

Why: hides internal/sensitive fields, lets the DB schema evolve without breaking the API contract, and avoids serializing lazy-loaded associations.

## 4. Bean Validation on every request body

Never write manual `if (x == null)` validation in controllers or services for request shape. Use Jakarta Bean Validation annotations on the request DTO and `@Valid` on the parameter.

```java
public record CreateUserRequest(
    @NotBlank(message = "Name is required")
    @Size(min = 2, max = 100) String name,

    @NotBlank @Email(message = "Email must be valid") String email,

    @NotBlank @Size(min = 8, message = "Password must be at least 8 characters") String password
) {}

@PostMapping
public ResponseEntity<UserResponse> createUser(@RequestBody @Valid CreateUserRequest request) { ... }
```

Always include human-readable `message` values — they surface in the error response. For cross-field or DB-dependent rules (e.g., unique email), validate in the service layer and throw a domain exception; write a custom `@Constraint` annotation only when the rule is reused across DTOs.

## 5. Controller → Service → Repository layering

Each layer has one job. Do not skip layers or leak responsibilities:

- **Controller**: HTTP mapping, DTO conversion boundary, status codes. No business logic, no repository access.
- **Service** (`@Service`, `@Transactional`): business rules, orchestration, entity↔DTO mapping, domain exceptions.
- **Repository** (`extends JpaRepository`): persistence only. Derived query methods or `@Query`; no business logic.

Use constructor injection everywhere. Never use field injection (`@Autowired` on fields).

```java
@RestController
@RequestMapping("/api/v1/users")
public class UserController {
    private final UserService userService;

    public UserController(UserService userService) {
        this.userService = userService;
    }
    ...
}
```

## 6. Pagination on every list endpoint

Never return an unbounded collection. Every list/search endpoint accepts pagination parameters.

- Default page size: **20**. Enforce a maximum of **100** in the service layer.
- Accept `page`, `size`, `sort`, `direction` request params, or a `Pageable` directly.
- Return Spring's `Page<T>` mapped to DTOs, or the project's `PaginatedResponse<T>` wrapper if one exists — follow whichever the codebase already uses.

```java
@GetMapping
public ResponseEntity<Page<UserSummaryResponse>> getUsers(
        @RequestParam(defaultValue = "0") int page,
        @RequestParam(defaultValue = "20") int size,
        @RequestParam(defaultValue = "id") String sort) {
    Pageable pageable = PageRequest.of(page, Math.min(size, 100), Sort.by(sort));
    return ResponseEntity.ok(userService.getUsers(pageable));
}
```

## 7. Global exception handling

All error responses come from one `@RestControllerAdvice` class (prefer it over `@ControllerAdvice` — it implies `@ResponseBody`). Controllers and services never build error `ResponseEntity`s inline; they throw domain exceptions.

Standard error body — use this exact shape across the whole API:

```java
public record ErrorResponse(String code, String message, List<String> details) {
    public ErrorResponse(String code, String message) {
        this(code, message, null);
    }
}
```

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ErrorResponse> handleValidation(MethodArgumentNotValidException ex) {
        List<String> details = ex.getBindingResult().getFieldErrors().stream()
            .map(e -> e.getField() + ": " + e.getDefaultMessage())
            .toList();
        return ResponseEntity.badRequest()
            .body(new ErrorResponse("VALIDATION_ERROR", "Validation failed", details));
    }

    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(ResourceNotFoundException ex) {
        return ResponseEntity.status(HttpStatus.NOT_FOUND)
            .body(new ErrorResponse("NOT_FOUND", ex.getMessage()));
    }

    @ExceptionHandler(DuplicateResourceException.class)
    public ResponseEntity<ErrorResponse> handleConflict(DuplicateResourceException ex) {
        return ResponseEntity.status(HttpStatus.CONFLICT)
            .body(new ErrorResponse("CONFLICT", ex.getMessage()));
    }

    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleGeneric(Exception ex) {
        // Log the real exception; never expose it to the client
        return ResponseEntity.internalServerError()
            .body(new ErrorResponse("INTERNAL_ERROR", "An unexpected error occurred"));
    }
}
```

Define domain exceptions as unchecked (`extends RuntimeException`) with descriptive names. Reuse existing ones in the codebase before creating new ones.

## 8. Security

When the project has (or needs) authentication:

- Stateless JWT: `SessionCreationPolicy.STATELESS`, a JWT filter registered before `UsernamePasswordAuthenticationFilter`.
- Hash passwords with `BCryptPasswordEncoder`. Never store or log plaintext passwords.
- Enable `@EnableMethodSecurity` and protect sensitive operations with `@PreAuthorize` (e.g., `hasRole('ADMIN')` on destructive admin endpoints) rather than relying only on URL-pattern rules.
- `permitAll()` only for auth endpoints (login/register), explicitly public routes, and Swagger paths (`/swagger-ui/**`, `/v3/api-docs/**`).
- CSRF may be disabled only for stateless token-based APIs — leave a comment explaining why when doing it; it's a deliberate tradeoff, not a default.

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http.authorizeHttpRequests(auth -> auth
            .requestMatchers("/api/v1/admin/**").hasRole("ADMIN")
            .anyRequest().authenticated());
        return http.build();
    }
}
```

Do not scaffold a full security setup uninvited into a project that currently has none — flag that security is missing and ask, unless explicitly requested.

## 9. API versioning

Version via the URL path: `/api/v1/...`. Every new controller gets a versioned base path from day one — retrofitting versioning later is a breaking change.

- Do not use header-based or content-negotiation versioning unless the codebase already does.
- Breaking changes (removed/renamed fields, changed semantics) go in a new version (`/api/v2/...`) with a separate controller and DTOs; keep v1 alive until consumers migrate.
- Additive changes (new optional fields, new endpoints) do not need a new version.

```java
@RestController @RequestMapping("/api/v1/users") public class UserControllerV1 {}
@RestController @RequestMapping("/api/v2/users") public class UserControllerV2 {}
```

## 10. OpenAPI documentation

Every API module includes springdoc. Use the latest stable version — check https://central.sonatype.com/artifact/org.springdoc/springdoc-openapi-starter-webmvc-ui rather than hardcoding an old one:

```xml
<dependency>
    <groupId>org.springdoc</groupId>
    <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
    <version><!-- latest stable --></version>
</dependency>
```

- Annotate controllers with `@Tag`, endpoints with `@Operation` (summary + description), and non-obvious parameters with `@Parameter`.
- Document error responses with `@ApiResponse` referencing the shared `ErrorResponse` schema.
- Verify docs are reachable at `/swagger-ui.html` and `/v3/api-docs`.

Skip annotation-heavy documentation only if the user explicitly asks for minimal code.

## 11. Rate limiting

For public-facing APIs, add request throttling via a `HandlerInterceptor` (backed by e.g. Bucket4j). Return **429** using the shared `ErrorResponse` shape — not a plain-text body — and include a `Retry-After` header so well-behaved clients know when to come back:

```java
@Component
public class RateLimitInterceptor implements HandlerInterceptor {

    private final RateLimiter rateLimiter;
    private final ObjectMapper objectMapper;

    public RateLimitInterceptor(RateLimiter rateLimiter, ObjectMapper objectMapper) {
        this.rateLimiter = rateLimiter;
        this.objectMapper = objectMapper;
    }

    @Override
    public boolean preHandle(HttpServletRequest request,
                             HttpServletResponse response,
                             Object handler) throws Exception {
        String clientId = getClientId(request); // e.g., API key, user id, or client IP
        if (!rateLimiter.tryAcquire(clientId)) {
            response.setStatus(HttpStatus.TOO_MANY_REQUESTS.value());
            response.setContentType(MediaType.APPLICATION_JSON_VALUE);
            response.getWriter().write(objectMapper.writeValueAsString(
                new ErrorResponse("RATE_LIMIT_EXCEEDED", "Too many requests")));
            return false;
        }
        return true;
    }
}
```

Register it in a `WebMvcConfigurer` via `addInterceptors`, scoped to the API paths that need protection.

## 12. Idempotency for mutating endpoints

Networks are unreliable — a request can fail before it reaches the server, mid-processing, or after the server finished but before the response gets back. A naive client retry can then run the operation twice. This matters most for endpoints with a real side effect that must not double-fire: payments, order creation, sending an email/notification.

- Accept an idempotency key from the client, typically an `Idempotency-Key` header, on mutating (POST) requests.
- Persist the key server-side alongside the eventual result of that operation.
- On a repeat request with the same key, return the cached result instead of re-running the side effect.
- If the server started processing but failed mid-operation, and that failed attempt was rolled back (e.g. via an ACID transaction), a retry with the same key is safe to run fresh.

```
-H "Idempotency-Key: AGJ6FJMkGQIpHUTX"
```

## 13. Resilience when calling external APIs

When your API depends on another service, design for that dependency failing or throttling you:

- **Exponential backoff with jitter** — wait `2^n` seconds before retry n, plus a random jitter component, so retries spread out instead of arriving in a synchronized burst.
- **Respect the `Retry-After` header** — if the server sends one (common on 429/503 responses), use that value instead of guessing a backoff interval.
- **Client-side rate limiting** — cap your own outgoing request rate with a token bucket (refills at a fixed rate, allows short bursts up to bucket size) or leaky bucket (processes at a fixed rate regardless of burst size) so you avoid getting throttled in the first place.
- **Circuit breaker** — track the failure rate to the external dependency; once it crosses a threshold, trip the breaker so further calls fail fast and serve a fallback (cached value, default, degraded response) instead of a wasted round-trip. After a cooldown, allow a trial request through to check recovery (half-open state). In Spring, this is typically Resilience4j's `@CircuitBreaker`.

If backoff alone doesn't resolve it and calls keep failing, treat it as a durability problem, not just a retry problem:

1. Persist the intent to a durable queue or an **outbox table** (written in the same DB transaction as the business write) so the pending work survives a process restart or crash.
2. Retry it later on a scheduler, decoupled from the original request thread.
3. Move anything that never succeeds after N attempts to a **dead letter queue** so it stops retrying forever and stays visible for manual handling.
4. Alert and dashboard on the failure rate so the problem surfaces immediately, not from a customer complaint days later.

Note the idempotency key requirement from §12 still applies to any retried request in this flow.

## 14. Performance

Work through each layer the request passes through rather than reaching for a single remembered trick:

- **Data access**: index the columns actually filtered/sorted/joined on (check the query plan, don't guess); avoid the N+1 problem (JPA lazy loading is the classic cause — fix with a fetch join, `@EntityGraph`, or batch fetching); reuse pooled DB connections (Spring Boot defaults to HikariCP) rather than opening one per request.
- **Response shaping**: paginate every list endpoint (§6); project only the fields needed rather than serializing whole entities (a slimmer response DTO for list views vs. the full detail DTO for a single-resource view, §3); enable response compression (gzip) for larger JSON payloads.
- **Caching**: cache read-heavy, rarely-changing data with `@Cacheable`, always paired with an explicit invalidation strategy — `@CacheEvict` on the write path, or a TTL if staleness is acceptable. A cache with no invalidation plan is a bug waiting to happen.
- **Offload slow work**: if part of the request is genuinely slow (a report, an external call, an email), don't block the caller's thread — return `202 Accepted` and deliver the result asynchronously via callback, webhook, or polling.
- **Measure against p95/p99 latency, not the average.** Averages hide the slow tail that actually hurts users; naming this metric is what confirms whether the optimizations above are actually working for the requests that matter.

```java
@Service
@Transactional
public class UserService {

    @Cacheable(value = "users", key = "#id")
    @Transactional(readOnly = true)
    public UserResponse getUserById(Long id) { ... }

    @CacheEvict(value = "users", key = "#id")
    public UserResponse updateUser(Long id, UpdateUserRequest request) { ... }
}
```

## 15. Request/response logging

Add a `OncePerRequestFilter` logging method, URI, status, duration, and a correlation ID (from an `X-Correlation-Id` header, or generated). Never log request bodies, credentials, or `Authorization` headers.

```java
@Component
public class RequestResponseLoggingFilter extends OncePerRequestFilter {

    private static final Logger logger = LoggerFactory.getLogger(RequestResponseLoggingFilter.class);

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain filterChain) throws ServletException, IOException {
        String correlationId = Optional.ofNullable(request.getHeader("X-Correlation-Id"))
            .orElse(UUID.randomUUID().toString());
        MDC.put("correlationId", correlationId);
        long startTime = System.currentTimeMillis();

        logger.info("Request: {} {}", request.getMethod(), request.getRequestURI());
        try {
            filterChain.doFilter(request, response);
        } finally {
            long duration = System.currentTimeMillis() - startTime;
            logger.info("Response: {} {} - {} ({}ms)",
                request.getMethod(), request.getRequestURI(), response.getStatus(), duration);
            MDC.remove("correlationId");
        }
    }
}
```

---

## Verification Checklist

Before declaring any endpoint work complete, verify all of these. When reviewing existing code, report each violation with file/line and fix it:

- [ ] URLs use plural nouns, no verbs; base path on the class
- [ ] Correct success status per operation (201 create, 204 delete)
- [ ] No entity appears in any controller signature — DTOs only
- [ ] Every request body DTO has Bean Validation annotations and `@Valid`
- [ ] Controller has no business logic or repository access; constructor injection only
- [ ] Every list endpoint is paginated with an enforced max page size
- [ ] Errors flow through `@RestControllerAdvice` with the shared `ErrorResponse` shape
- [ ] No stack traces, entity internals, or credentials in responses or logs
- [ ] Base path is versioned (`/api/v1/...`)
- [ ] OpenAPI annotations present; springdoc dependency at latest stable
- [ ] Mutating endpoints with real side effects (payments, orders, notifications) accept an idempotency key
- [ ] Calls to external APIs have backoff+jitter and a circuit breaker, not a bare retry loop
- [ ] Caches have an explicit eviction strategy — never `@Cacheable` without `@CacheEvict` or a TTL
