---
name: asp-net-core-architecture-default
description: Default architecture guidance for designing, reviewing, or implementing corporate backend applications with ASP.NET Core, C#, and .NET. Use when Codex proposes project structure, dependencies, DI, configuration, logging, package selection, Mediator handlers, FluentResults, EF Core persistence, pipeline behaviors, validation, middleware, APIs, API versioning, OpenAPI, XML comments, or tests for enterprise ASP.NET Core backends. Prefer Layered or Clean Architecture and do not default to Vertical Slice Architecture.
---

# ASP.NET Core Architecture Default

Use this skill to design, review, or implement corporate ASP.NET Core backend applications. Prefer Layered / Clean Architecture. Do not use Vertical Slice Architecture by default.

When using .NET APIs, always prefer the latest stable Microsoft recommendations over legacy approaches.

Avoid obsolete packages and APIs, including old `Startup.cs`-centric setups, legacy Swagger defaults, `Newtonsoft.Json` for normal API JSON, and outdated dependency injection patterns.

If a request contradicts this skill, explicitly explain the tradeoffs before violating the architecture.

Do not silently ignore architecture rules. If the user asks for a conflicting choice, name the conflict, recommend the default standard, and then show the requested alternative only when the user still needs it.

When modifying an existing project:

- Prefer consistency with the existing project.
- Do not rewrite large parts of the project only to satisfy a preferred style.
- Refactor only the parts required by the requested change unless the user explicitly asks for a larger architecture change.

## Existing Projects

When working in an existing solution:

- inspect the existing architecture first
- reuse existing abstractions
- reuse existing dependency injection style
- reuse existing folder structure
- reuse existing naming conventions
- reuse existing logging abstractions
- reuse existing repository implementations
- reuse existing testing conventions
- do not introduce a second implementation of an already solved problem
- prefer extending the current architecture over replacing it

The goal is consistency with the existing codebase.

## Architecture Decision

When several implementations are possible:

1. Prefer the simplest correct solution.
2. Do not introduce abstractions before they are needed.
3. Avoid overengineering.
4. Explain tradeoffs.
5. Explain why the chosen design is preferable.

## Required Response Order

When proposing architecture or code, present the solution in this order:

1. Project structure.
2. Project dependencies.
3. Dependency Injection registrations.
4. Command/query, validator, handler.
5. Pipeline behaviors.
6. Endpoint or controller.
7. Tests.
8. Explain why the solution does not violate the architecture rules.

## Project Structure

Start from at least four projects:

```text
{ProjectName}.Api
{ProjectName}.App
{ProjectName}.Domain
{ProjectName}.Infrastructure
```

Add these only when justified by the problem:

```text
{ProjectName}.Tech
{ProjectName}.Contracts
{ProjectName}.Tests.Unit
{ProjectName}.Tests.Integration
```

Prefer `.slnx` over `.sln` for new .NET solutions when supported by the SDK.

Organize the solution into `src` and `tests`:

```text
src/
tests/
```

Place all production projects under `src`. Place all test projects under `tests`.

Do not keep generated template files that are not used by the project. Remove artifacts such as:

- `*.http`
- `WeatherForecast` sample code
- default controllers
- sample endpoints
- sample DTOs
- sample configuration
- template README files when they contain only boilerplate

The generated solution should contain only files actually used by the project.

## Project Dependencies

Enforce these references:

- `{ProjectName}.Api` depends on `{ProjectName}.App` and `{ProjectName}.Infrastructure`.
- `{ProjectName}.App` depends on `{ProjectName}.Domain`.
- `{ProjectName}.Infrastructure` depends on `{ProjectName}.App` and `{ProjectName}.Domain`.
- `{ProjectName}.Domain` depends on no application project.

Keep domain model and business rules independent from ASP.NET Core, EF Core, JSON serialization, external services, and infrastructure frameworks.

## Dependency Injection

Put all service registrations for each project in a root-level dependencies class:

- `ApiDependencies`
- `AppDependencies`
- `DomainDependencies`
- `InfrastructureDependencies`
- `TechDependencies`, only when `{ProjectName}.Tech` exists

