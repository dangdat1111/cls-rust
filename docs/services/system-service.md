# SystemService — Deep Dive

> Path: `CLS4.0-SystemService/src/System.API/`. Aggregate root: `EmailTemplate`, `EmailEvent`, `NotificationConfig`, `Widget`, `CustomReport`. **22 bảng SQL**.
>
> Quản lý cấu hình cấp portal — email template, notification toggle, widget UI, custom report.

## 1. Use case — module EmailTemplate / EmailEvent

### CreateOrUpdateEmailTemplateCommand
| Input | `{EventId, Subject, Body (HTML), LanguageCode, Variables[]}` |
| Business rule | `EventId` tồn tại trong `EmailEvent`; `Id=0` → tạo mới, `Id>0` → update; HTML body có thể validate qua `ICustomClient` external |
| Side-effect | Upsert `EmailTemplate`; `EmailTemplateField` lưu danh sách variable |
| Dependency | `EmailTemplateRepository`, `CustomClient` |

### CreateOrUpdateEmailEventCommand
| Input | `{Name, Category (EventCategoryId), Description, IsEnabled}` |
| Business rule | `Name` unique trong portal; `EventCategoryId` tồn tại |
| Side-effect | Upsert `EmailEvent`; link nhiều `EmailTemplate` |

### DeleteEmailTemplate
- Soft-delete; preserve cho audit.

## 2. Use case — module NotificationConfig

### UpdateNotificationConfigCommand
| Input | `{EventId, LanguageCode, IsMailSending, IsWebNotification, IsMobileNotification, Time?, Remind?, Interval?}` |
| Business rule | `EventId` tồn tại; nếu `Remind/Interval` set → reminder scheduling |
| Side-effect | `INotificationConfigManager.AddOrUpdateAsync` upsert; `UnitOfWork.SaveEntitiesAsync` |
| Dependency | `NotificationConfigManager`, `NotificationConfigRepository` |

### ToggleNotificationCommand (bulk)
| Side-effect | Bulk update `IsEnabled` theo event |

### BulkUpdateConfigCommand
| Side-effect | Apply config template lên nhiều event cùng lúc |

## 3. Use case — module Widget

### CreateWidgetCommand
| Input | `{TypeId (HeaderWidget/FooterWidget/Banner/...), Position, Title, Subtitle, Text, MultimediaTypeId (Video/Image/HTML), CourseCriteriaId?, MaximumCourse?, IsCourseByCriteria, BannerWidgetItems[], CourseWidgetItems[]}` |
| Business rule | Type-specific validation; Position chiếm slot trong layout grid |
| Side-effect | Insert `Widget` polymorphic + child items (polymorphic) |

### UpdateWidgetCommand
| Business rule | Position immutable qua UpdateWidget — phải dùng `UpdateWidgetPositionCommand` riêng |
| Side-effect | Update + refresh Items |

### AddItemCommand (polymorphic item)
| Business rule | Item schema validate theo `TypeId` (Banner / Course / Multimedia / Hyperlink) |
| Side-effect | Insert `WidgetItem` (table tương ứng) |

### Loại Widget supported
- `HeaderWidget` — top bar config
- `FooterWidget` — footer config
- `BannerWidgetItem` — carousel
- `CourseWidgetItem` — block khoá học featured
- `HyperlinkWidgetItem` — link nav
- `LearnerWidgetItem` — block học viên
- `TeacherWidgetItem` — block giảng viên

## 4. Use case — module CustomReport

### CreateCustomReportCommand
| Input | `{Name, TypeId (FinancialSummary/UserProgress/...), Models (JSON config: filter, columns, aggregation), IsMerge (merge nhiều source)}` |
| Side-effect | Insert `CustomReport`; persist models JSON |

### RunReportCommand
| Input | `{CustomReportId, DateRange, OutputFormat (PDF/Excel/JSON)}` |
| Logic | Load config → build SQL query theo Models → execute → format output |
| Dependency | `QueryExecutor`, `ReportFormatter` |

## 5. Use case — module ConfigEvent

### CRUD config event hệ thống
| Side-effect | Insert/update `ConfigEvent` — phân loại event mức enum |

## 6. Integration events

### Publish
- (Rất ít) `EmailTemplateUpdatedEvent` để invalidate cache phía MailSender.
- `NotificationConfigUpdatedEvent`.

### Subscribe
- Không nhiều — service này chủ yếu được service khác query (qua HTTP).

## 7. Dependency

- SQL Server (System DB, 22 bảng).
- RabbitMQ (publish event invalidate cache).
- `ICustomClient` (validate HTML email template qua service ngoài).
- MailSender (consumer của EmailTemplate).
- SignalR (consumer của NotificationConfig).

## 8. Logic đặc thù

### Per-portal override
- Mọi config (EmailTemplate, NotificationConfig, Widget, Theme) đều có `PortalId` → cho phép từng tenant tự custom.
- Default template (PortalId NULL) làm fallback.

### Template rendering
- SystemService chỉ lưu template HTML + variable schema.
- Actual render xảy ra ở MailSender (đọc template từ SystemService qua HTTP).
- Variables thay bằng data từ event (`{{user_name}}`, `{{course_name}}`).

### Custom Report engine
- `Models` JSON chứa config: source tables, columns, filters, group by, aggregation function.
- Run-time builder convert JSON → SQL/Dapper query.
- Risk: SQL injection nếu không sanitize chặt — cần dùng parameterized query.

## 9. Gap

- CustomReport builder có thể là SQL-injection vector nếu không sanitize.
- EmailTemplate variable không có whitelist → có thể chèn HTML/script vào subject.
- Widget polymorphic item table phức tạp — query JOIN nặng, nên có view hoặc materialized result.

## 10. Cross-link

- [mail-sender.md](mail-sender.md) — consumer EmailTemplate.
- [notification.md](notification.md) — consumer NotificationConfig.
- [signalr.md](signalr.md) — consumer NotificationConfig.
- [../04-database-schema.md](../04-database-schema.md#6-systemservice-db)
