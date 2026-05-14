# MailSender — Deep Dive

> Path: `CLS4.0-MailSender/src/EmailSender/`. Background worker — không có Controller public.
>
> Vai trò: consume `EmailEvent` từ RabbitMQ, render template (lấy từ SystemService), gửi SMTP qua Hangfire background job.

## 1. Pattern: RabbitMQ + Hangfire

```
Service X → RabbitMQ (EmailIntegrationEvent)
              ↓ subscribe
MailSender consume handler
              ↓ BackgroundJob.Enqueue
Hangfire queue (MemoryStorage)
              ↓ background server
EmailService.SendMail()
              ↓ SmtpClient
SMTP server
```

## 2. Cấu hình Hangfire

- **Storage**: `MemoryStorage` (in-process — chỉ phù hợp single-server).
- Setup: `services.AddHangfire(...).AddHangfireServer()`.
- Compatibility level: `Version_170`.
- Type serializer: `SimpleAssemblyNameTypeSerializer` + `DefaultTypeSerializer`.
- Auto retry: default 3 attempts với exponential backoff.

## 3. Cấu hình SMTP

`EmailService` dùng .NET `SmtpClient` chuẩn:
- Đọc `EmailConfigurationDto`: `SenderEmail`, `Password`, `Host`, `Port`, `IsRequireSsl`, `Username`.
- HTML body, UTF-8 encoding.
- Display name = `Username` field.
- **Thread safety**: `lock (Program.Resource)` ép gửi sequential, delay 1000ms giữa các email (tránh SMTP rate-limit).
- Error: log thành công/lỗi, swallow exception (không throw lên).

## 4. Template rendering

**Không** dùng Razor / Liquid engine. Cách hiện tại:
- Template HTML lấy từ SystemService qua HTTP (`/api/emailtemplate/...`).
- Body đã render gần xong upstream — Hoặc:
- `IMailInfoCollector` gom field từ event data + replace placeholder trong template.
- Field types dùng enum `TemplateFields` + package `Microservices.Languages` cho multi-lang.

Variable phổ biến:
- `{{user_name}}`, `{{user_email}}`
- `{{course_name}}`, `{{course_url}}`
- `{{exam_name}}`, `{{exam_start_time}}`
- `{{certificate_url}}`
- `{{portal_name}}`, `{{portal_url}}`
- `{{reset_password_link}}`, `{{confirm_email_link}}`

## 5. Subscribed integration events (~12 handler)

| Handler | Event | Mục đích |
|---|---|---|
| `EmailIntegrationEventHandler` | `EmailIntegrationEvent` | Generic email handler |
| `SendMailVersion2EventHandler` | `SendMailVersion2Event` | Handler V2 với schema mới |
| `ScheduleEmailEventHandler` | `ScheduleEmailEvent` | Lên lịch gửi mai/sau N giờ (reminder course/exam) |
| `DeleteScheduleEmailEventHandler` | `DeleteScheduleEmailEvent` | Huỷ schedule |
| `UpdateUserEventHandler` | `UserUpdatedIntegrationEvent` | Gửi mail thông báo cập nhật |
| `DeleteUserEventHandler` | `UserDeletedIntegrationEvent` | Gửi mail thông báo xoá |
| `UpdateCourseEventHandler` | `CourseUpdatedIntegrationEvent` | Course updated → notify learner |
| `UpdateExamTestEventHandler` | `ExamUpdatedIntegrationEvent` | Exam updated |
| `DeleteExamTestEventHandler` | `ExamDeletedIntegrationEvent` | Exam deleted |
| `UpdatePortalEventHandler` | `PortalUpdatedIntegrationEvent` | Portal updated |
| `DeletePortalEventHandler` | `PortalDeletedIntegrationEvent` | Portal deleted |
| `UpdateNotificationConfigEventHandler` | `NotificationConfigUpdatedEvent` | Cache invalidate |

## 6. Logic per handler tiêu biểu

### EmailIntegrationEventHandler
```
1. Receive event với data: { user_email, event_type, payload }
2. Validate email format
3. Query NotificationConfig (System Service) qua INotificationConfigQueries
   - Check user có cho phép nhận mail event_type này không
   - Get LanguageCode preferred
4. Cache portal settings qua IPortalRedisService (Redis)
5. Query EmailTemplate theo (event_type, portal_id, language)
6. Render template với payload data
7. BackgroundJob.Enqueue(() => EmailService.SendMail(settings))
```

### ScheduleEmailEventHandler
```
1. Lưu schedule vào DB (deferred email)
2. Hangfire delayed job: BackgroundJob.Schedule(...)
3. Khi tới giờ → execute SendMail
```

## 7. Retry & reliability

| Cấp | Mechanism |
|---|---|
| Hangfire | Auto retry 3 lần, exponential backoff |
| RabbitMQ | Polly retry (CLS Core EventBus, count = `EventBus:RetryCount`) |
| SMTP | Lock + 1000ms delay tránh rate-limit |
| Filter pre-enqueue | Validate email + notification preference trước enqueue (tiết kiệm) |
| Cache | Portal settings qua Redis để giảm DB hit |

## 8. Dependency

- RabbitMQ (broker `micro_event_bus`).
- SystemService (HTTP query template).
- UserService (qua event hoặc HTTP query user preference).
- SMTP server (`MailSettings:Server`, `:Port`, `:User`, `:Password`, `:From`, `:IsRequireSsl`).
- Hangfire (queue + retry + dashboard tuỳ chọn).
- Redis (cache portal settings).

## 9. Logic đặc thù

### Hangfire MemoryStorage limitation
- Job lưu RAM → restart pod = mất queue.
- **Improvement**: chuyển sang `Hangfire.SqlServer` hoặc `Hangfire.Redis` để persistent.

### Sequential send với lock
- `lock (Program.Resource)` ép gửi 1 mail/giây.
- Mục đích: tránh SMTP rate-limit (Gmail/Office365 throttle).
- Trade-off: throughput thấp (~3600 mail/giờ). Khi cần spike → bottleneck.

### Multi-language
- `Microservices.Languages` package multi-lang.
- Template chọn theo `LanguageCode` của user.

## 10. Gap / improvement

- `MemoryStorage` → mất queue khi restart. Chuyển SQL/Redis storage.
- Sequential lock → throughput thấp. Chuyển batch send qua nhiều worker, dùng SMTP relay (SendGrid, SES) thay vì self-hosted.
- Template rendering string replace đơn giản → không support condition / loop. Nên dùng Liquid / Razor.
- Không có dead-letter queue cho event không gửi được sau retry.
- Không tracking open/click rate.

## 11. Migration sang Rust note

- Hangfire → **Apalis** (Rust job queue) với Redis backend (persistent).
- SMTP → **lettre** crate.
- Template engine → **tera** (Jinja-like).
- Multi-lang → **fluent** (Mozilla) hoặc tự build.

## 12. Cross-link

- [system-service.md](system-service.md) — provider EmailTemplate.
- [notification.md](notification.md) — kênh push fallback.
- [user-service.md](user-service.md) — user preference.
- [../07-rust-migration-plan.md](../07-rust-migration-plan.md#pha-1--poc-service-đơn-giản-1-2-tháng) — Pha 1 PoC chọn MailSender.