Keep `Program.cs` thin. It should compose dependency modules, configure API-level services, and start the app.

Prefer constructor injection.

Never resolve services through `IServiceProvider` unless implementing infrastructure or framework integration code.

Never inject `IServiceProvider`.

Never inject `IServiceScopeFactory` unless implementing infrastructure or framework integration code.

Avoid property injection.

Use primary constructors / concise constructors for services when suitable.

## Configuration

Use `IOptions<T>` for application settings.

Prefer `ValidateOnStart()`.

Avoid `IConfiguration` usage outside the composition root and infrastructure configuration code.

Group settings into dedicated option classes.

Keep option classes strongly typed and close to the layer that owns the setting.

## AppSettings

Do not silently fallback to hardcoded configuration values.

Required configuration values must be read from `appsettings` or environment variables and validated on startup.

If a required connection string or option is missing, fail fast with a clear exception.

## Package Selection

Prefer official Microsoft libraries when they provide equivalent functionality.

Avoid adding third-party packages without clear justification.

Before introducing a package, verify that .NET does not already provide the feature.

Default examples:

- Prefer `System.Text.Json` over `Newtonsoft.Json`.
- Prefer ordinary projections such as `Select(...)` over `AutoMapper` when mapping is simple.
- Prefer `Microsoft.Extensions.Resilience` on .NET 8+ over adding `Polly` directly unless the project needs Polly-specific APIs.
- Prefer built-in ASP.NET Core OpenAPI plus Scalar over Swashbuckle/Swagger by default.

## Mediator

Use Martin Othamar's `Mediator` package. Do not use `MediatR` by default.

Handlers must not return `IResult`, `Results`, `ActionResult`, status codes, or any other HTTP-specific type. Handlers return:

```csharp
FluentResults.Result
FluentResults.Result<T>
```

Endpoints or controllers call `mediator.Send(...)` and map `Result<T>` to HTTP responses in the Api layer.

## Result Pattern

Use `FluentResults` for expected business outcomes and errors.

- Use `Result` / `Result<T>` for expected business errors.
- Do not throw exceptions for normal validation, not-found, conflict, or business-rule outcomes.
- Map `FluentResults` to HTTP only in `{ProjectName}.Api`.
- Do not let handlers know about HTTP.

## Error Codes

Do not use magic strings for error codes.

Define shared constants for error codes.

Use these constants in `FluentResults` metadata and HTTP result mapping.

## User-Facing Messages

Prefer Russian user-facing messages in validation errors, `ProblemDetails`, and business errors unless the project explicitly uses English.

## JSON

Use `System.Text.Json`. Do not use `Newtonsoft.Json` by default.

- Keep JSON options in `{ProjectName}.Api`.
- Do not introduce JSON serialization concerns into `{ProjectName}.Domain`.
- Prefer `System.Text.Json` source generators when appropriate.
- Do not use dynamic JSON.
- Prefer strongly typed DTOs.

## Nullable Reference Types

Enable Nullable Reference Types.

Do not suppress nullable warnings unless there is a justified reason.

Prefer fixing the nullability issue instead of using the null-forgiving operator (`!`).

Use `!` only at narrow framework boundaries or in tests where the invariant is obvious and documented.

## XML Comments

Use XML documentation comments for public classes, records, interfaces, methods, constructors, and important properties.

Write XML comments in Russian.

Comments should describe business meaning and intent. Do not mechanically repeat the code.

Use:

- `<summary>` for purpose.
- `<param>` for parameters.
- `<returns>` for return value when useful.
- `<inheritdoc />` or `<inheritdoc cref="TypeName"/>` for constructors or interface implementations when appropriate.
- `<see cref="TypeName"/>` for references.

Prefer concise comments like:

```csharp
/// <summary>
/// Запрос на создание задачи в списке работ пользователя.
/// </summary>
/// <param name="Title">Краткое название задачи.</param>
/// <param name="Description">Дополнительное описание задачи.</param>
public sealed record CreateTaskRequest(string Title, string? Description);
```

