---
name: asp-net-core-signalr-default
description: Default standard for adding, reviewing, or refactoring SignalR in corporate ASP.NET Core applications. Use for SignalR hubs, hub clients, realtime notifications, websocket messaging, backend services communicating with connected frontend or terminal clients, SignalR configuration, hub authentication, user id providers, connection lifecycle, server-to-client calls, hub endpoint mapping, payload rules, logging, frontend SignalR handlers, and tests.
---

# ASP.NET Core SignalR Default

## Purpose

Use this skill when adding, reviewing, or refactoring SignalR hubs, hub clients, realtime notifications, websocket messaging, or integration between backend services and connected frontend or terminal clients.

Use SignalR only for small realtime messages:

- notify that data changed;
- tell a client to refresh;
- send an event id, document id, message id, or aggregate id;
- ask a client to fetch details through HTTP API;
- send heartbeat or lightweight status;
- send command-level notifications.

Do not use SignalR as the main transport for large payloads.

## Core Principle

SignalR messages should be tiny.

Prefer:

- `RefreshRequested`;
- `MessageCreated(messageId)`;
- `OrderChanged(orderId)`;
- `TerminalStatusChanged(terminalId)`;
- `FetchDocument(documentId)`.

Avoid:

- sending large DTOs;
- sending tables or lists;
- sending files;
- sending logs;
- sending binary payloads;
- replacing HTTP API with SignalR.

The client receives a small event through SignalR, then loads full data through HTTP API when needed.

## Project Placement

Use this placement:

```text
src/
    Project.Api/
        Hubs/
            TerminalHub.cs
            TerminalHubClient.cs
    Project.App/
        Abstractions/
            ITerminalHubClient.cs
```

Rules:

- hub implementation lives in `{ProjectName}.Api`;
- hub route mapping lives in `{ProjectName}.Api`;
- hub client implementation lives in `{ProjectName}.Api`;
- hub client interface or abstraction lives in `{ProjectName}.App`;
- App layer knows only about the hub client interface;
- App layer must not depend on SignalR types;
- Domain must not know about SignalR;
- Infrastructure should not own hubs by default;
- Api composes SignalR and implements the App abstraction.

## App Abstraction

Declare hub client interfaces in `{ProjectName}.App`.

```csharp
/// <summary>
/// Клиент уведомлений терминалов.
/// </summary>
public interface ITerminalHubClient
{
    /// <summary>
    /// Запрашивает обновление данных терминала.
    /// </summary>
    /// <param name="terminalId">Уникальный идентификатор терминала.</param>
    /// <param name="cancellationToken">Токен отмены операции.</param>
    Task RequestTerminalRefreshAsync(Guid terminalId, CancellationToken cancellationToken);

    /// <summary>
    /// Уведомляет о появлении сообщения.
    /// </summary>
    /// <param name="messageId">Уникальный идентификатор сообщения.</param>
    /// <param name="cancellationToken">Токен отмены операции.</param>
    Task NotifyMessageCreatedAsync(Guid messageId, CancellationToken cancellationToken);
}
```

Rules:

- interface methods should be business-oriented;
- method names must end with `Async`;
- always pass `CancellationToken`;
- do not expose `IHubContext`, `Hub`, `HubCallerContext`, `Clients`, `Groups`, or SignalR-specific types in App interfaces;
- App handlers call the interface, not SignalR directly;
- public C# API must have Russian XML comments.

## Api Hub

Hub classes live in `{ProjectName}.Api/Hubs`.

```csharp
/// <summary>
/// SignalR hub терминалов.
/// </summary>
public sealed class TerminalHub : Hub
{
}
```

Rules:

