# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project overview

"苍穹外卖" (Sky Take-Out) — a Spring Boot food-delivery / takeout management system. It has two client sides that share one backend:

- **Admin side** (`/admin/**`) — restaurant staff management (employees, dishes, set meals, orders, statistics).
- **User side** (`/user/**`) — WeChat mini-program customers (browsing dishes, shopping cart, ordering, payment).

Only the admin employee login/logout flow is implemented so far; the rest of the controllers/services/mappers are expected to be added over time.

## Build & run

The project is a Maven multi-module build with **no Maven wrapper** — use a system `mvn`. It targets **JDK 25** (IntelliJ SDK `ms-25`; `bytecodeTargetLevel` 25).

```bash
# build everything (from the sky-take-out root)
mvn clean install

# build/run just the runnable server module (pulls in sky-common + sky-pojo)
mvn -pl sky-server -am package
mvn -pl sky-server -am spring-boot:run

# run all tests (currently none)
mvn test
```

The entry point is `sky-server`'s `com.sky.SkyApplication` (`@SpringBootApplication`, `@EnableTransactionManagement`). The server listens on port `8080`.

- API docs (knife4j/Swagger): http://localhost:8080/doc.html

## Module structure

Three modules under a `com.sky` group, all depending on the parent POM (Spring Boot `2.7.3`):

- **`sky-common`** — shared, non-web infrastructure: constants, exceptions, `Result`/`PageResult` wrappers, `@ConfigurationProperties` classes, `BaseContext` (ThreadLocal), and utilities (JWT, Aliyun OSS, WeChat Pay, HTTP client, Jackson mapper).
- **`sky-pojo`** — data model classes only: `entity`, `dto`, `vo`. No logic. `sky-server` depends on both `sky-common` and `sky-pojo`; `sky-pojo` is otherwise standalone.
- **`sky-server`** — the runnable Spring Boot app and the web/MyBatis layers: `controller`, `service`/`service.impl`, `mapper` (with XML in `src/main/resources/mapper/*.xml`), `config`, `interceptor`, `handler`.

## Architecture conventions

- **Layered flow**: `Controller` → `Service` interface → `ServiceImpl` → `Mapper` (MyBatis). Mappers are `@Mapper` interfaces; SQL lives either in `@Select` annotations or in `sky-server/src/main/resources/mapper/*.xml` (configured via `mybatis.mapper-locations: classpath:mapper/*.xml`, camel-case mapping enabled).
- **Unified response envelope**: every controller returns `Result<T>` (from `sky-common`), where `code=1` means success and `code=0` failure. Use `Result.success(data)` / `Result.error(msg)`.
- **Error handling**: business failures throw a `BaseException` subclass (e.g. `AccountNotFoundException`, `PasswordErrorException`, each in `sky-common/.../exception`). A single `@RestControllerAdvice` `GlobalExceptionHandler` in `sky-server` catches `BaseException` and maps it to `Result.error(...)`. Add new business exceptions as `BaseException` subclasses and reference their message from `MessageConstant`.
- **Auth (admin)**: `JwtTokenAdminInterceptor` validates a JWT for every `/admin/**` path except `/admin/employee/login`, returning HTTP 401 on failure. Login (`EmployeeController.login`) issues the JWT with claim `empId` (key in `JwtClaimsConstant.EMP_ID`). JWT secret/TTL/header name are bound to `JwtProperties` from `sky.jwt.*` in `application.yml`.
- **Per-request user context**: `BaseContext` holds the current user id in a `ThreadLocal<Long>` (set in an interceptor, read anywhere, removed at request end).
- **Passwords**: stored/compared as MD5 hex via `DigestUtils.md5DigestAsHex` (see `EmployeeServiceImpl`).

## Configuration

- `sky-server/src/main/resources/application.yml` — active profile is `dev`, plus the `sky.jwt` admin settings and MyBatis/logging config.
- `sky-server/src/main/resources/application-dev.yml` — dev-only `sky.datasource` values (MySQL host/port/database/credentials). The DB is named `sky_take_out`.
- External-service credentials are wired through `@ConfigurationProperties` beans under the `sky.*` prefix: `JwtProperties` (`sky.jwt`), `AliOssProperties` (`sky.alioss`), `WeChatProperties` (`sky.wechat`). Only `sky.jwt` and `sky.datasource` are currently populated in YAML; OSS/WeChat keys are added as those features are built.

## Notable tech / gotchas

- **JDK 25 + Lombok**: the parent POM sets `maven-compiler-plugin` to `3.13.0` with `<proc>full</proc>` because JDK 23+ disables annotation processing by default and Lombok needs it. If a clean checkout fails to resolve Lombok symbols, this is the first thing to check.
- **Druid** connection pool, **PageHelper** pagination, **Redis** (`spring-boot-starter-data-redis`), **WebSocket**, and **knife4j** (Swagger UI) are all on the classpath for later features.
- **LocalDateTime serialization** goes through `sky-common`'s `JacksonObjectMapper`, which formats `LocalDateTime` as `yyyy-MM-dd HH:mm` (note: no seconds).
- **Aliyun OSS** is used for file upload; **WeChat Pay** (`wechatpay-apiv3`) for user-side payment. Utilities are already in `sky-common/utils` (`AliOssUtil`, `WeChatPayUtil`).
- **`spring.main.allow-circular-references: true`** is enabled in `application.yml` — keep this in mind before refactoring bean dependencies.