Avoid useless comments like:

- `Gets or sets value`.
- `Handler class`.
- `Method handles request`.

Do not generate XML comments for every private member.

Do not add XML comments to obvious private implementation details unless they explain an important business rule or non-obvious behavior.

## EF Core

Place `DbContext` in `{ProjectName}.Infrastructure`.

Every entity configuration must live in a separate class implementing `IEntityTypeConfiguration<TEntity>`:

```csharp
internal sealed class OrderConfiguration : IEntityTypeConfiguration<Order>
{
    public void Configure(EntityTypeBuilder<Order> builder)
    {
        // Fluent API mapping here
    }
}
```

Never configure entities inline inside `OnModelCreating` except `ApplyConfigurationsFromAssembly`. In `OnModelCreating`, only call:

```csharp
modelBuilder.ApplyConfigurationsFromAssembly(typeof(AppDbContext).Assembly);
```

Keep database mapping separate from domain entities. Avoid EF Core attributes in domain entities when Fluent API can express the mapping.

For read-only queries, use `AsNoTracking()`.

Watch for N+1 queries, unnecessary `Include`, premature or unnecessary `ToList`, and multiple enumeration.

## Repositories / Persistence Ports

Use application persistence abstractions / repositories in this architecture standard.

App handlers must not inject EF Core `DbContext` directly.

App handlers should depend on repository abstractions declared in `{ProjectName}.App`.

Infrastructure implements these abstractions using EF Core.

`DbContext` stays in `{ProjectName}.Infrastructure`.

Do not create generic `IRepository<T>` by default.

Use only the `Repository` suffix for persistence abstractions in one project. Do not mix `Repository` and `Store` styles.

Prefer focused repositories named by business capability:

- `ITaskItemRepository`
- `IOrderRepository`
- `IUserRepository`

Repositories should expose operations required by use cases, not `IQueryable` by default.

Do not leak EF Core types from repository abstractions.

Repositories should not call `SaveChangesAsync` by default.

Commit should be centralized through `IUnitOfWork` and `TransactionBehavior` / `UnitOfWorkBehavior`.

`IUnitOfWork` is an App-layer transaction abstraction. Infrastructure implements it using EF Core.

## Domain Entity Model

Domain entities should be classes, not records, when they have identity, lifecycle, and behavior.

Do not use record `with` or public `init` setters for mutable entities.

Use private setters and behavior methods to protect invariants.

Parameterless constructors for EF Core should be private when possible.

Keep EF Core attributes out of Domain entities; use Fluent API configurations in Infrastructure.

Entity state changes should happen through domain methods.

Public setters are not allowed on domain entities unless there is a strong reason.

Validate invariants inside factory methods and domain methods.

Return `FluentResults.Result` / `Result<T>` for expected domain rule failures.

Use UTC `DateTime` values or `TimeProvider` from the application layer. Do not call `DateTime.Now` inside entities.

Do not inject services into entities.