- keep hub classes thin;
- hub methods should not contain business logic;
- hub methods may identify connections, manage groups, or receive lightweight client events;
- do not inject repositories or `DbContext` into hubs;
- if a hub receives a client command, delegate to Mediator or an App service;
- public hub methods must end with `Async`;
- public hub methods must have Russian XML comments;
- use `[HubMethodName]` when the client protocol requires stable or legacy method names;
- hub method names may use C# naming while wire protocol names remain stable;
- always await handler calls;
- always pass `CancellationToken`.
- track connections in `OnConnectedAsync` and `OnDisconnectedAsync` when the project needs server-to-client calls by host id, terminal id, or user id;
- keep connection lifecycle methods focused on connection bookkeeping.

Example:

```csharp
/// <summary>
/// Хаб для работы с хостами.
/// </summary>
[Authorize(AuthenticationSchemes = "HostJwt")]
public sealed class HostHub(AcceptHeartbeatWSHandler heartbeatWsHandler) : Hub
{
    /// <summary>
    /// Получает heartbeat хоста.
    /// </summary>
    /// <param name="heartbeatJson">Heartbeat хоста.</param>
    /// <param name="cancellationToken">Токен отмены операции.</param>
    [HubMethodName("addHeartbeat")]
    public async Task AddHeartbeatAsync(
        IHeartbeatRequest heartbeatJson,
        CancellationToken cancellationToken)
    {
        await heartbeatWsHandler.HandleAsync(heartbeatJson, cancellationToken);
    }

    /// <inheritdoc />
    public override async Task OnConnectedAsync()
    {
        var userId = Context.UserIdentifier;
        if (userId is not null)
        {
            webSocketConnectionTracker.AddConnection(userId);
            connectionRegistry.Set(userId, Context.ConnectionId);
        }

        await base.OnConnectedAsync();
    }

    /// <inheritdoc />
    public override async Task OnDisconnectedAsync(Exception? exception)
    {
        var userId = Context.UserIdentifier;
        if (userId is not null)
        {
            webSocketConnectionTracker.RemoveConnection(userId);
            connectionRegistry.Remove(userId);
        }

        await base.OnDisconnectedAsync(exception);
    }
}
```

## Api Hub Client Implementation

Implementation of the App hub client interface lives in `{ProjectName}.Api`.

```csharp
/// <summary>
/// SignalR реализация клиента уведомлений терминалов.
/// </summary>
public sealed class TerminalHubClient(IHubContext<TerminalHub> hubContext) : ITerminalHubClient
{
    /// <inheritdoc />
    public Task RequestTerminalRefreshAsync(Guid terminalId, CancellationToken cancellationToken)
    {
        return hubContext
            .Clients
            .Group(TerminalHubGroups.GetTerminalGroupName(terminalId))
            .SendAsync(TerminalHubMethods.TerminalRefreshRequested, terminalId, cancellationToken);
    }

    /// <inheritdoc />
    public Task NotifyMessageCreatedAsync(Guid messageId, CancellationToken cancellationToken)
    {
        return hubContext
            .Clients
            .All
            .SendAsync(TerminalHubMethods.MessageCreated, messageId, cancellationToken);
    }
}
```

Rules:

- SignalR-specific code stays in Api implementation;
- server-side hub client implementation may use `IHubContext<THub>`, `WebSocketConnectionRegistry`, `WebSocketClientExtensions`, method-name constants, and `RemoteMethodResult`;
- use constant method names instead of magic strings when possible;
- keep payloads small;
- prefer sending ids and event names;
- do not send full aggregates or large DTOs;
- do not block on async calls;
- always await SignalR calls when called from async methods.
- App interfaces must stay free from `IHubContext`, `Hub`, `HubConnectionContext`, `IClientProxy`, `Clients`, and other SignalR-specific types;
- App interfaces should expose business-oriented methods and may return an App-level result abstraction.

