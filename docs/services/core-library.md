# CLS4.0-Core — Deep Dive (Shared Library)

> Path: `CLS4.0-Core/src/Core/`. Internal NuGet/project library được mọi service backend reference.
>
> Cung cấp foundation: EventBus, Repository, JWT, MediatR Behaviors, Consul config, Healthcheck, JsonResponse.

## 1. Cấu trúc namespace chính

```
Core/
├── EventBus/
│   ├── IEventBus.cs
│   ├── IIntegrationEventHandler.cs
│   ├── IntegrationEvent.cs
│   └── RabbitMQ/
│       ├── EventBusRabbitMQ.cs
│       ├── DefaultRabbitMQPersistentConnection.cs
│       └── IRabbitMQPersistentConnection.cs
├── Domain/SeedWork/
│   ├── Entity.cs
│   ├── AggregateRoot.cs
│   ├── Enumeration.cs
│   ├── IRepository.cs
│   ├── IUnitOfWork.cs
│   └── ISoftDelete.cs
├── Infrastructure/
│   ├── IDbContextCore.cs
│   ├── BaseDbContext.cs
│   └── Repositories/Repository.cs
├── API/
│   ├── Controllers/CustomController.cs
│   ├── Controllers/ClsRequestHandler.cs
│   ├── Middleware/ (Exception, Jwt, CorrelationId)
│   ├── Filters/AuthorizeFilter.cs
│   └── Behaviors/ (Logging, Validator, Transaction)
├── Authentication/
│   ├── JwtHandler.cs
│   └── JwtConfigurationExtensions.cs
├── Discovery/
│   └── EurekaConfigurationExtensions.cs
├── Configuration/
│   └── ConfigurationBuilderExtensions.cs (Consul loader)
├── Logging/
│   └── SerilogConfigurationExtensions.cs
├── Healthcheck/
├── Models/Response/
│   └── JsonResponse.cs
└── Common/
    └── (Extension, Helper)
```

## 2. EventBus

### IEventBus
```csharp
public interface IEventBus {
    void Publish(IntegrationEvent @event);
    void Subscribe<T, TH>() where T: IntegrationEvent where TH: IIntegrationEventHandler<T>;
    void Unsubscribe<T, TH>() where T: IntegrationEvent where TH: IIntegrationEventHandler<T>;
}
```

### EventBusRabbitMQ
| Method | Logic |
|---|---|
| `Publish(@event)` | Serialize event với Newtonsoft.Json → publish vào exchange `micro_event_bus` (type=direct) với routing key = event name; Polly retry (exponential backoff 2^N seconds); handle `BrokerUnreachableException`, `SocketException` |
| `Subscribe<T, TH>()` | Register handler trong `_subsManager`; bind queue (`_queueName` = `SubscriptionClientName` từ config) với routing key |
| `Unsubscribe<T, TH>()` | Remove handler khỏi `_subsManager` |
| `DoInternalSubscription` | Tạo consumer channel; xử lý message deserialize + dispatch tới handler qua Autofac scope |

Fields:
- `_autofac` (DI container)
- `_consumerChannel` (RabbitMQ channel)
- `_queueName`, `_subsManager`, `_retryCount`

Configuration via `EventBus:Connection`, `EventBus:UserName`, `EventBus:Password`, `EventBus:RetryCount`, `EventBus:SubscriptionClientName`.

### IntegrationEvent
Base class với:
- `Id (Guid)` — unique event id
- `CreationDate (DateTime)` — UTC
- (Convention thêm) `PortalId`, `UserId`, `CorrelationId`

## 3. Domain SeedWork

### IRepository<T>
- Generic CRUD interface cho aggregate root.
- `IUnitOfWork` chứa `SaveEntitiesAsync` (commit + dispatch domain event).

