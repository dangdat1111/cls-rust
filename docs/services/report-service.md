# ReportService — Deep Dive

> Path: `CLS4.0-ReportService/`.
>
> Service báo cáo tổng hợp — đa số query trực tiếp các DB (qua Dapper read-only) hoặc proxy gọi service tương ứng.

## 1. Cấu trúc thư mục

```
Report.API/
├── Program.cs (Serilog setup)
├── Startup.cs (AddCustomServices, AddCustomDbContext)
├── AutofacModules/ApplicationModule.cs (placeholder)
└── Controllers/ (queries qua MediatR)

Report.Infrastructure/
└── ApplicationDbContext.cs (EF Core)

Report.Domain/
└── (domain models)
```

## 2. Đặc điểm

- Service "mỏng" — chủ yếu compose query từ DB của các service khác (Course, User, Question, TrainingRoute).
- Có thể dùng **read replica** SQL Server để giảm load DB chính.
- Tích hợp Consul config (`UseCustomConfiguration`) như mọi service khác.
- MediatR CQRS pattern (chỉ Query, không Command).

## 3. Loại báo cáo chính

| Báo cáo | Source data | Logic |
|---|---|---|
| Course completion | Course DB + CourseUser DB | % user complete / tổng user assigned |
| User progress | User DB + CourseUser/Exam DB | timeline học của 1 user |
| Exam analytics | Question DB | Phân phối điểm, top câu hỏi sai |
| Training plan report | Course + Plan DB | Tỷ lệ hoàn thành theo plan |
| Training route report | TrainingRoute DB | Progress theo step |
| Cost report | Course + Plan + Route DB | Tổng chi phí đào tạo |
| Lecturer report | Course DB + User DB | Hiệu suất giảng viên (số khoá, rating, học viên) |
| Branch / Position / Qualification report | User DB | Báo cáo theo cơ cấu tổ chức |
| Custom report | SystemService config | Run dynamic query theo config |

## 4. Endpoint pattern

Báo cáo thường có 2 dạng endpoint:
- `GET /api/report/<type>/list?...` — paged data.
- `GET /api/report/<type>/export?...&format=excel|pdf` — export file.

## 5. Logic đặc thù

### Cross-service read
- Service Report **đọc trực tiếp** SQL Server của Course/User/Question (không gọi HTTP).
- Connection string trỏ về cùng SQL Server cluster, mỗi service-DB chỉ cấp read role.
- **Trade-off**: nhanh hơn HTTP nhưng tight-coupling schema.

### Export Excel
- EPPlus library.
- Stream output cho file lớn.

### Export PDF
- iText / PdfSharp render từ template.

### Materialized view / pre-compute
- Một số báo cáo nặng dùng materialized table (`TrainingRouteReportUser`, `LearningStepReportUser` ở TrainingRoute) được service nghiệp vụ tự update khi có event.

## 6. Custom Report builder

- Đọc config `CustomReport.Models` JSON từ SystemService.
- Build SQL theo config (filter, group by, aggregation).
- **Security risk**: nếu không sanitize → SQL injection. Cần whitelist allowed tables/columns.

## 7. Scheduled report

- Có thể tích hợp Hangfire cron để generate báo cáo định kỳ + push qua MailSender/Notification.
- Hiện tại implementation chưa rõ — improvement candidate.

## 8. Dependency

- SQL Server (multi-DB read).
- Consul (config).
- (Tuỳ chọn) Hangfire (scheduled report).
- EPPlus (Excel).
- PDF library.
- MailSender (gián tiếp khi báo cáo định kỳ).

## 9. Gap / improvement

- Tight-coupling schema → khi service nghiệp vụ migration schema, Report phải đồng bộ.
- Không có abstraction layer cho query → khó test.
- Custom report builder SQL injection risk.
- Báo cáo nặng chạy đồng bộ → timeout. Cần chuyển background + tải kết quả sau.

## 10. Cross-link

- [course-service.md](course-service.md)
- [user-service.md](user-service.md)
- [question-service.md](question-service.md)
- [training-route-service.md](training-route-service.md)
- [system-service.md](system-service.md) — CustomReport config.