```csharp
/// <summary>
/// SignalR реализация клиента хостов.
/// </summary>
public sealed class HostHubClient(
    IHubContext<HostHub> hubContext,
    WebSocketConnectionRegistry connectionRegistry)
    : IHostHubClient
{
    /// <inheritdoc />
    public async Task RequestRefreshAsync(string hostId, CancellationToken cancellationToken)
    {
        var result = await hubContext.SendSignalAsync(
            connectionRegistry,
            hostId,
            HostHubMethods.RefreshRequested,
            cancellationToken);

        ...
    }

    /// <inheritdoc />
    public async Task<RemoteMethodResult<HostStatusDto>> GetStatusAsync(
        string hostId,
        CancellationToken cancellationToken)
    {
        return await hubContext.GetDataAsync<HostHub, HostStatusDto>(
            connectionRegistry,
            hostId,
            HostHubMethods.GetStatus,
            cancellationToken);
    }
}
```

## Message Names

Do not scatter SignalR method names as magic strings.

```csharp
/// <summary>
/// Названия клиентских SignalR методов.
/// </summary>
public static class TerminalHubMethods
{
    /// <summary>
    /// Запрос обновления терминала.
    /// </summary>
    public const string TerminalRefreshRequested = "TerminalRefreshRequested";

    /// <summary>
    /// Создано сообщение.
    /// </summary>
    public const string MessageCreated = "MessageCreated";
}
```

Rules:

- method names should be stable;
- use past tense for facts, for example `MessageCreated`;
- use `Requested` suffix for commands or requests, for example `TerminalRefreshRequested`;
- keep names short and explicit.

## Groups

Use groups for targeted notifications.

```csharp
private static string GetTerminalGroupName(Guid terminalId) => $"terminal:{terminalId}";
private static string GetUserGroupName(Guid userId) => $"user:{userId}";
private static string GetBranchGroupName(Guid branchId) => $"branch:{branchId}";
```

Rules:

- centralize group name generation;
- do not build group names inline everywhere;
- prefer stable prefixes;
- validate ids before adding connections to groups;
- do not trust client-provided group membership blindly;
- use authorization checks before joining sensitive groups.

## SignalR Configuration

SignalR options must be configured from application configuration.

```csharp
/// <summary>
/// Добавление зависимостей SignalR.
/// </summary>
/// <param name="services"><see cref="IServiceCollection"/>.</param>
/// <param name="appConfig"><see cref="AppConfig"/>.</param>
internal static IServiceCollection AddSignalRDependencies(
    this IServiceCollection services,
    AppConfig appConfig)
{
    services.Configure<HubOptions>(options =>
    {
        options.KeepAliveInterval = TimeSpan.FromSeconds(appConfig.SignalR.KeepAliveSeconds);
        options.ClientTimeoutInterval = TimeSpan.FromSeconds(appConfig.SignalR.TimeoutSeconds);
        options.EnableDetailedErrors = appConfig.SignalR.EnableDetailedErrors;
    });

    services.AddSignalR();
    services.AddSingleton<IUserIdProvider, HostKeyUserIdProvider>();

    return services;
}
```

Rules:

- do not hardcode keep-alive, timeout, or detailed-error settings;
- prefer strongly typed options or app config;
- SignalR configuration belongs to `{ProjectName}.Api`;
- keep configuration registration in API dependency registration methods;
- if the project uses `IOptions<T>` instead of direct `AppConfig`, follow the existing project configuration style.

## SignalR Appsettings

SignalR settings should live in `appsettings` or environment configuration.

```json
{
  "SignalR": {
    "KeepAliveSeconds": "...",
    "TimeoutSeconds": "...",
    "EnableDetailedErrors": "..."
  }
}
```

Rules:

- validate SignalR settings on startup when the project has options validation;
- do not silently fall back to hardcoded defaults;
- use the project's existing configuration pattern.

## DI And Hub Mapping

Register SignalR and hub clients in `{ProjectName}.Api`.

```csharp
builder.Services.AddSignalR();
builder.Services.AddScoped<ITerminalHubClient, TerminalHubClient>();

app.MapHub<TerminalHub>("/hub/terminal");
```

