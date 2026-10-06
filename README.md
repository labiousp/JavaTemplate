# Java App Template

A free, layered Java starter for building any application quickly.
Strict layers (**model → dao → service → ui**), PostgreSQL, Flyway migrations,
connection pooling, unit and integration tests, and CI. One complete example
slice (**Student**) shows the pattern to copy.

Everything in the default stack is free and open source. Optional extras
(such as AI or premium features) live in `extensions/` and are never required.

## Quick start

Prerequisites: JDK 21+, Maven 3.9+, and a container runtime (Docker Engine,
Podman or Colima) for PostgreSQL and integration tests.

```bash
git clone <your-repo-url> my-app && cd my-app
bash scripts/rename-project.sh com.acme.myapp   # once, right after cloning
docker compose up -d                            # local PostgreSQL
mvn verify                                      # unit + integration tests
mvn compile exec:java                           # run the console app
```

- `mvn test` runs only fast unit tests (no Docker needed).
- Integration tests (`*IT`) start a throwaway PostgreSQL through Testcontainers
  and skip themselves automatically if no container runtime is available.
- No Maven installed? Install it, or run `mvn -N wrapper:wrapper` once to add
  the Maven wrapper to the repo.

## Start a new application from this template

1. On GitHub: **Settings → General → Template repository** (tick it, once).
2. For each new app: **Use this template → Create a new repository**, then clone it.
3. Run `bash scripts/rename-project.sh <your.package> [artifactId]`.
4. Follow `docs/NEW_APP_CHECKLIST.md`: nouns and verbs, schema, one slice at a time.

## Project layout

```
src/main/java/com/yourname/app/
├── Main.java            composition root: the only place that wires objects
├── core/                config, database + transactions, errors, feature flags
├── model/               plain data (records)
├── dao/                 DAO interfaces; dao/jdbc/ = SQL implementations
├── service/             business rules, validation, transactions
├── infrastructure/      adapters for external ports (email, clock...) - empty for now
└── ui/cli/              console UI (add ui/api/ for REST later)
src/main/resources/      config.properties, logback.xml, db/migration/V1__init.sql
src/test/java/           *Test = unit tests, *IT = integration tests
extensions/              optional modules (AI, premium); not part of the core build
docs/                    new-app checklist and architecture decision records (ADRs)
```

The rules each layer must follow are in **[ARCHITECTURE.md](ARCHITECTURE.md)**.
The short version:

| Layer | Does | Never does |
|---|---|---|
| `ui` | read input, call one service method, print result | rules, SQL |
| `service` | validation, rules, transactions | SQL, printing, reading input |
| `dao` | SQL only, returns models | rules, validation |
| `model` | plain data | anything else |
| `Main` | wiring | business logic |

## Add a new slice (for example `Course`)

Copy the Student files and rename. Each slice has the same seven parts:

1. `db/migration/V2__create_courses.sql` (a new file; never edit an applied one)
2. `model/Course.java`
3. `dao/CourseDao.java` (extends `BaseDao<Course, Long>`) and `dao/jdbc/JdbcCourseDao.java` (extends `JdbcDao`)
4. `service/CourseService.java` (validation and rules; depends on `CourseDao` and `TransactionManager`)
5. `ui/cli/CourseMenu.java`
6. Wire it in `Main.java`
7. Tests: `CourseServiceTest` with a fake DAO, `JdbcCourseDaoIT` against PostgreSQL

## Configuration

Defaults are in `src/main/resources/config.properties`. Any key can be
overridden with an environment variable (`db.url` becomes `DB_URL`).
Never commit secrets; `.env` is git-ignored (Docker Compose reads it).

## Tech stack (all free)

Java 21, Maven, PostgreSQL 16, HikariCP, Flyway Community, JUnit 5,
Testcontainers, SLF4J + Logback, GitHub Actions. Dependency versions are
pinned in `pom.xml`; update them with `mvn versions:display-dependency-updates`.

## Extensions

AI and other optional or premium features are separate modules under
`extensions/`, gated by `core.feature.Entitlements`, and never required to
build or run the core. See ARCHITECTURE.md section 16.

## License

MIT. Change the name in `LICENSE`.
