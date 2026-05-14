# `docs/services/` — Deep-dive per-service

Mỗi file dưới đây mô tả chi tiết 1 service backend với cấu trúc nhất quán:

1. Cấu trúc thư mục
2. Aggregate root + bảng
3. Use case chi tiết (Command/Query) — Actor, Validator, Business rule, Side-effect, Event
4. Integration event publish/subscribe
5. Dependency
6. Logic đặc thù
7. Gap / improvement
8. Migration Rust note (nếu áp dụng)
9. Cross-link

Đọc cùng với [../02-backend-feature-catalog.md](../02-backend-feature-catalog.md) (tổng quan + bảng tóm tắt) và [../04-database-schema.md](../04-database-schema.md) (schema chi tiết).

## Danh sách service

| Service | File | Phức tạp | Bảng SQL |
|---|---|---|---|
| **Auth & Identity** | | | |
| IdentityServer | [identity-server.md](identity-server.md) | Trung | ~20 (IS4 std) |
| **Cốt lõi nghiệp vụ** | | | |
| UserService | [user-service.md](user-service.md) | Cao | 64 |
| CourseService | [course-service.md](course-service.md) | Cao | 44 |
| QuestionService | [question-service.md](question-service.md) | Rất cao | 59 |
| TrainingRouteService | [training-route-service.md](training-route-service.md) | Trung | 11 |
| **Shared & Config** | | | |
| SharedServices | [shared-services.md](shared-services.md) | Trung | 40 |
| SystemService | [system-service.md](system-service.md) | Trung | 22 |
| **Realtime & Messaging** | | | |
| SignalR | [signalr.md](signalr.md) | Cao | (no DB) |
| CommunicationService | [communication-service.md](communication-service.md) | Thấp | Mongo |
| NotificationService | [notification.md](notification.md) | Thấp | Mongo |
| MailSender | [mail-sender.md](mail-sender.md) | Thấp | (no DB) |
| **Infrastructure** | | | |
| LogService | [log-service.md](log-service.md) | Thấp | Mongo |
| ServerFiles | [server-files.md](server-files.md) | Trung | 5 |
| ReportService | [report-service.md](report-service.md) | Trung | (read-only) |
| APIGateway | [api-gateway.md](api-gateway.md) | Thấp | (no DB) |
| **Shared library** | | | |
| CLS4.0-Core | [core-library.md](core-library.md) | (foundation) | - |

## Ánh xạ với pha migration Rust

(Tham chiếu [../07-rust-migration-plan.md](../07-rust-migration-plan.md))

| Pha | Service deep-dive liên quan |
|---|---|
| Pha 0 — Chuẩn bị | [core-library.md](core-library.md) làm template cho `cls-rust/crates/cls-core` |
| Pha 1 — PoC | [mail-sender.md](mail-sender.md) |
| Pha 2 — Stateless IO | [notification.md](notification.md), [log-service.md](log-service.md) |
| Pha 3 — Realtime + file | [server-files.md](server-files.md), [signalr.md](signalr.md) |
| Pha 4 — Outer ring | [shared-services.md](shared-services.md), [system-service.md](system-service.md), [communication-service.md](communication-service.md), [report-service.md](report-service.md) |
| Pha 5 — Core nghiệp vụ | [training-route-service.md](training-route-service.md), [question-service.md](question-service.md), [course-service.md](course-service.md), [user-service.md](user-service.md) |
| Pha 6 — Replace IdentityServer | [identity-server.md](identity-server.md) → Keycloak |
| Pha 7 — Replace Gateway | [api-gateway.md](api-gateway.md) → Pingora/Envoy |