Map hubs explicitly in Api endpoints.

```csharp
endpoints.MapHub<HostHub>(
    "/hub/host",
    options =>
    {
        options.Transports = HttpTransportType.WebSockets;
    });
```

Rules:

- hub route starts with `/hub`;
- keep route names stable;
- prefer WebSockets transport when the project requires WebSocket-only connections;
- map hubs in the Api layer;
- do not map hubs in App, Domain, or Infrastructure;
- register hub client implementation in Api dependencies;
- App only depends on the interface.

## SignalR UserId Provider

Use a custom `IUserIdProvider` when connections should be identified by host key, terminal key, user id, or another domain identifier.

```csharp
/// <summary>
/// Провайдер конфигурации SignalR UserId по ключу хоста.
/// </summary>
public sealed class HostKeyUserIdProvider : IUserIdProvider
{
    /// <inheritdoc />
    public string? GetUserId(HubConnectionContext connection)
    {
        return connection
            .User?
            .FindFirst(ClaimTypes.NameIdentifier)?
            .Value;
    }
}
```

Rules:

- return nullable `string?` if the SignalR interface requires it;
- do not throw when claim is missing;
- use stable claim types;
- do not trust unauthenticated connection data;
- register the provider in Api: `services.AddSingleton<IUserIdProvider, HostKeyUserIdProvider>();`.

## Hub Authentication

Use a dedicated authentication scheme for host or terminal hubs when clients authenticate as hosts.

```csharp
.AddJwtBearer("HostJwt", options =>
{
    var issuer = appConfig.HostAuth.Issuer;
    var audience = appConfig.HostAuth.Audience;
    var secret = appConfig.Secrets.HostKeyAuth;

    options.IncludeErrorDetails = appConfig.HostAuth.IncludeErrorDetails;
    options.RequireHttpsMetadata = appConfig.HostAuth.RequireHttpsMetadata;
    options.TokenValidationParameters = new TokenValidationParameters
    {
        ValidateIssuer = true,
        ValidIssuer = issuer,
        ValidateAudience = true,
        ValidAudience = audience,
        ValidateLifetime = true,
        IssuerSigningKey = new SymmetricSecurityKey(Encoding.ASCII.GetBytes(secret)),
        ValidateIssuerSigningKey = true
    };
});
```

Rules:

- do not hardcode issuer, audience, secret, HTTPS metadata, or error-detail flags;
- read authentication settings from configuration;
- validate secrets and options on startup when possible;
- use a named scheme such as `"HostJwt"` for host hubs;
- apply authorization on the hub with the scheme.

```csharp
/// <summary>
/// Хаб для работы с хостами.
/// </summary>
[Authorize(AuthenticationSchemes = "HostJwt")]
public sealed class HostHub : Hub
{
    ...
}
```

## Connection Tracking

If the project tracks WebSocket or SignalR connections:

- keep tracking infrastructure in Api or an existing realtime infrastructure layer;
- do not put connection tracking into Domain;
- avoid business logic inside connection trackers;
- use connection registry or tracker only for operational connection state;
- use `Context.UserIdentifier` as the stable connection key when `IUserIdProvider` is configured;
- register connection id on connect;
- remove connection state on disconnect;
- always call `base.OnConnectedAsync()` and `base.OnDisconnectedAsync(exception)`;
- keep lifecycle methods focused on connection bookkeeping.

Examples: `WebSocketConnectionTracker`, `WebSocketConnectionRegistry`.

Use a registry when the server needs to invoke a method on a specific connected client.

