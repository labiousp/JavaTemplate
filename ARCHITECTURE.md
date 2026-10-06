# Architecture: Java Layered Application Template

Version 1.0. This document is the contract for every project cloned from the template. If code and this document disagree, fix one of them the same day.

---

## 1. Purpose and principles

The template exists so that any new application starts from a known-good structure and the only real thinking is about the domain.

1. **Separation of concerns is strict.** Each layer has one job.
2. **Dependencies point inward.** Outer layers know about inner layers, never the reverse.
3. **Business rules live in one place** (services) and are deterministic.
4. **Everything external sits behind an interface** (database, email, payments, clock).
5. **One vertical slice at a time.** Finish an entity end to end before starting the next.
6. **Composition happens in one place** (`Main` / the Spring configuration).
7. **Zero cost to build, run and test.** Paid services are optional adapters only (see section 13).
8. **The core is complete without extras.** Optional features (AI, premium tiers) are extensions that depend on the core; the core never depends on them (see section 16).

---

## 2. Layer map

```
   ┌────────────────────────────────────────┐
   │  ui (cli, api)                         │  input/output only
   └───────────────────┬────────────────────┘
                       │ calls
   ┌───────────────────▼────────────────────┐
   │  service                               │  rules, validation, transactions
   └──────┬─────────────────────────┬───────┘
          │ uses                    │ uses
   ┌──────▼─────────┐        ┌──────▼─────────┐
   │ dao (ports)    │        │ other ports    │  interfaces owned by the app
   └──────┬─────────┘        │ (email, clock) │
          │ implemented by   └──────┬─────────┘
   ┌──────▼─────────┐        ┌──────▼─────────┐
   │ JDBC / JPA     │        │ SMTP / clock   │  infrastructure adapters
   └────────────────┘        └────────────────┘

   model: plain data, used by every layer, depends on nothing
   core:  db, config, error, shared by everyone
```

### Dependency rule

| Layer | May depend on | Must never depend on |
|---|---|---|
| `model` | nothing (JDK only) | everything else |
| `core` | `model` (errors only if needed) | `dao`, `service`, `ui` |
| `dao` | `model`, `core.db` | `service`, `ui` |
| `service` | `model`, `dao` interfaces, other ports, `core.error` | `ui`, JDBC types (`ResultSet`, `Connection` details) |
| `ui` | `service`, `model` | `dao`, SQL |
| `Main` | everything (wiring only) | no business logic |

---

## 3. Package layout

```
src/main/java/com/yourname/app/
├── Main.java                      composition root
├── core/
│   ├── config/AppConfig.java      reads config.properties / env vars
│   ├── db/Database.java           HikariCP pool, Flyway migrate, transactions
│   ├── db/TransactionManager.java what services depend on (testable)
│   ├── db/SqlFunction.java        lambda type that may throw SQLException
│   ├── error/                     AppException, ValidationException, NotFoundException,
│   │                              ConflictException, InfrastructureException,
│   │                              DataAccessException
│   └── feature/                   Feature, Entitlements, DefaultEntitlements
├── model/                         records / POJOs, one per concept
├── dao/
│   ├── BaseDao.java               generic CRUD contract
│   ├── RowMapper.java             ResultSet -> model
│   ├── StatementBinder.java       sets ? parameters
│   ├── StudentDao.java            interface
│   └── jdbc/
│       ├── JdbcDao.java           shared queryList / queryOne / execute helpers
│       └── JdbcStudentDao.java    implementation (SQL only)
├── service/
│   └── StudentService.java
├── infrastructure/                adapters for core ports (SMTP, clock...)
└── ui/
    ├── cli/StudentMenu.java
    └── api/StudentController.java (added when exposed over REST)

src/main/resources/
├── config.properties
└── db/migration/V1__init.sql      Flyway migrations, never edited once applied

src/test/java/...                  mirrors main package structure

extensions/                        OPTIONAL separate modules (section 16)
└── app-ai/                        AI add-on, built only when needed
```

---

## 4. Layer contracts

### 4.1 `model`
- Plain data: Java `record`s where immutable, POJOs where mutation is needed.
- No SQL, no logic beyond trivial derived getters, no framework annotations (until a framework requires them).
- Identifiers use a wrapper type or `long`; use `Optional` only for return values, never fields.
- Separate **entities** (stored) from **DTOs / request objects** (what the UI sends in) when they differ.

### 4.2 `dao`
- **SQL only.** No business rules, no validation, no "if student is inactive then...".
- Returns models or `Optional<Model>`; never returns `ResultSet`.
- Uses `PreparedStatement` for every query. String concatenated SQL is a defect.
- One DAO per aggregate or table family.
- Methods are named for what they fetch: `findById`, `findByClassId`, `insert`, `update`, `deactivate`.
- Throws a wrapped `DataAccessException` (unchecked), never raw `SQLException`.
- Receives its connection from `Database`; participates in a transaction when one is active.

```java
public interface BaseDao<T, ID> {
    Optional<T> findById(ID id);
    List<T> findAll();
    T insert(T entity);
    void update(T entity);
    void deleteById(ID id);
}
```