Value Objects may be records when immutable.

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

        return Result.Ok(new TaskItem(
            id,
            title.Trim()));
    }

    public Result Update(...)
    {
        ...
        return Result.Ok();
    }
}
```

Bad entity style:

```csharp
public sealed record TaskItem
{
    public Guid Id { get; init; }
    public string Title { get; init; }
    public TaskItemStatus Status { get; init; }
}
```

Reason: `record` / `init` encourages external state mutation and bypasses domain methods for entities with lifecycle.

## Date and Time

Use `DateTime.UtcNow` or `TimeProvider`.

Prefer `TimeProvider` for testable code, domain rules that depend on current time, scheduled jobs, and application services.

Avoid `DateTime.Now`.

Store timestamps in UTC unless the domain explicitly requires local time.

## Identifiers

Prefer `Guid` identifiers unless the domain naturally uses another identifier such as SKU, EAN, ISBN, email, username, or other business keys.

Use strongly typed identifiers when they improve domain clarity without adding unnecessary ceremony.

## Pipeline Behaviors

Centralize handler-level cross-cutting concerns in Mediator pipeline behaviors. Do not write repetitive `try/catch` blocks in every endpoint, controller, or handler.

Use pipeline behaviors for:

- exception handling
- validation
- logging
- transactions
- authorization
- metrics and tracing

Unexpected exceptions should be logged in `ExceptionBehavior`.

Business logic should not rely on exceptions.

Expected failures should always be represented by `FluentResults`.

Pipeline behaviors belong to `{ProjectName}.App`.

Place these behaviors in `App/Pipeline`:

- `ExceptionBehavior`
- `ValidationBehavior`
- `TransactionBehavior` / `UnitOfWorkBehavior`

Do not place pipeline behaviors in `{ProjectName}.Infrastructure`.

Pipeline behaviors must not depend on EF Core, `DbContext`, infrastructure implementations, or `Microsoft.EntityFrameworkCore`.

## Transactions and SaveChanges

Do not call `SaveChangesAsync` manually in every command handler.

When EF Core persistence is used, prefer a Mediator `TransactionBehavior` / `UnitOfWorkBehavior` for write commands.

Register `TransactionBehavior` automatically in the Mediator pipeline when introduced.

`TransactionBehavior` belongs to `{ProjectName}.App/Pipeline`.

`TransactionBehavior` should depend on an App-layer `IUnitOfWork` abstraction, not on EF Core `DbContext`.

Declare `IUnitOfWork` in `{ProjectName}.App`:

```csharp
public interface IUnitOfWork
{
    Task BeginTransactionAsync(CancellationToken cancellationToken);
    Task CommitAsync(CancellationToken cancellationToken);
    Task RollbackAsync(CancellationToken cancellationToken);
}
```

Prefer explicit `Begin` / `Commit` / `Rollback` methods when the behavior manages transaction flow.

Infrastructure implements `IUnitOfWork` using EF Core `AppDbContext`.

The Infrastructure implementation:

- lives in `Infrastructure/Persistence`
- uses `AppDbContext.Database.BeginTransactionAsync(...)`
- calls `AppDbContext.SaveChangesAsync(...)`
- commits and rolls back the EF Core transaction

The App layer must not know about EF Core.

Handlers should change domain state and call repositories, but transaction commit should be centralized through `IUnitOfWork`.

Command handlers should not call `SaveChangesAsync` directly when `TransactionBehavior` is used.

In this architecture standard, `IUnitOfWork` is allowed and preferred as an App-layer transaction abstraction.

`IUnitOfWork` exists to keep `TransactionBehavior` in App and prevent App from depending on EF Core.

`IUnitOfWork` should be focused only on transaction / commit / rollback orchestration.

Do not put repositories, query methods, or generic data access methods into `IUnitOfWork`.

`TransactionBehavior` should:

- run only for commands that modify persistence state
- not run for read-only queries
- use a marker interface such as `ITransactionalCommand`
- call the handler
- rollback when the handler returns failed `FluentResults.Result` / `Result<T>`
- commit when the handler returns a successful result
- rollback and rethrow on unexpected exceptions
- not depend on `{ProjectName}.Infrastructure`, EF Core, `AppDbContext`, or `Microsoft.EntityFrameworkCore`

Do not open transactions for read-only queries.

Avoid double `SaveChangesAsync`: never call it both in a handler and `TransactionBehavior`.

## Logging

Use `Microsoft.Extensions.Logging`.

Log only meaningful business events and operational failures.

Do not log successful CRUD operations unless required by audit, security, or product requirements.

Log message templates must be constant strings.

Do not use string interpolation or string concatenation in log messages.

Put dynamic values into structured logging parameters or logger scope tags.

Log exceptions through overloads where the exception is the first argument after log level extension target, for example `logger.LogError(exception, "Unexpected task processing error")`.

Do not log and rethrow the same exception repeatedly.

Unexpected exceptions are usually logged once, preferably in the centralized exception pipeline behavior or HTTP fallback.

Never log passwords, tokens, secrets, or personal data.

Always include identifiers that help troubleshooting, such as request id, aggregate id, tenant id, correlation id, or external system id.

Prefer `EventId` for important application logs when the project uses `EventId`.

Domain entities must never depend on `ILogger`.

Do not perform logging inside domain entities.

## Logger Scope

Use logger scopes for operation tags that should be attached to all log entries inside an operation.

If the project already provides a `LoggingScope` abstraction, always use it.

Do not invent another logging scope implementation.

Prefer existing project abstractions over creating new ones.

Before introducing a new helper, extension method, service, wrapper, or utility, check whether an equivalent abstraction already exists in the project.

When the project does not already have a logging scope abstraction, use a dedicated `LoggingScope` helper or equivalent extension only when scoped tags are useful.

Handlers should create logger scope at the start of execution and add important operation identifiers before writing log entries.

Keep log messages short and constant.

Do not insert contextual data into the message string; put it into scope tags or structured logging parameters.

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

Logger scope helpers should expose domain-specific tag methods when useful:

```csharp
public LoggingScope AddTaskId(Guid taskId) => WithTag("TaskId", taskId);
public LoggingScope AddUserId(Guid userId) => WithTag("UserId", userId);
public LoggingScope AddCorrelationId(string correlationId) => WithTag("CorrelationId", correlationId);
```

Prefer stable tag names:

- `RequestId`
- `CorrelationId`
- `TraceId`
- `UserId`
- `UserName`
- `Role`
- `BranchId`
- `EntityId`
- `AggregateId`
- `ExternalSystemId`

Do not put sensitive data into tags:

- passwords
- tokens
- secrets
- connection strings
- personal data unless explicitly required and approved

## Validation

Use `FluentValidation`.

- `ValidationBehavior` runs validators before the handler.
- `ValidationBehavior` should never throw `ValidationException`.
- Validation errors should be converted into `FluentResults` failures.
- Api maps validation errors to `400 Bad Request`.

## FluentValidation Registration

Do not register each validator manually.

Use assembly scanning for validators:

```csharp
services.AddValidatorsFromAssembly(typeof(AppDependencies).Assembly);
```

Manual validator registration is allowed only when validators must be registered conditionally.

Add `FluentValidation.DependencyInjectionExtensions` when assembly scanning is needed.

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

## Middleware

Use middleware for HTTP-level cross-cutting concerns:

- exception-to-ProblemDetails fallback
- correlation id
- request logging
- authentication and authorization

Middleware must not contain business logic.

## API Layer

Keep `{ProjectName}.Api` thin.

Endpoints or controllers should:

- accept the HTTP request
- map request DTOs to commands or queries
- call `IMediator.Send(...)`
- map `FluentResults.Result<T>` to HTTP responses

Do not put business logic in controllers or endpoints. Do not inject `DbContext` directly into controllers or endpoints.

Use `ProblemDetails` for API errors.

## API Versioning

Prefer ASP.NET API Versioning.

Do not hardcode versions into controller names.

Use URL versioning unless the project explicitly requires another strategy.

Support OpenAPI generation for each API version.

## OpenAPI

For .NET 9+ and .NET 10+, prefer built-in ASP.NET Core OpenAPI generation via `Microsoft.AspNetCore.OpenApi`.

Do not add Swashbuckle or Swagger by default.

Use `Scalar.AspNetCore` as the default interactive API documentation UI when an API UI is needed.

Recommended setup:

```csharp
builder.Services.AddOpenApi();