```csharp
/// <summary>
/// Регистратор WebSocket соединений.
/// </summary>
public sealed class WebSocketConnectionRegistry
{
    private readonly ConcurrentDictionary<string, string> _byUser = new();

    /// <summary>
    /// Задать соединение.
    /// </summary>
    /// <param name="userId">Идентификатор пользователя.</param>
    /// <param name="connectionId">Идентификатор соединения.</param>
    public void Set(string userId, string connectionId)
    {
        _byUser[userId] = connectionId;
    }

    /// <summary>
    /// Получить соединение.
    /// </summary>
    /// <param name="userId">Идентификатор пользователя.</param>
    public string? GetConnection(string userId) => _byUser.GetValueOrDefault(userId);

    /// <summary>
    /// Удалить соединение.
    /// </summary>
    /// <param name="userId">Идентификатор пользователя.</param>
    public void Remove(string userId)
    {
        _byUser.TryRemove(userId, out _);
    }
}
```

Registry rules:

- use `ConcurrentDictionary`;
- registry stores operational connection state only;
- registry must not contain business logic;
- prefer `Set`, `GetConnection`, and `Remove`;
- clean stale connection ids on disconnect;
- if multiple connections per user or host are supported, store a collection instead of a single connection id.

## Remote Method Result

When the server invokes a method on a connected client and expects a response, use a small typed result object.

```csharp
/// <summary>
/// Результат ответа удалённого метода.
/// </summary>
public record RemoteMethodResult
{
    /// <summary>
    /// Состояние результата.
    /// </summary>
    [JsonPropertyName("status")]
    public RemoteMethodResultStatus Status { get; init; }

    /// <summary>
    /// Текст ошибки.
    /// </summary>
    [JsonPropertyName("error")]
    public string? Error { get; init; }

    /// <summary>
    /// Признак успешного результата.
    /// </summary>
    [JsonIgnore]
    public bool IsSuccess => Status == RemoteMethodResultStatus.Success;

    /// <summary>
    /// Признак ошибки.
    /// </summary>
    [JsonIgnore]
    public bool IsFailed => Status != RemoteMethodResultStatus.Success;

    /// <summary>
    /// Создаёт успешный ответ.
    /// </summary>
    public static RemoteMethodResult Ok() => new() { Status = RemoteMethodResultStatus.Success };

    /// <summary>
    /// Создаёт ответ с ошибкой.
    /// </summary>
    public static RemoteMethodResult Fail(string error) => new() { Status = RemoteMethodResultStatus.Fail, Error = error };

    /// <summary>
    /// Создаёт успешный ответ с данными.
    /// </summary>
    public static RemoteMethodResult<T> Ok<T>(T value) => new() { Status = RemoteMethodResultStatus.Success, Value = value };

    /// <summary>
    /// Создаёт ответ с ошибкой и типом данных.
    /// </summary>
    public static RemoteMethodResult<T> Fail<T>(string error) => new() { Status = RemoteMethodResultStatus.Fail, Error = error };

    ...
}

/// <summary>
/// Результат ответа удалённого метода с данными.
/// </summary>
public record RemoteMethodResult<T> : RemoteMethodResult
{
    /// <summary>
    /// Данные ответа.
    /// </summary>
    [JsonPropertyName("value")]
    public T? Value { get; init; }
}
```

Rules:

- use `System.Text.Json` attributes;
- keep result payload small;
- use `RemoteMethodResult<T>` only for small response values;
- do not return large DTOs, lists, logs, files, or binary data;
- for large data, return an id or status and fetch through HTTP API;
- public members must have Russian XML comments.

```csharp
/// <summary>
/// Статус ответа удалённого метода.
/// </summary>
public enum RemoteMethodResultStatus
{
    /// <summary>
    /// Неизвестно.
    /// </summary>
    Undefined = 0,

    /// <summary>
    /// Успешно.
    /// </summary>
    Success = 1,

    /// <summary>
    /// Ошибка.
    /// </summary>
    Fail = 2
}
```

## WebSocket Client Extensions

Use extension methods around `IHubContext<THub>` for server-to-client request/response calls when this is the project style.