### 4.3 `service`
- Owns **validation, rules, orchestration and transaction boundaries**.
- Depends on DAO **interfaces**, so tests can substitute fakes.
- Throws domain exceptions (`ValidationException`, `NotFoundException`, `ConflictException`).
- Never prints, never reads input, never imports `java.sql.*`.
- One public method = one use case, named as a verb phrase: `enrollStudentInClass`, `recordAttendance`, `calculateFinalAverage`.

```java
public Student deactivateStudent(long id) {
    Student s = studentDao.findById(id)
        .orElseThrow(() -> new NotFoundException("Student " + id));
    if (!s.active()) throw new ConflictException("Already inactive");
    return tx.inTransaction(() -> {   // tx is a TransactionManager (Database in production)
        studentDao.deactivate(id);
        attendanceDao.closeOpenRecords(id);
        return studentDao.findById(id).orElseThrow();
    });
}
```

### 4.4 `ui`
- **Input and output only.** Parse input, call one service method, display the result or the error.
- Catches `AppException` and shows a friendly message; unexpected exceptions are logged and shown generically.
- No rules (for example, "mark must be 0 to 100" belongs in the service, not the menu).
- CLI and REST controllers are interchangeable clients of the same services.

### 4.5 `core`
- `Database`: HikariCP pool, `migrate()` (Flyway), `withConnection(...)` for DAOs and `inTransaction(...)` for services. DAOs inside a transaction join it automatically (same thread).
- `TransactionManager`: the interface services depend on; `TransactionManager.passThrough()` makes unit tests database-free.
- `feature`: `Entitlements` is the only hook extensions use (section 16).
- `AppConfig`: all settings come from `config.properties` or environment variables. Secrets are never committed.
- `error`: the exception hierarchy below.

### 4.6 `Main` (composition root)
The only place that uses `new` on DAOs, services and adapters.

```java
var config   = AppConfig.load();
try (var db = Database.from(config)) {
    db.migrate();
    var students = new JdbcStudentDao(db);
    var service  = new StudentService(students, db, Clock.systemDefaultZone());
    new StudentMenu(new Scanner(System.in), System.out, service).run();
}
```

---

## 5. Error handling

```
AppException (unchecked, base)
├── ValidationException     bad input, rule violated        -> 400
├── NotFoundException       entity does not exist           -> 404
├── ConflictException       state conflict, duplicate       -> 409
└── InfrastructureException DB or external service failed   -> 500 / 503
```

- Convert low-level exceptions at the boundary (DAO wraps `SQLException`; an HTTP adapter wraps network failures).
- Messages for users are separate from log detail.
- Never swallow an exception silently.

---

## 6. Transactions

- Boundaries are declared **in the service layer**, never in DAOs or UI.
- A use case that changes more than one table runs in one transaction.
- Read-only queries do not need a transaction.
- Many-to-many changes (for example, assigning subjects to a student) are a single transaction.

---

## 7. Database conventions

- Table names: plural snake_case (`students`, `student_subjects`).
- Primary key: `id`. Foreign keys: `<table_singular>_id`.
- Join table for many-to-many: composite key of the two foreign keys.
- Every table has `created_at`; add `active` or `deleted_at` for soft deactivation instead of hard delete when history matters.
- Constraints (foreign keys, `NOT NULL`, `UNIQUE`) live in the schema, not only in code.
- Schema changes go in a new numbered migration file, never an edit to an old one.
- Index every foreign key and every column used in frequent lookups.

---

## 8. External services as ports

Pattern for anything outside the process (email, clock, payments, file storage):

1. Define an interface in `service/` expressed in **domain language**.
2. Implement it in `infrastructure/`.
3. Provide a **fake** implementation for tests.

```java
public interface EmailSender {
    void send(String to, String subject, String body);
}
```

Core ports must have a free default (for example, log-to-console or local SMTP). Features that need a paid or heavy dependency are **extensions**, not core: see section 16.

---

## 9. Testing strategy

| Level | Target | Tools | Speed |
|---|---|---|---|
| Unit | services with fake DAOs and fake ports | JUnit 5, Mockito or hand-written fakes | milliseconds |
| Integration | DAOs against a real database | Testcontainers | seconds |
| Contract/API | controllers | Spring `MockMvc` or HTTP client | seconds |

Minimum bar: every service method has tests for the success path, each validation failure and each not-found case.

---

## 10. Naming and style

- Classes: nouns (`StudentService`), interfaces without an `I` prefix, implementations named by technology (`JdbcStudentDao`).
- Methods: verbs (`findById`, `enrollStudent`).
- Packages: lowercase, by layer first, then feature if the project grows large.
- Constants in `UPPER_SNAKE_CASE`; no magic numbers or strings in logic.
- One class, one reason to change.
- Prefer constructor injection and `final` fields.

---

## 11. Definition of done for a vertical slice

A slice (for example, "Student management") is finished when:

