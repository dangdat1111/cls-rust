# IdentityServer — Deep Dive

> Path: `CL4.0-IdentityServer/`. Tech: IdentityServer4 trên .NET Core 3.1. Vai trò: authorization server OAuth2/OIDC cho toàn hệ thống.

## 1. DbContext

| DbContext | Bảng | Mục đích |
|---|---|---|
| `ConfigurationDbContext` | `Clients*`, `ApiResources*`, `ApiScopes*`, `IdentityResources*` | Cấu hình client + scope + resource |
| `PersistedGrantDbContext` | `PersistedGrants`, `DeviceCodes` | Lưu token/code do cấp (auto-cleanup bật) |

Note: User credential **không** lưu ở IdentityServer DB — query trực tiếp UserService DB (`IdentityConnectionString` trỏ về UserService DB hoặc shared schema).

## 2. ResourceOwnerPasswordValidator

Class chính chịu trách nhiệm validate username/password khi grant `password`.

| Bước | Logic |
|---|---|
| 1. Đọc context | `PortalHelper` trích `Portal`, `Branch`, `Domain`, `PrivateDomain` từ header `Portal` + URL `Referer` |
| 2. Lookup user | `AccountQueries` tìm user theo email/username **trong scope portal** (multi-tenant) |
| 3. Validate password | Gọi `UserService.CheckPasswordAsync(user, password)` — so sánh hash |
| 4. Check email confirmed | Reject nếu `EmailConfirmed = false` |
| 5. Trả kết quả | `GrantValidationResult` với `Subject = userId`, `AuthenticationMethod = "password"` |
| 6. Log | Mọi attempt được log qua `ILogger` |

**Không có** captcha integration trong validator hiện tại (CLS frontend gọi `Captcha:SecretKey` ở phía UserService UI flow, không phải IdentityServer).

## 3. ProfileService

Bổ sung claim vào JWT.

| Method | Chức năng |
|---|---|
| `GetProfileDataAsync(context)` | Đọc `sub` claim → gọi `IUserService.GetClaimsAsync(userId, role)` → set `context.IssuedClaims`. Có đọc thêm header `Role` từ HTTP request để filter scope. |
| `IsActiveAsync(context)` | Check user vẫn tồn tại trong DB (cho phép force-revoke khi user bị xoá/lock). |

Claim được issue tiêu biểu:
- `sub` (userId), `email`, `name`, `preferred_username`
- `portalId`, `managementId`, `branchId`
- `role` (multi-value)
- `feature` (multi-value, tên feature bật)
- `auth_time` (epoch, dùng để force-logout khi đổi mật khẩu)

## 4. Logout

`HomeController.LogoutAsync(clientId, subjectId)`:
- Gọi `IPersistedGrantService.RemoveAllGrantsAsync(subjectId, clientId)` → xoá toàn bộ refresh token + authorization code của user-client.
- Publish `LogoutIntegrationEvent` qua RabbitMQ — SignalR consume để xoá Redis online map.

## 5. Endpoint chính

| Endpoint | Mục đích |
|---|---|
| `/connect/token` | `grant_type=password` / `refresh_token` / `client_credentials` |
| `/connect/authorize` | Auth code flow (chưa thấy social login chuẩn ở backend; social được handle ở UserService) |
| `/connect/userinfo` | Trả claims user |
| `/connect/endsession` | Logout, xoá session cookie |
| `/.well-known/openid-configuration` | Discovery |
| `/.well-known/jwks` | Public key (RS256) |

## 6. Cấu hình quan trọng

| Setting | Giá trị |
|---|---|
| Signing | Certificate (RSA) — load từ file hoặc Azure KeyVault |
| Token lifetime | Access token: 1 hour (default), refresh: 30 ngày |
| Token cleanup | `EnableTokenCleanup = true` (auto xoá PersistedGrant hết hạn) |
| CORS | Cho phép domain frontend |
| Cookie | Cấu hình SameSite phù hợp social flow nếu có |

## 7. Lỗi tiềm ẩn / improvement

- **Không có captcha trong validator** — bot có thể brute-force `/connect/token`. Nên thêm rate-limit hoặc captcha required sau N lần fail.
- **Token revocation cascade**: khi đổi password chưa thấy auto-revoke refresh token đã cấp → nên implement.
- **JWT signing cert rotation** — cần plan rotate cert định kỳ.
- **Social login** — hiện tại MSAL/Google login được xử lý ở UserService (frontend gọi `/api/authentication`) thay vì qua IdentityServer external provider flow chuẩn. Có thể consolidate.

## 8. Dependency

- SQL Server (IdentityServer config DB + access UserService DB qua `IdentityConnectionString`).
- RabbitMQ (publish `LogoutIntegrationEvent`).
- UserService (qua DB direct connection, không qua HTTP).

## 9. Cross-link

- [user-service.md](user-service.md) — provider của user credential.
- [signalr.md](signalr.md) — consume LogoutEvent.
- [../04-database-schema.md](../04-database-schema.md#9-identityserver-db) — schema chi tiết.
