---
name: asp-net-core-architecture-default
description: Default architecture guidance for designing, reviewing, generating, or refactoring corporate backend applications with ASP.NET Core, C#, and .NET. Use for project structure, dependencies, DI, configuration, logging, package selection, Mediator handlers, FluentResults, EF Core persistence, pipeline behaviors, validation, middleware, APIs, API versioning, OpenAPI, XML comments, or tests for enterprise ASP.NET Core backends. Prefer Layered or Clean Architecture and do not default to Vertical Slice Architecture.
---

# ASP.NET Core Architecture Default

Use this skill to design, review, generate, or refactor corporate ASP.NET Core backend applications. Prefer Layered / Clean Architecture. Do not use Vertical Slice Architecture by default.

## Core Principles

1. Prefer consistency over perfection.
2. Prefer reuse over rewriting.
3. Prefer the simplest correct solution.
4. Prefer official Microsoft guidance and latest stable APIs.
5. Keep business logic independent from infrastructure.
6. Keep projects minimal, production-ready, and buildable.
7. Do not silently ignore architecture rules.
8. Explain tradeoffs when a request conflicts with this standard.

Avoid obsolete packages and APIs, including old `Startup.cs`-centric setups, legacy Swagger defaults, `Newtonsoft.Json` for normal API JSON, and outdated dependency injection patterns.

## Existing Code First

When working in an existing solution:

- inspect the existing architecture first;
- reuse existing abstractions;
- reuse existing dependency injection style;
- reuse existing folder structure;
- reuse existing naming conventions;
- reuse existing logging abstractions;
- reuse existing repository implementations;
- reuse existing testing conventions;
- extend existing code instead of creating parallel implementations;
- never introduce a second solution to a problem that is already solved in the project;
- do not rewrite large parts of the project only to satisfy a preferred style;
- refactor only the parts required by the requested change unless a larger refactoring is explicitly requested.

Before creating a new abstraction, helper, wrapper, extension method, interface, or service, search the project for an existing implementation, reuse it whenever possible, and avoid parallel implementations.

Follow Existing Code First whenever a local project already has a consistent convention.

## Architecture Decision

When several implementations are possible, choose the simplest correct solution, avoid premature abstractions, explain tradeoffs, and explain why the chosen design is preferable.

## Required Response Order

When proposing architecture or code, present:

1. Project structure.
2. Project dependencies.
3. Dependency Injection registrations.
4. Command/query, validator, handler.
5. Pipeline behaviors.
6. Endpoint or controller.
7. Tests.
8. Why the solution does not violate the architecture rules.

## Project Layout

Start from at least four projects:

```text
{ProjectName}.Api
{ProjectName}.App
{ProjectName}.Domain
{ProjectName}.Infrastructure
```

Add only when justified:

```text
{ProjectName}.Tech
{ProjectName}.Contracts
{ProjectName}.Tests.Unit
{ProjectName}.Tests.Integration
```

Prefer `.slnx` over `.sln` for new .NET solutions when supported by the SDK. Organize production projects under `src/` and test projects under `tests/`.

Remove unused template artifacts such as `*.http`, `WeatherForecast`, default controllers, sample endpoints, sample DTOs, sample configuration, and boilerplate README files.

## Project Dependencies

Enforce these references:

- `{ProjectName}.Api` depends on `{ProjectName}.App` and `{ProjectName}.Infrastructure`.
- `{ProjectName}.App` depends on `{ProjectName}.Domain`.
- `{ProjectName}.Infrastructure` depends on `{ProjectName}.App` and `{ProjectName}.Domain`.
- `{ProjectName}.Domain` depends on no application project.

Keep domain model and business rules independent from ASP.NET Core, EF Core, JSON serialization, external services, and infrastructure frameworks.

## Dependency Injection And Configuration

Put project service registrations in root-level dependency classes:

- `ApiDependencies`
- `AppDependencies`
- `DomainDependencies`
- `InfrastructureDependencies`
- `TechDependencies`, only when `{ProjectName}.Tech` exists