```csharp
/// <summary>
/// Методы расширения для работы с WebSocket клиентом.
/// </summary>
public static class WebSocketClientExtensions
{
    /// <summary>
    /// Получить данные от клиента.
    /// </summary>
    public static async Task<RemoteMethodResult<TResponse>> GetDataAsync<THub, TResponse>(
        this IHubContext<THub> context,
        WebSocketConnectionRegistry connectionRegistry,
        string hostId,
        string methodName,
        CancellationToken cancellationToken)
        where THub : Hub
    {
        try
        {
            var connection = connectionRegistry.GetConnection(hostId);
            if (string.IsNullOrEmpty(connection))
                return RemoteMethodResult.Fail<TResponse>($"Не удалось {methodName}, соединение не найдено");

            return await context
                .Clients
                .Client(connection)
                .InvokeAsync<RemoteMethodResult<TResponse>>(methodName, cancellationToken);
        }
        catch (Exception exception)
        {
            return RemoteMethodResult.Fail<TResponse>(exception.Message);
        }
    }

    /// <summary>
    /// Отправить сигнал клиенту.
    /// </summary>
    public static async Task<RemoteMethodResult> SendSignalAsync<THub>(
        this IHubContext<THub> context,
        WebSocketConnectionRegistry connectionRegistry,
        string hostId,
        string methodName,
        CancellationToken cancellationToken)
        where THub : Hub
    {
        try
        {
            var connection = connectionRegistry.GetConnection(hostId);
            if (string.IsNullOrEmpty(connection))
                return RemoteMethodResult.Fail($"Не удалось {methodName}, соединение не найдено");

            await context.Clients.Client(connection).SendAsync(methodName, cancellationToken);

            return RemoteMethodResult.Ok();
        }
        catch (Exception exception)
        {
            return RemoteMethodResult.Fail(exception.Message);
        }
    }

    /// <summary>
    /// Отправить сигнал клиенту с данными.
    /// </summary>
    public static async Task<RemoteMethodResult> SendSignalAsync<THub, TRequest>(
        this IHubContext<THub> context,
        WebSocketConnectionRegistry connectionRegistry,
        string hostId,
        string methodName,
        TRequest request,
        CancellationToken cancellationToken)
        where THub : Hub
    {
        try
        {
            var connection = connectionRegistry.GetConnection(hostId);
            if (string.IsNullOrEmpty(connection))
                return RemoteMethodResult.Fail($"Не удалось {methodName}, соединение не найдено");

            await context.Clients.Client(connection).SendAsync(methodName, request, cancellationToken);

            return RemoteMethodResult.Ok();
        }
        catch (Exception exception)
        {
            return RemoteMethodResult.Fail(exception.Message);
        }
    }

    ...
}
```

Rules:

- async extension methods must end with `Async`;
- use `GetDataAsync` for request/response remote calls;
- use `SendSignalAsync` for send-only micro-signals where no client response is needed;
- do not use send-only signals when the server must know whether the client completed the action;
- always await `InvokeAsync` and `SendAsync`;
- always pass `CancellationToken`;
- return `RemoteMethodResult` or `RemoteMethodResult<T>` instead of throwing for expected remote-call failures;
- if connection is missing, return failed result with a clear Russian error;
- catch unexpected invoke errors and convert to failed result;
- do not log or expose sensitive exception details unless the project allows it;
- keep helpers focused on transport invocation, not business logic;
- send only small payloads;
- prefer method-name constants instead of magic strings.

## Handlers Usage

App handlers may use the hub client abstraction.

```csharp
/// <summary>
/// Обработчик создания сообщения.
/// </summary>
public sealed class CreateMessageCommandHandler(
    IMessageRepository messageRepository,
    ITerminalHubClient terminalHubClient)
    : ICommandHandler<CreateMessageCommand, Result<Guid>>
{
    /// <inheritdoc />
    public async ValueTask<Result<Guid>> Handle(
        CreateMessageCommand command,
        CancellationToken cancellationToken)
    {
        ...

        await terminalHubClient.NotifyMessageCreatedAsync(messageId, cancellationToken);

        return Result.Ok(messageId);
    }
}
```

