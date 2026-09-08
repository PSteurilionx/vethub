# Agent Guide

## Repository layout

- `server/` is the Spring Boot backend; `client/` is the SvelteKit frontend. They have separate build tools and must be checked independently.
- `server/src/main/java/dev/ilionx/workshop/Application.java` is the backend entry point. Backend API paths are under `/v1`; the normal servlet context adds `/api`.
- Backend domain code is grouped by feature under `server/src/main/java/dev/ilionx/workshop/api/` with controller, service, repository, model, and mapper layers.
- `server/src/main/resources/db/changelog/` is the Liquibase source of truth for schema and seed data. Do not use Hibernate schema generation to change the database.
- OpenAPI-generated frontend types live in tracked files under `client/src/lib/types/`; regenerate them when backend API contracts change.

## Toolchain and commands

- `mise.toml` pins Java `temurin-25`, Bun `1.3.0`, and Node `22.20.0`; use these versions when available. The backend wrapper uses Gradle `9.3.1`.
- Backend commands run from `server/`: `./gradlew test`, `./gradlew check`, `./gradlew build`, and `./gradlew bootRun`.
- Use focused backend tests with `./gradlew test --tests '*PetServiceTest'` or `./gradlew test --tests '*PetControllerTest'`.
- `./gradlew check` includes formatting and quality checks. Java compilation depends on `spotlessApply`, so formatting can modify Java files during a build.
- Frontend commands run from `client/`: `bun run check`, `bun run build`, `bun run dev`, and `bun run preview`.
- There is no frontend test script in `client/package.json`; `bun run check` is the available static/type validation command.
- Use the Gradle and Bun lockfiles already present; do not introduce a third package manager or regenerate a different lockfile without a concrete need.

## Runtime and data

- Development uses an in-memory H2 database. `mise.toml` selects the `dev` Spring profile, which drops and reseeds the database on backend startup using Liquibase test-context seed data.
- The development backend defaults to port `8080`, with the API reachable under `http://localhost:8080/api`.
- The frontend defaults to `http://localhost:8080/api`; override it with `VITE_SERVER_BASE_URL`. Demo Basic Auth defaults are `user`/`password`, overridable with `VITE_API_USERNAME` and `VITE_API_PASSWORD`.
- Integration tests use the `test` profile, an isolated H2 database, and Liquibase `tst` context. Tests extending `IntegrationTest` clean database state before and after each test.
- Unit tests extend `UnitTest` and use Mockito without starting Spring or a database. Controller integration tests extend `IntegrationTest` and use `MockMvc` with the real Spring context and repositories.

## Test base classes

- `UnitTest` is the base class for fast, isolated unit tests. Extend it when testing a service, validator, mapper, or other class with dependencies that can be mocked. It enables Mockito through JUnit 5 and does not start Spring or connect to a database.
- `IntegrationTest` is the base class for tests that need the application wiring, HTTP layer, repositories, or the database. Extend it for controller/API tests and persistence-backed behavior. It uses the real Spring test context and `MockMvc`, runs with the isolated H2 test database and Liquibase `tst` data, and cleans database state before and after each test.
- Prefer `UnitTest` unless the behavior under test specifically depends on Spring configuration, request handling, repository persistence, or interactions between multiple application layers.

## API contract workflow

- After changing backend controllers or DTOs, run `cd client && bun run sync:api` with the backend available; this downloads the OpenAPI document and regenerates `client/src/lib/types/api.d.ts`.
- `client/src/lib/types/openapi.json` and `client/src/lib/types/api.d.ts` are tracked generated artifacts. Review generated diffs together with the API change.
- The sync script also manages a temporary backend process and Gradle daemons; do not run it concurrently with another backend or Gradle operation.

## Style and verification

- Java formatting is configured in `server/build.gradle.kts` using the checked-in Spotless styling file at `server/src/quality/config/spotless/styling.xml`.
- Backend quality tools are configured under `server/src/quality/config/`; prefer `./gradlew check` for the final server verification rather than invoking individual tools ad hoc.
- Keep schema changes in a new Liquibase changeset under `server/src/main/resources/db/changelog/changesets/` and ensure the appropriate `prd`/`tst` context behavior is intentional.