Keep `Program.cs` thin. It should contain only configuration, dependency registration, middleware, endpoint mapping, and application startup. Business logic must never be placed in `Program.cs`.

Prefer constructor injection and primary constructors / concise constructors when suitable. Avoid property injection. Never inject or resolve through `IServiceProvider`; never inject `IServiceScopeFactory` unless implementing infrastructure or framework integration code.

Use a single strongly typed root configuration record, usually `AppConfig`, as the shared source of truth for application configuration. Store it in `{ProjectName}.Domain/Models/Config`, for example `AppConfig.cs`. Place nested configuration records such as `SignalRSettings`, `JaegerSettings`, and `DatabaseSettings` in the same folder.

Register root configuration only in `Program.cs`:

```csharp
builder.Services.Configure<AppConfig>(builder.Configuration);
```

Consume configuration only through `IOptions<AppConfig>`. Read `options.Value` once and reuse the value inside the class. Do not inject `IConfiguration` into handlers, services, repositories, or domain objects.

Do not call `configuration["..."]`, `GetValue<T>()`, or `GetSection()` outside the composition root (`Program.cs` / DI registration). Use `ConfigurationKeyName` when configuration keys differ from property names.

Configuration classes should be immutable record types with `init` properties. Group related settings into nested records. Required configuration must be validated on application startup and fail fast instead of silently using hardcoded defaults. Prefer `ValidateOnStart()` when the project uses options validation.

Example:

```csharp
/// <summary>
/// Конфиг приложения.
/// </summary>
public sealed record AppConfig
{
    /// <summary>
    /// МРУ.
    /// </summary>
    [ConfigurationKeyName("Mru")]
    public string Mru { get; init; } = string.Empty;

    /// <summary>
    /// Настройки Jaeger.
    /// </summary>
    [ConfigurationKeyName("Jaeger")]
    public JaegerSettings JaegerSettings { get; init; } = new();
}
```

## Package Selection

Prefer official Microsoft libraries when they provide equivalent functionality. Avoid adding third-party packages without clear justification. Before introducing a package, verify that .NET does not already provide the feature.

Defaults:

- `System.Text.Json` over `Newtonsoft.Json`.
- Ordinary projections such as `Select(...)` over `AutoMapper` when mapping is simple.
- `Microsoft.Extensions.Resilience` on .NET 8+ over adding `Polly` directly unless Polly-specific APIs are required.
- Built-in ASP.NET Core OpenAPI plus Scalar over Swashbuckle / Swagger by default.

## Mediator And Result Pattern

Use Martin Othamar's `Mediator` package. Do not use `MediatR` by default.

Handlers must not return `IResult`, `Results`, `ActionResult`, status codes, or any HTTP-specific type. Handlers return:

```csharp
FluentResults.Result
FluentResults.Result<T>
```

Use `FluentResults` for expected business outcomes and errors. Do not throw exceptions for normal validation, not-found, conflict, or business-rule outcomes.

Endpoints or controllers call `mediator.Send(...)` and map `Result<T>` to HTTP responses only in `{ProjectName}.Api`. Handlers must not know about HTTP.

Do not use magic strings for error codes. Define shared constants and use them in `FluentResults` metadata and HTTP result mapping.

Prefer Russian user-facing messages in validation errors, `ProblemDetails`, and business errors unless the project explicitly uses English.

## DTOs

DTOs belong to the App layer. API communicates only through DTOs. Domain entities must never be returned directly from API endpoints. Map Domain to DTO inside App handlers or dedicated mappers. Domain entities must not contain serialization concerns.

## JSON

Use `System.Text.Json`. Do not use `Newtonsoft.Json` by default.

- Keep JSON options in `{ProjectName}.Api`.
- Do not introduce JSON serialization concerns into `{ProjectName}.Domain`.
- Prefer `System.Text.Json` source generators when appropriate.
- Do not use dynamic JSON.
- Prefer strongly typed DTOs.

## XML Documentation

XML documentation is mandatory for all public API and must be written in Russian.