Rules:

- handler calls App interface;
- handler must not know about SignalR;
- SignalR notification should not replace persistence;
- persist first, notify after successful domain operation;
- if `TransactionBehavior` is used, avoid sending irreversible notifications before commit unless the project has an outbox or post-commit mechanism;
- for strict consistency, prefer outbox or post-commit notification.

## Payload Rules

Allowed payloads:

- ids;
- small status values;
- small command or event names;
- timestamps;
- version numbers;
- correlation ids.

Avoid payloads:

- full entity graphs;
- collections;
- tables;
- files;
- logs;
- base64;
- binary blobs;
- large JSON.

If payload grows, send an id and let the client load data through HTTP API.

## Logging

Use structured logging.

Good:

```csharp
logger.LogInformation("SignalR notification sent");
```

Better with scope tags:

```csharp
using var scope = logger
    .Prepare()
    .AddTerminalId(terminalId);

logger.LogInformation("SignalR refresh requested");
```

Bad:

```csharp
logger.LogInformation($"SignalR refresh requested for {terminalId}");
```

Rules:

- message templates are constant strings;
- dynamic values go to structured parameters or logger scope tags;
- do not log large payloads;
- do not log secrets or tokens.

## Frontend Client Rules

Frontend SignalR handlers should also treat messages as small events.

Good:

```ts
connection.on('MessageCreated', async (messageId: string) => {
    await MessagesManager.loadMessageAsync(messageId)
})
```

Bad:

```ts
connection.on('MessageCreated', (message: FullMessageDto) => {
    MessagesStore.setMessage(message)
})
```

Rules:

- receive small events;
- fetch full data through API when needed;
- do not put large DTOs into SignalR event payloads;
- keep event handler logic thin;
- delegate work to managers, stores, or API clients.

## Testing

Test:

- App handlers call hub client interface when notification is required;
- hub client sends correct SignalR method name;
- hub client sends correct group;
- hub methods do not contain business logic.

Mock `ITerminalHubClient` in App tests.

Do not depend on real SignalR connections in unit tests.

## Final Validation

Before completing a SignalR task, verify:

- payloads are tiny;
- full data is loaded through HTTP API;
- hub implementation lives in Api;
- hub client implementation lives in Api;
- hub client interface lives in App;
- App does not reference SignalR types;
- Domain does not reference SignalR;
- hub methods contain no business logic;
- SignalR method names are constants;
- group names are centralized;
- connection lifecycle tracks and removes connection ids when needed;
- connection registry contains no business logic;
- registry cleanup happens on disconnect;
- server-to-client request/response calls use `GetDataAsync`;
- send-only micro-signals use `SendSignalAsync`;
- extension methods end with `Async`;
- `InvokeAsync` and `SendAsync` are awaited;
- missing connection returns failed `RemoteMethodResult`;
- remote-call failures do not throw for expected failures;
- `RemoteMethodResult` payloads remain small;
- async methods end with `Async`;
- tasks and promises are awaited;
- XML comments are present and written in Russian for public C# API;
- SignalR options are read from `appsettings`, options, or app config;
- no SignalR timing or security values are hardcoded;
- hub endpoint is mapped in Api;
- hub endpoint route starts with `/hub`;
- WebSockets transport is configured when required;
- custom `IUserIdProvider` is registered when hub clients need domain-specific user ids;
- host or terminal hubs use the correct authentication scheme;
- JWT settings are read from configuration;
- public hub methods have Russian XML comments;
- public hub methods end with `Async`;
- `[HubMethodName]` is used when wire method names must stay stable;
- hub methods delegate to handlers or services;
- handlers are awaited;
- `CancellationToken` is propagated;
- App interfaces do not expose SignalR-specific types.