### Repository<T>
| Method | Mục đích |
|---|---|
| `AddRangeAsync(entities)` | Bulk add |
| `BulkAddRangeAsync(entities, config)` | EFCore.BulkExtensions bulk insert |
| `Update(entity)` | Update |
| `Delete(id)` [Obsolete] | Delete |
| `GetAsync(id)` | Get by Id |
| `GetAllAsync()` | Get all |
| `FindAsync(expression)` | LINQ query |
| `SaveChangesAsync()` | Persist |

Base entity: `EntityBase` (có `Id`). Hỗ trợ `ISoftDelete` — filter `IsDeleted=false` mặc định ở base repository.

### Bulk performance
- `EFCore.BulkExtensions` cho insert/update/delete số lượng lớn (>1000).
- Bypass change tracking → nhanh hơn standard EF.

## 4. CustomController + ClsRequestHandler

### CustomController
```csharp
public abstract class CustomController : Controller {
    protected async Task<IActionResult> ExecuteCommand<T>(T command, CancellationToken ct)
        where T: IRequest<JsonResponse<...>> {
        var response = await _mediator.Send(command, ct);
        return CustomResult(response);
    }
    
    protected IActionResult CustomResult<T>(JsonResponse<T> response) {
        return response.IsSuccess ? Ok(response) : BadRequest(response);
    }
}
```

Mọi controller backend kế thừa CustomController + thực hiện thin controller pattern (chỉ call ExecuteCommand).

### ClsRequestHandler
Base command handler với DI sẵn:
- `IUnitOfWork`
- `ISystemActivityLogger`
- `IEventBus`
- `IHttpContextAccessor`
- `IClaimManager` (trích claim từ JWT)

## 5. MediatR Behaviors

### LoggingBehavior<TRequest, TResponse>
- Log `"Handling command {CommandName}"` trước handler.
- Log `"Command {CommandName} handled"` sau.
- Capture duration + exception nếu có.

### ValidatorBehavior<TRequest, TResponse>
| Bước | Logic |
|---|---|
| 1 | Lookup tất cả `IValidator<TRequest>` trong DI |
| 2 | Validate request, gom errors |
| 3 | Extract claims từ `HttpContext.User`: `ManagementId`, `OwnerId`, `LanguageCode`, `UserId` |
| 4 | Set vào `RequestBase` properties (TRequest kế thừa RequestBase) |
| 5 | Nếu có lỗi validation → return `BadRequest` response với detail `"Core_DataValidationError"` |
| 6 | Nếu OK → invoke next handler |

### TransactionBehavior
- Wrap handler trong `IDbContextTransaction`.
- Commit nếu success, rollback nếu exception.
- Áp dụng cho command (không cho query).

## 6. ConfigurationBuilderExtensions (Consul loader)

```csharp
public static IConfigurationBuilder AddConfigurationFromHost(
    this IConfigurationBuilder builder,
    string hostAddress,
    string[] keys) {
    foreach (var key in keys) {
        builder.AddConsul(key, options => {
            options.ConsulConfigurationOptions = cco => {
                cco.Address = new Uri(hostAddress);
            };
            options.ReloadOnChange = true;
            options.OnLoadException = ctx => ctx.Ignore = true;
        });
    }
    return builder;
}
```

Logic:
- Load config từ Consul KV theo list keys (`<service-name>/<env>`).
- `ReloadOnChange = true` — config thay đổi tự reload, không cần restart.
- `OnLoadException.Ignore = true` — graceful nếu Consul tạm thời unavailable.

## 7. Authentication

### JwtConfigurationExtensions
- Setup JWT Bearer middleware.
- Token validation: issuer (IdentityServer), audience, signing key.
- Cấu hình `MapInboundClaims = false` để giữ claim type gốc.

### JwtHandler / AuthorizeJwtAttribute / IClaimManager
- Trích claims từ token đã validated.
- Convenience getters: `GetUserId()`, `GetPortalId()`, `GetManagementId()`, `GetFeatures()`.

## 8. Discovery