Document public classes, records, interfaces, enums, delegates, constructors, methods, and public API properties. Use `<summary>`, `<param>`, `<typeparam>`, `<returns>` when useful, `<exception>` for intentional contract exceptions, and `<inheritdoc />` / `<see cref="TypeName"/>` where appropriate.

Explain business meaning, not implementation mechanics. Do not document every private member; document private members only for important business rules or non-obvious behavior.

Before finishing any task, verify that no public members miss XML documentation.

Example:

```csharp
/// <summary>
/// Получает задачу по уникальному идентификатору.
/// </summary>
/// <param name="id">Уникальный идентификатор задачи.</param>
/// <param name="cancellationToken">Токен отмены операции.</param>
Task<TaskItem?> GetByIdAsync(Guid id, CancellationToken cancellationToken);
```

## Domain Entity Model

Domain entities should be classes, not records, when they have identity, lifecycle, and behavior. Value Objects may be records when immutable.

Use private setters and behavior methods to protect invariants. Do not use record `with` or public `init` setters for mutable entities. Public setters are not allowed on domain entities unless there is a strong reason.

Parameterless constructors for EF Core should be private when possible. Keep EF Core attributes out of Domain entities; use Fluent API configurations in Infrastructure.

Entity state changes should happen through domain methods. Validate invariants inside factory methods and domain methods. Return `FluentResults.Result` / `Result<T>` for expected domain rule failures.

Use UTC `DateTime` values or `TimeProvider` from the application layer. Do not call `DateTime.Now` inside entities. Do not inject services or `ILogger` into entities.

Good entity style:

```csharp
public sealed class TaskItem
{
    private TaskItem()
    {
    }

    private TaskItem(Guid id, string title)
    {
        Id = id;
        Title = title;
    }

    public Guid Id { get; private set; }

    public string Title { get; private set; } = string.Empty;

    public static Result<TaskItem> Create(Guid id, string title)
    {
        ...
        return Result.Ok(new TaskItem(id, title.Trim()));
    }

    public Result Update(...)
    {
        ...
        return Result.Ok();
    }
}
```

Bad entity style: `record` / `init` for entities with lifecycle, because it encourages external state mutation and bypasses domain methods.

## EF Core

Place `DbContext` in `{ProjectName}.Infrastructure`.

Every entity configuration must live in a separate class implementing `IEntityTypeConfiguration<TEntity>`. In `OnModelCreating`, only call:

```csharp
modelBuilder.ApplyConfigurationsFromAssembly(typeof(AppDbContext).Assembly);
```

Keep database mapping separate from domain entities. Avoid EF Core attributes in domain entities when Fluent API can express the mapping.

For read-only queries, use `AsNoTracking()`. Watch for N+1 queries, unnecessary `Include`, premature or unnecessary `ToList`, and multiple enumeration.

## Repositories / Persistence Ports

Use App-layer repository abstractions for persistence. App handlers must not inject EF Core `DbContext` directly.

Rules:

- repository interfaces are declared in `{ProjectName}.App`;
- Infrastructure implements repositories using EF Core;
- `DbContext` stays in `{ProjectName}.Infrastructure`;
- use only the `Repository` suffix; do not mix `Repository` and `Store`;
- do not create generic `IRepository<T>` by default;
- prefer focused repositories named by business capability, such as `ITaskItemRepository`, `IOrderRepository`, `IUserRepository`;
- repositories should represent business capabilities rather than generic CRUD operations;
- expose operations required by use cases, not `IQueryable` by default;
- do not leak EF Core types from repository abstractions;
- repositories do not call `SaveChangesAsync` by default;
- commit is centralized through `IUnitOfWork` and `TransactionBehavior` / `UnitOfWorkBehavior`.

Prefer methods such as `GetByIdAsync(...)`, `ExistsAsync(...)`, `GetActiveOrdersAsync(...)`, and `FindAvailableCellsAsync(...)`. Avoid exposing generic CRUD APIs unless explicitly justified.

## Pipeline Behaviors

Centralize handler-level cross-cutting concerns in Mediator pipeline behaviors. Do not write repetitive `try/catch` blocks in every endpoint, controller, or handler.