if (app.Environment.IsDevelopment())
{
    app.MapOpenApi();
    app.MapScalarApiReference();
}
```

OpenAPI and Scalar configuration belongs to `{ProjectName}.Api`.

Do not introduce OpenAPI, Swagger, or Scalar dependencies into `{ProjectName}.App`, `{ProjectName}.Domain`, or `{ProjectName}.Infrastructure`.

Use XML comments and endpoint metadata to improve OpenAPI documentation.

Do not use `Newtonsoft.Json` for OpenAPI or API serialization.

## Async

Always use `async` / `await`.

Never use:

- `Task.Run` for async I/O
- `.Result`
- `.Wait()`

Always propagate `CancellationToken`.

`CancellationToken` should be the last parameter of every async method.

Do not use `ConfigureAwait(false)` in ASP.NET Core applications unless there is a very specific reason.

## LINQ

Avoid multiple enumeration.

Prefer `Any()` over `Count() > 0`.

Prefer `FirstOrDefault()` over `Where(...).FirstOrDefault()`.

Avoid unnecessary materialization with `ToList()`, `ToArray()`, or similar methods.

Defer materialization until the boundary where a concrete collection is needed.

## Collections

Prefer `IReadOnlyList<T>` when element order matters.

Prefer `IReadOnlyCollection<T>` when only enumeration and `Count` are required.

Avoid exposing mutable collections from domain entities, DTOs, and application contracts unless mutation is part of the explicit contract.

## Public API Arguments

Validate public API arguments.

Use `ArgumentNullException.ThrowIfNull(...)` where appropriate.

Use guard clauses for public constructors and methods when invalid inputs would break invariants.

## Testing

Write unit tests for App handlers and Domain logic.

When implementing a feature:

1. Create unit tests first.
2. Cover every business branch.
3. Mock only external dependencies.
4. Never mock the class under test.
5. Keep one assert concept per test.
6. Use the AAA pattern: Arrange, Act, Assert.
7. Name tests as `Method_State_ExpectedResult`.

- Unit tests must not depend on a real database.
- Use mocks or fakes for unit tests.
- Use EF Core InMemory cautiously because it does not behave like a relational database.
- For realistic application or integration tests, prefer SQLite InMemory or Testcontainers PostgreSQL.
- Test Domain logic without DI and without a database.
- Test Infrastructure with integration tests.
- Prefer FluentAssertions for unit tests unless the existing project already follows another assertion style.

Prefer:

```csharp
result.IsSuccess.Should().BeTrue();
task.Status.Should().Be(TaskStatus.Done);
```

Instead of:

```csharp
Assert.True(result.IsSuccess);
Assert.Equal(TaskStatus.Done, task.Status);
```

Cover:

- successful handler scenario
- validation errors
- not found
- conflict
- transaction rollback
- exception pipeline behavior
- `FluentResults` to HTTP mapping

## Code Style Rules

Avoid:

- god services
- automatic repository wrappers over EF Core
- `Manager`, `Processor`, `Executor`, `Helper`, and `Utils` names
- `DataManager`, `BusinessLogic`, `Common`, `BaseService`, and `GlobalHelper` names
- folders or namespaces named `Common`, `Shared`, `Helpers`, `Utils`, or `Base` by default
- static service locators
- `.Result` and `.Wait()`

Do not create folders or namespaces named `Common`, `Shared`, `Helpers`, or `Utils` by default. These names are allowed only when the project already uses them consistently or when the user explicitly asks for them.

Prefer purpose-specific folders and namespaces. Folder names must describe the architectural role or business capability.

Prefer names such as:

- `Pipeline`
- `Results`
- `Logging`
- `Contracts`
- `Endpoints`
- `Persistence`
- `Configurations`
- `Options`
- `Tasks`
- `Orders`
- `Users`

Avoid vague names such as:

- `Common`
- `Shared`
- `Helpers`
- `Utils`
- `Base`

For single-line `if` statements, prefer no braces.

Use braces only when the body contains multiple statements or when it improves readability.

Prefer names that describe the business capability.

Keep services focused. Prefer explicit dependencies and clear application use-case boundaries.

## Project Cleanliness

Do not generate placeholder code.

Do not leave `TODO` comments unless explicitly requested.

Do not leave unused using directives.

Do not leave dead code.

Do not create empty folders.

Do not add files that are never referenced by the project.

Prefer a minimal, production-ready project structure.

## Generated Files

When generating a new project:

- create only the files required by the requested functionality
- remove template artifacts
- ensure the solution builds successfully without warnings caused by generated template files