- [ ] Migration exists and applies cleanly to an empty database
- [ ] Model created
- [ ] DAO interface, JDBC implementation and integration test
- [ ] Service with validation, rules and unit tests
- [ ] UI (CLI menu or controller) wired through `Main`
- [ ] Errors surface as friendly messages
- [ ] Manual smoke test passed end to end
- [ ] Committed with a clear message

---

## 12. Procedure for starting any new application

1. Clone the template; rename the base package.
2. Write the **nouns and verbs** list (entities and use cases). 15 minutes.
3. Draw the schema and mark relationships (1-to-many, many-to-many).
4. Write `V1__init.sql`.
5. Pick the first slice and complete it using the definition of done.
6. Repeat per slice.
7. Add REST. Add extensions (section 16) only after the core slices work and there is a real need.
8. Record decisions in the log below.

---

## 13. Technology baseline

**Hard constraint: every tool, library and service in the default stack must be free.** Anything paid is optional, lives behind a port, and is never required to run or test the app.

| Concern | Choice (free) | Notes |
|---|---|---|
| Language/JDK | OpenJDK 21+ (Temurin) | open source |
| IDE | IntelliJ IDEA Community or VS Code | both free |
| Build | Maven | pick one and keep it for all projects |
| Database | PostgreSQL | free, open source; pgvector for vector search. MySQL Community is also free |
| Pooling | HikariCP | open source |
| Migrations | Flyway Community | the free edition covers everything in this template |
| Testing | JUnit 5, Mockito, Testcontainers | open source; needs a container runtime (Docker Engine, Podman or Colima) |
| Logging | SLF4J + Logback | no `System.out` in non-UI code |
| Web (later) | Spring Boot | open source |
| Extensions | see section 16 | optional, never required to run the core |
| Source control/CI | Git + GitHub (Actions free tier) | check current free-tier limits for private repos |
| Hosting | Run locally first; free-tier hosts as a bonus | free tiers change often, so never depend on one |
| Diagrams | Mermaid or plain ASCII | text-based, lives in the repo |

---

## 14. Decision log (template)

Copy this block for each significant decision.

```
### ADR-001: <title>
Date:
Status: proposed | accepted | superseded
Context: what problem forced a decision
Decision: what we chose
Alternatives considered:
Consequences: what becomes easier or harder
```

---

## 15. Anti-patterns (reject in review)

- SQL or `ResultSet` outside `dao`
- Business rules in `ui` or `dao`
- `new SomeService()` anywhere except the composition root
- Services that read from `System.in` or print
- Catching `Exception` and doing nothing
- Hardcoded credentials or API keys
- Editing an already-applied migration
- Core code importing an extension module, or any core path that fails when an extension is missing
- Extension output (such as AI) written to the database without validation
- Big-bang development: building every layer for every entity before anything runs

---

## 16. Optional extension: AI (premium tier)

AI is **not part of the core template**. It is a non-blocking add-on, built only when there is a real user need, and designed to be offered to premium users.

### 16.1 Isolation rules
- Lives in its own module (`extensions/app-ai`). It depends on the core; the core has no reference to it.
- The application builds, runs and passes all tests with the extension absent.
- `Main` registers the extension only if it is on the classpath **and** its feature flag is enabled.
- AI calls never sit on the critical path of a core use case. If the model is slow, down or wrong, the core feature still completes (timeouts, fallback, or run after the transaction commits).

### 16.2 The only core hook: feature flags and entitlements
The core contains one tiny, free, generic mechanism so extensions can be gated without core changes:

```java
public enum Feature { ADVANCED_REPORTS, AI_ASSIST }  // examples; add as extensions appear

public interface Entitlements {
    boolean isEnabled(Feature feature, long userId);
}
```

- Default implementation: everything off, or a simple tier column on the user.
- Premium gating is a check in the **service** (or the extension's entry point), never in the UI alone.
- Billing and payments are a separate extension; payment providers charge fees, so treat that as optional cost.

### 16.3 Structure inside the extension
```
extensions/app-ai/
├── AiAssistant.java            port (interface, domain language)
├── OllamaAiAssistant.java      free default adapter (local model)
├── HostedAiAssistant.java      optional paid provider adapter
├── FakeAiAssistant.java        for tests
├── resources/prompts/          versioned prompt files
└── evals/                      saved inputs and quality checks
```

### 16.4 Cost model (keeps the free constraint honest)
- The core stays free to build and run.
- A local model (Ollama) keeps the extension free but depends on the host machine.
- A hosted model costs money per use. Offer it only behind the premium tier, and price the tier to cover usage. Never let free users trigger paid calls.

### 16.5 Rules for AI output
- AI **suggests**; services **validate and decide**. Model output never writes data without validation.
- Request structured output, parse it into a record, validate it, handle failure and retry.
- Log prompt version, latency, token usage and cost.
- Cache and rate-limit calls (per user, especially on the premium tier).
- Decide deliberately what personal data may leave the system.
- Rerun the eval set whenever a prompt or model changes.
- Embeddings and retrieval (RAG) use PostgreSQL with pgvector and local embeddings by default.

### 16.6 When to build it
Only after the core slices are done and a user has a problem the AI actually solves. Suggested order inside the extension: structured-output feature, then tool calling, then retrieval.