Use pipeline behaviors for exception handling, validation, logging, transactions, authorization, metrics, and tracing.

Pipeline behaviors belong to `{ProjectName}.App`, usually in `App/Pipeline`. `ExceptionBehavior`, `ValidationBehavior`, and `TransactionBehavior` / `UnitOfWorkBehavior` must not live in `{ProjectName}.Infrastructure`.

Pipeline behaviors must not depend on EF Core, `DbContext`, infrastructure implementations, or `Microsoft.EntityFrameworkCore`.

Unexpected exceptions should be logged in `ExceptionBehavior`. Business logic should not rely on exceptions. Expected failures should always be represented by `FluentResults`.

## Transactions And SaveChanges

`TransactionBehavior` lives in `{ProjectName}.App/Pipeline` and depends only on App-layer abstractions such as `IUnitOfWork`. It must not depend on EF Core, Infrastructure, `AppDbContext`, or `Microsoft.EntityFrameworkCore`.

Declare `IUnitOfWork` in `{ProjectName}.App`:

```csharp
public interface IUnitOfWork
{
    Task BeginTransactionAsync(CancellationToken cancellationToken);
    Task CommitAsync(CancellationToken cancellationToken);
    Task RollbackAsync(CancellationToken cancellationToken);
}
```

Infrastructure implements `IUnitOfWork` using EF Core `AppDbContext` in `Infrastructure/Persistence`. The implementation uses `AppDbContext.Database.BeginTransactionAsync(...)`, calls `SaveChangesAsync(...)` during commit, and keeps EF Core isolated from App.

Implementation rules:

- guard against repeated `BeginTransactionAsync`; throw a clear `InvalidOperationException` if a transaction is already started;
- `CommitAsync` verifies a transaction started, calls `SaveChangesAsync`, commits, disposes in `finally`, and resets the transaction field to `null`;
- `RollbackAsync` is a safe no-op when no transaction started, otherwise rolls back, disposes in `finally`, and resets the transaction field to `null`;
- do not add `IAsyncDisposable` unless the implementation actually needs it;
- `IUnitOfWork` is only for transaction / commit / rollback orchestration; do not put repositories, query methods, or generic data access methods into it.

Behavior rules:

- write commands implement `ITransactionalCommand<T>`;
- `TransactionBehavior` runs only for `ITransactionalCommand<T>`;
- do not open transactions for read-only queries;
- call the handler;
- commit only on successful `Result` / `Result<T>`;
- rollback on failed `Result` / `Result<T>`;
- rollback and rethrow on unexpected exceptions;
- repositories do not call `SaveChangesAsync` by default;
- handlers do not call `SaveChangesAsync` when `TransactionBehavior` is used;
- avoid double `SaveChangesAsync`.

Command example:

```csharp
public sealed record DeleteTaskCommand(Guid Id)
    : ITransactionalCommand<Result>;
```

Prefer this unified abstraction over combining separate `ICommand<Result>` and `ITransactionalCommand` interfaces.

## Validation

Use `FluentValidation`.

Rules:

- `ValidationBehavior` runs before the handler;
- `ValidationBehavior` should never throw `ValidationException`;
- validation errors become `FluentResults` failures;
- Api maps validation errors to `400 Bad Request`;
- do not register each validator manually;
- use assembly scanning: `services.AddValidatorsFromAssembly(typeof(AppDependencies).Assembly);`
- manual registration is allowed only for conditional validators;
- add `FluentValidation.DependencyInjectionExtensions` when assembly scanning is needed.

Default `AppDependencies` shape:

```csharp
public static IServiceCollection AddAppDependencies(this IServiceCollection services)
{
    services.AddMediator(options =>
    {
        options.Assemblies = [typeof(AppDependencies)];
        options.ServiceLifetime = ServiceLifetime.Scoped;
        options.PipelineBehaviors =
        [
            typeof(ExceptionBehavior<,>),
            typeof(ValidationBehavior<,>),
            typeof(TransactionBehavior<,>)
        ];
    });

    services.AddValidatorsFromAssembly(typeof(AppDependencies).Assembly);
    services.AddSingleton(TimeProvider.System);

    return services;
}
```