### EurekaConfigurationExtensions
- `services.AddDiscoveryClient(Configuration)` register Steeltoe Eureka client.
- Service tự register vào Eureka lúc start với metadata (port, healthcheck URL).
- Default Eureka URL: `cls4-support:8761`.

## 9. Healthcheck extensions

```csharp
public static IServiceCollection AddCheckMongoDb(this IServiceCollection services, ...)
public static IServiceCollection AddCheckRabbitmq(this IServiceCollection services, ...)
public static IServiceCollection AddCheckSqlServer(this IServiceCollection services, ...)
```

Mỗi service register checker tương ứng dependency của nó. Endpoint `/hc` expose tổng hợp status.

## 10. Logging

### SerilogConfigurationExtensions
- Setup Serilog với enricher: `MachineName`, `ThreadId`, `CorrelationId`, `UserId`.
- Sink Console (luôn) + Elasticsearch (production, theo `ElasticConfiguration:Uri`).
- Index pattern: `cls-{service}-{date}`.

## 11. JsonResponse<T>

```csharp
public class JsonResponse<T> {
    public bool IsSuccess { get; set; }
    public T Data { get; set; }
    public List<ErrorDetail> Errors { get; set; }
    public List<string> Messages { get; set; }
    public int? StatusCode { get; set; }
}
```

Mọi API endpoint backend trả về `JsonResponse<T>` chuẩn. Frontend axios interceptor parse uniform.

## 12. Common utilities (Extension)

- `HttpContextExtensions.GetClaim(name)` — trích claim từ JWT.
- `ExceptionExtensions.GetErrorDetails()` — format exception thành ErrorDetail.
- `ValidationExceptionExtensions` — convert FluentValidation → JsonResponse.
- `GenericTypeExtensions` — type reflection.
- `StringExtensions` — helpers (slugify, mask).
- `DateExtensions` — VN timezone, format.

## 13. ServiceCollectionExtensions (DI helpers)

| Method | Mục đích |
|---|---|
| `AddMicroserviceEnvironment(Configuration)` | Bundle: MediatR + Behaviors + JsonOptions + healthcheck + auth |
| `AddHangfireJobService(Configuration)` | Hangfire setup (storage tuỳ env) |
| `AddCustomAuthentication(Configuration)` | JWT setup |
| `AddSwaggerGen(...)` | Swagger config |
| `AddLogService(Configuration)` | Serilog setup |

## 14. RequestBase

```csharp
public abstract class RequestBase {
    public Guid ManagementId { get; set; }
    public Guid OwnerId { get; set; }
    public Guid UserId { get; set; }
    public string LanguageCode { get; set; }
}
```

Mọi Command/Query đều kế thừa RequestBase. `ValidatorBehavior` tự inject từ JWT claim → handler không cần đọc `HttpContext`.

## 15. Migration Rust mapping

| .NET Core class | Rust equivalent |
|---|---|
| `IEventBus` + RabbitMQ | trait `EventBus` + impl với `lapin` |
| `IRepository<T>` + EF Core | trait `Repository<T>` + `sqlx` |
| MediatR + Behaviors | Custom dispatcher + tower middleware layers |
| `CustomController` | Axum handler + extractor |
| `ClsRequestHandler` | Generic command handler trait |
| `IUnitOfWork` | `sqlx::Transaction` wrapper |
| `JwtHandler` | `jsonwebtoken` crate |
| `Consul config loader` | `consul-rs` + `config` |
| Serilog | `tracing` + `tracing-subscriber` |
| Healthcheck | `axum-health` custom |
| `JsonResponse<T>` | Generic struct với `serde` |

## 16. Cross-link

- Mọi service backend reference library này.
- [../07-rust-migration-plan.md](../07-rust-migration-plan.md#pha-0--chuẩn-bị-1-tháng) — Pha 0 build `cls-rust/crates/cls-core` tương đương.
- [../05-deployment-operations.md](../05-deployment-operations.md) — config Consul KV.
