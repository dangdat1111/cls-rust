# APIGateway — Deep Dive

> Path: `CLS4.0-APIGateway/src/APIGateway/`. Endpoint public: `lmsapi.doffice.com.vn:8080`.
>
> Ocelot gateway routing + JWT validate + Swagger aggregate + Eureka service discovery.

## 1. Cấu trúc

```
APIGateway/
├── Program.cs (Autofac + Serilog setup)
├── Startup.cs (Configure Ocelot middleware)
├── ocelot.json (route config) — có thể load từ Consul
└── ...
```

## 2. Middleware pipeline

```
Request
  ↓ NGINX Ingress (TLS, CORS, body limit)
  ↓ APIGateway
    ├── Serilog request logging
    ├── CORS (AddCustomCors)
    ├── JWT Bearer Authentication
    ├── Ocelot routing
    └── (downstream service)
```

## 3. Ocelot configuration

### Route config (ocelot.json hoặc Consul KV)
```json
{
  "Routes": [
    {
      "UpstreamPathTemplate": "/user/{everything}",
      "UpstreamHttpMethod": ["Get", "Post", "Put", "Delete"],
      "DownstreamPathTemplate": "/api/{everything}",
      "DownstreamHostAndPorts": [
        { "Host": "user-api-service", "Port": 80 }
      ],
      "AuthenticationOptions": {
        "AuthenticationProviderKey": "IgnoreExpirationTime",
        "AllowedScopes": []
      }
    },
    // ... routes cho từng service
  ],
  "SwaggerEndPoints": [...],
  "GlobalConfiguration": {
    "BaseUrl": "https://lmsapi.doffice.com.vn"
  }
}
```

### Service discovery (Eureka)
- `AddOcelot().AddEureka()` → Ocelot resolve service host từ Eureka registry.
- Có thể fallback K8s DNS service nếu Eureka down.

## 4. Authentication

JWT Bearer config (`AddCustomAuthentication`):
```csharp
options.TokenValidationParameters = new TokenValidationParameters {
    ValidateAudience = false,
    RequireExpirationTime = false,
    ValidateLifetime = false,   // ⚠️ KHÔNG validate expiry
    ValidateIssuer = true,
    ValidIssuer = "https://identityserver..."
};
```

**Cảnh báo bảo mật**: `ValidateLifetime = false` ở Gateway cho phép expired token đi qua. Service phía sau phải tự validate expiry. **Khuyến nghị fix**: bật `ValidateLifetime = true` để reject expired token ngay ở Gateway.

Scheme name: `IgnoreExpirationTime` — phản ánh design hiện tại nhưng nên rename khi fix.

## 5. Swagger aggregation

- `Development/Frontend` env: bật `SwaggerForOcelotUI`.
- Aggregate upstream service swagger ở `/swagger/docs`.
- Mỗi service expose `/swagger/v1/swagger.json` → gateway fetch + merge.

## 6. Rate limiting

Cấu hình trong `GlobalConfiguration.RateLimitOptions`:
```json
{
  "ClientWhitelist": [],
  "EnableRateLimiting": true,
  "Period": "1m",
  "PeriodTimespan": 60,
  "Limit": 1000
}
```
(Giá trị tùy chỉnh per portal nếu cần)

## 7. Logging

- Serilog enrich với CorrelationId.
- Mỗi request log: method, path, downstream service, status, latency.
- Sink: Elasticsearch ở production.

## 8. CORS

`AddCustomCors`:
```csharp
options.AddPolicy("CorsPolicy", builder => {
    builder.WithOrigins("https://portal.doffice.com.vn", ...)
           .AllowAnyMethod()
           .AllowAnyHeader()
           .AllowCredentials();
});
```

Headers cho phép: `Authorization`, `Content-Type`, `Portal` (custom header dùng cho multi-tenant routing).

## 9. Logic đặc thù

### Multi-tenant via header
- Request từ frontend có header `Portal: <portal-domain>`.
- Gateway forward header xuống service.
- Service đọc header để biết portal context (kết hợp claim trong JWT).

### Aggregation routes
- Một số endpoint frontend cần data từ nhiều service → Ocelot aggregate route trả về object compose.
- Ví dụ: `/aggregate/dashboard` → call Course/User/Question/aggregate trả 1 JSON.

## 10. Dependency

- Eureka (service discovery).
- Consul (config — có thể load ocelot.json từ Consul).
- IdentityServer (validate JWT issuer).
- Tất cả service backend (downstream).

## 11. Gap / improvement

- `ValidateLifetime = false` → **bảo mật risk cao**. Fix: bật validate lifetime.
- Eureka phụ thuộc thêm 1 component runtime → khi migrate Rust nên bỏ Eureka, dùng K8s DNS.
- Ocelot performance vừa phải — khi migrate Rust → Pingora/Envoy cho throughput tốt hơn.
- Rate limit per-IP không đủ cho multi-tenant — cần rate limit theo `PortalId` claim.
- Swagger aggregate gặp lỗi khi 1 service down → fail open. Cần fallback.

## 12. Migration Rust note

- Web framework: **Pingora** (Cloudflare) cho hiệu năng tối đa, hoặc **Envoy** (config-driven, dễ vận hành).
- Service discovery: K8s DNS thay Eureka.
- Swagger aggregate: tự build tool aggregate từ `utoipa` spec mỗi service.
- JWT: `jsonwebtoken` crate.

## 13. Cross-link

- [identity-server.md](identity-server.md) — issuer JWT.
- [core-library.md](core-library.md) — middleware shared.
- [../05-deployment-operations.md](../05-deployment-operations.md#3-ingress-nginx) — ingress trước Gateway.
- [../07-rust-migration-plan.md](../07-rust-migration-plan.md#pha-7--replace-gateway-1-2-tháng) — Pha 7 replace Gateway.