Register `TransactionBehavior<,>` when EF Core write commands exist.

## Logging

### General

Use `Microsoft.Extensions.Logging`. Log only meaningful business events and operational failures. Do not log successful CRUD operations unless required by audit, security, or product requirements. Prefer `EventId` for important application logs when the project uses `EventId`.

Domain entities must never depend on `ILogger`, and logging must not happen inside domain entities.

### Structured Logging

Log message templates must be constant strings. Do not use string interpolation or string concatenation in log messages.

Put dynamic values into structured logging parameters or logger scope tags. If an identifier is already in scope, do not duplicate it in the message.

Log exceptions through overloads where the exception is the first argument after the log-level extension target, for example `logger.LogError(exception, "Unexpected task processing error")`.

Do not log and rethrow the same exception repeatedly. Unexpected exceptions are usually logged once, preferably in the centralized exception pipeline behavior or HTTP fallback.

### Logger Scopes

Use logger scopes for operation tags that should be attached to all log entries inside an operation. If the project already provides a `LoggingScope` abstraction, always use it. Do not invent a second `LoggingScope`. Before introducing a new helper, extension method, service, wrapper, or utility, check whether an equivalent abstraction already exists in the project.

Handlers should create a scope at the start of execution and add important operation identifiers before writing log entries. Keep messages short and constant; context belongs in tags or structured parameters.

Good:

```csharp
using var loggingScope = logger
    .Prepare()
    .AddTaskId(command.TaskId)
    .AddUserId(command.UserId);

logger.LogInformation("Запрошено изменение статуса задачи");
```

Bad:

```csharp
logger.LogInformation($"Task {command.TaskId} status change requested");
```

Prefer stable tag names: `RequestId`, `CorrelationId`, `TraceId`, `UserId`, `UserName`, `Role`, `BranchId`, `EntityId`, `AggregateId`, `ExternalSystemId`.

### Sensitive Data

Never log passwords, tokens, secrets, connection strings, or personal data unless explicitly required and approved.

## Middleware And API Layer

Use middleware for HTTP-level cross-cutting concerns such as exception-to-ProblemDetails fallback, correlation id, request logging, authentication, and authorization. Middleware must not contain business logic.

Keep `{ProjectName}.Api` thin. Endpoints or controllers should accept HTTP requests, map DTOs to commands or queries, call `IMediator.Send(...)`, and map `FluentResults.Result<T>` to HTTP responses.

Do not put business logic in controllers or endpoints. Do not inject `DbContext` directly into controllers or endpoints. Use `ProblemDetails` for API errors.

## API Versioning

Prefer ASP.NET API Versioning. Do not hardcode versions into controller names. Use URL versioning unless the project explicitly requires another strategy. Support OpenAPI generation for each API version.

## OpenAPI

For .NET 9+ and .NET 10+, prefer built-in ASP.NET Core OpenAPI generation via `Microsoft.AspNetCore.OpenApi`.

Do not add Swashbuckle or Swagger by default. Use `Scalar.AspNetCore` as the default interactive API documentation UI when an API UI is needed.

Use `builder.Services.AddOpenApi()`, `app.MapOpenApi()`, and `app.MapScalarApiReference()` in `{ProjectName}.Api`. Do not introduce OpenAPI, Swagger, or Scalar dependencies into App, Domain, or Infrastructure.

Use XML comments and endpoint metadata to improve OpenAPI documentation. Do not use `Newtonsoft.Json` for OpenAPI or API serialization.

## Runtime Code Rules

Enable Nullable Reference Types. Do not suppress nullable warnings unless justified. Prefer fixing nullability instead of using `!`; use `!` only at narrow framework boundaries or in tests where the invariant is obvious and documented.

Always use `async` / `await`. Do not use `Task.Run` for async I/O, `.Result`, or `.Wait()`. Always propagate `CancellationToken`; it should be the last parameter of every async method. Do not use `ConfigureAwait(false)` in ASP.NET Core applications unless there is a very specific reason.

