# Spring Boot API Best Practices Agent Skill

A specialized, model-agnostic coding agent skill that enforces REST API best practices for Java Spring Boot projects — resource naming, HTTP status codes, DTOs, validation, layering, pagination, global exception handling, security, versioning, OpenAPI docs, rate limiting, idempotency, resilience, and performance.

Designed to work across **any LLM agent or coding assistant** (Cursor, Antigravity / Gemini CLI, Claude Code, GitHub Copilot, Windsurf, Aider, ChatGPT, and others). Nothing in the skill is version-specific — it applies equally to Spring Boot 3.x and 4.x projects.

---

## ✨ What it enforces

- **RESTful resource naming** — plural nouns, no verbs in URLs, versioned base paths, bounded sub-resource nesting.
- **Correct HTTP status codes** — the right success/error code per operation, including the idempotency distinction between PUT and PATCH.
- **DTOs, never entities** — controllers never accept or return JPA entities directly.
- **Bean Validation** on every request body, with human-readable messages.
- **Controller → Service → Repository layering** with constructor injection only.
- **Pagination** on every list endpoint, with an enforced max page size.
- **Centralized exception handling** via `@RestControllerAdvice` and one shared `ErrorResponse` shape.
- **Security** — stateless JWT, `BCryptPasswordEncoder`, `@PreAuthorize` on sensitive operations.
- **API versioning** via the URL path, with a clear additive-vs-breaking-change rule.
- **OpenAPI/Swagger documentation** on every endpoint.
- **Rate limiting** (token bucket, `429` + `Retry-After`) for public-facing APIs.
- **Idempotency keys** on mutating endpoints with real side effects (payments, orders, notifications).
- **Resilience** when calling external APIs — backoff + jitter, circuit breakers, and the outbox/DLQ pattern when retries alone aren't enough.
- **Performance** — indexing, N+1 avoidance, caching with an invalidation strategy, async offload for slow work, and measuring p95/p99 latency.

Every point above closes with a verification checklist the assistant is expected to run against any endpoint it touches.

---

## 🚀 How to Use Across Any LLM Agent

Install this as a regular skill in your agent environment, or provide [`api-best-practices/SKILL.md`](https://github.com/hi-rullah/api-best-practices-spring-boot-skill/blob/main/api-best-practices/SKILL.md) directly as context to any AI assistant.

### IDE Assistants (Cursor, Windsurf, Copilot)

- **Cursor / Windsurf:** Add the contents of [`api-best-practices/SKILL.md`](https://github.com/hi-rullah/api-best-practices-spring-boot-skill/blob/main/api-best-practices/SKILL.md) to your `.cursorrules` or `.windsurfrules` file.
- **GitHub Copilot:** Add it to `.github/copilot-instructions.md`.

### Web & Chat Assistants (Claude, ChatGPT, Aider)

1. Open [api-best-practices/SKILL.md](https://github.com/hi-rullah/api-best-practices-spring-boot-skill/blob/main/api-best-practices/SKILL.md).
2. Attach or paste the content into your prompt.
3. Instruct the assistant:

> *"Follow the api-best-practices skill defined in this file for my Spring Boot project."*

### Claude (Skills-native)

Drop the `api-best-practices/` folder into your skills directory. Claude will consult it automatically whenever you create or modify a Spring Boot controller, endpoint, DTO, or service class.

---

## 🛠 Prompts & Commands

Once integrated, prompt your assistant naturally:

- **Build a new endpoint:**

> "Add a POST endpoint to create an order, following api-best-practices."

- **Review existing code:**

> "Review UserController against api-best-practices and fix any violations."

- **Targeted topics:**

> "How should this endpoint handle retries against a flaky payment provider?"
> "Add an idempotency key to this checkout endpoint."

---

## 📋 To-Do / Roadmap

- [x] Core REST conventions: naming, status codes, DTOs, validation, layering, pagination, exception handling.
- [x] Security, versioning, OpenAPI docs.
- [x] Rate limiting, idempotency, resilience (retries/circuit breakers/outbox), performance levers.
- [ ] Spring Boot 4-specific notes (called out separately if/when real differences from 3.x turn up — the core conventions above are intentionally version-agnostic).
- [ ] Example project / reference controller demonstrating the full checklist end-to-end.

---

## 📜 License

MIT