Use `DateTime.UtcNow` or `TimeProvider`. Always prefer `TimeProvider` for application services, business logic, domain rules that depend on current time, scheduled jobs, and any current-time usage that must be testable. Prefer `TimeProvider` over direct `DateTime.UtcNow` whenever current time must be testable. Do not inject `ISystemClock`. Do not use `DateTime.Now`. Store timestamps in UTC unless the domain explicitly requires local time.

Prefer `Guid` identifiers unless the domain naturally uses another identifier such as SKU, EAN, ISBN, email, username, or other business keys. Use strongly typed identifiers when they improve domain clarity without unnecessary ceremony.

Avoid multiple enumeration. Prefer `Any()` over `Count() > 0`, `FirstOrDefault()` over `Where(...).FirstOrDefault()`, and avoid unnecessary materialization with `ToList()`, `ToArray()`, or similar methods.

Prefer `IReadOnlyList<T>` when element order matters. Prefer `IReadOnlyCollection<T>` when only enumeration and `Count` are required. Avoid exposing mutable collections unless mutation is part of the explicit contract.

Validate public API arguments. Use `ArgumentNullException.ThrowIfNull(...)` where appropriate and guard clauses when invalid inputs would break invariants.

## Testing

Write unit tests for App handlers and Domain logic. Test Domain without DI and without a database. Unit tests must not depend on a real database; use mocks or fakes and never mock the class under test.

When implementing a feature:

1. Create unit tests first.
2. Cover every business branch.
3. Mock only external dependencies.
4. Keep one assert concept per test.
5. Use AAA: Arrange, Act, Assert.
6. Name tests as `Method_State_ExpectedResult`.

Use EF Core InMemory cautiously because it does not behave like a relational database. For realistic application or integration tests, prefer SQLite InMemory or Testcontainers PostgreSQL. Test Infrastructure with integration tests.

Prefer FluentAssertions for unit tests unless the existing project already follows another assertion style:

```csharp
result.IsSuccess.Should().BeTrue();
task.Status.Should().Be(TaskStatus.Done);
```

Cover successful handler scenarios, validation errors, not found, conflict, transaction rollback, exception pipeline behavior, and `FluentResults` to HTTP mapping.

When `IUnitOfWork` / `TransactionBehavior` exists, also cover repeated `BeginTransactionAsync`, `CommitAsync` without a started transaction, safe no-op rollback, successful commit, failed-result rollback, and exception rollback with rethrow.

## Code Style And Project Cleanliness

Avoid:

- god services;
- automatic generic repository wrappers over EF Core;
- static service locators;
- `.Result` and `.Wait()`;
- vague names such as `Manager`, `Processor`, `Executor`, `Helper`, `Utils`, `DataManager`, `BusinessLogic`, `Common`, `BaseService`, `GlobalHelper`;
- folders or namespaces named `Common`, `Shared`, `Helpers`, `Utils`, or `Base` by default.

Use forbidden folder or namespace names only when the project already uses them consistently or the user explicitly asks for them. Follow Existing Code First.

Prefer purpose-specific folders and namespaces that describe architectural role or business capability, such as `Pipeline`, `Results`, `Logging`, `Contracts`, `Endpoints`, `Persistence`, `Configurations`, `Options`, `Tasks`, `Orders`, `Users`.

For single-line `if` statements, prefer no braces. Use braces only when the body contains multiple statements or when it improves readability.

Do not generate placeholder code, `TODO` comments unless requested, unused `using` directives, dead code, empty folders, or files that are never referenced by the project.

When generating a new project, create only files required by the requested functionality, remove template artifacts, and ensure the solution builds without warnings caused by generated template files.

## Final Validation

Before completing any task, verify:

- public XML documentation is complete and Russian;
- architecture boundaries are preserved;
- configuration uses a single `AppConfig` root record, is registered only in `Program.cs`, and is consumed through `IOptions<AppConfig>`;
- `TransactionBehavior`, repositories, and `IUnitOfWork` follow this standard;
- no forbidden folders or names were introduced;
- no template artifacts, dead code, or unused files were introduced;
- the project builds successfully;
- tests pass when tests exist.
