# LogService — Deep Dive

> Path: `CLS4.0-LogService/src/Services/SystemLogging/SystemLogging.API/`.
>
> Audit log toàn hệ thống + giám sát vi phạm thi. Persistent vào MongoDB.

## 1. Cấu trúc thư mục

```
SystemLogging.API/
├── Controllers/
│   ├── UserController.cs
│   ├── DashboardController.cs
│   ├── NotificationController.cs
│   ├── TimelineController.cs
│   └── ViolateUserController.cs
├── Application/
│   ├── UserApp/Queries/
│   │   └── GetTotalLoginQuery.cs
│   ├── DashboardApp/Queries/
│   │   ├── GetJoiningExamQueryHandler.cs
│   │   ├── UserAnalyticQueryHandler.cs
│   │   └── GetActivityUserQueryHandler.cs
│   ├── Logging/IntegrationEvents/
│   │   └── (event handlers ghi audit)
│   └── NotificationApp/IntegrationEvents/
└── ...
```

## 2. MongoDB collections

| Collection | Schema | Mục đích |
|---|---|---|
| `SystemActivities` | `_id, PortalId, UserId, Module, Action, EntityType, EntityId, Description, Timestamp, IpAddress, UserAgent, Metadata{}` | Timeline action toàn hệ |
| `Notifications` | (mirror cho audit) | Lưu lại notification đã gửi |
| `ViolateUsers` | `_id, ExamId, TestId, UserId, ShiftId, ViolationType, Severity, Timestamp, Details` | Vi phạm thi |
| `ViolateImageUsers` | `_id, ViolateUserId, ImagePath (ServerFiles path), Timestamp, FaceDetected, Score` | Snapshot webcam |

## 3. Endpoints

### UserController
| `GET /api/user/get-total-login?userId=X` | Tổng số lượt login user X (authorized) |

### DashboardController
- Analytics queries: `GetJoiningExam`, `UserAnalytic`, `GetActivityUser`.
- Trả số liệu time-series cho dashboard admin.

### TimelineController
- Trả timeline action của 1 user theo thời gian.
- Filter: date range, module, action type.

### ViolateUserController
- List vi phạm theo exam/shift/user.
- View evidence: gọi ServerFiles tải ảnh snapshot.

## 4. Use case — Queries

### GetTotalLoginQuery
| Input | `{UserId}` |
| Logic | Đếm `SystemActivities WHERE UserId=X AND Action='Login'` |
| Output | `{TotalLogin: int}` |

### UserAnalyticQuery (dashboard)
| Output | `{daily login count, active users 7d/30d, top portal by activity}` |
| Logic | Aggregate pipeline trên `SystemActivities` |

### GetActivityUserQuery (timeline)
| Input | `{UserId, FromDate, ToDate, PageSize, PageIndex}` |
| Output | List activity sorted by timestamp desc |

### GetJoiningExamQuery (real-time exam join stats)
| Logic | Đếm event `Action='StartExam'` trong khoảng thời gian |

## 5. Integration event handlers (subscribe mọi service)

LogService subscribe gần như **toàn bộ integration event** để ghi audit:

| Event | Handler logic |
|---|---|
| `UserCreated/Updated/Deleted` | Insert `SystemActivities {Module:User, Action:Create/Update/Delete}` |
| `LoginSuccess` | Insert `Action:Login` |
| `CourseCreated/Updated/Deleted/Enrolled/Completed` | Insert `Module:Course` |
| `ExamCreated/Started/Completed` | Insert `Module:Exam` |
| `ViolateDetected` | Insert `ViolateUsers` Mongo doc; nếu có image → save `ViolateImageUsers` |
| `CertificationIssued` | Insert `Action:CertificateIssued` |
| `NotificationSent` | Insert `Notifications` (mirror) |

## 6. Logic đặc thù

### Async write
- LogService write Mongo (nhanh) — không block service nghiệp vụ.
- Nếu LogService down → event nằm RabbitMQ retry queue → không mất.

### TTL retention
- `SystemActivities` nên có TTL index ~90 ngày (tránh DB phình).
- `ViolateUsers` giữ vĩnh viễn (evidence pháp lý).
- Hiện tại **chưa có TTL** → cần bổ sung.

### Aggregation dashboard
- Mongo aggregation pipeline cho time-series.
- Nên materialized view (hoặc cron pre-compute) cho dashboard nặng.

## 7. Dependency

- MongoDB.
- RabbitMQ (subscribe events).
- ServerFiles (lưu snapshot ảnh vi phạm).

## 8. Gap

- Không có TTL retention → DB phình theo thời gian.
- Không có index `(UserId, Timestamp desc)` đầy đủ → query timeline chậm khi data lớn.
- Không có archive/export định kỳ.
- Một số event không được log (gap audit) — cần inventory để bổ sung.

## 9. Cross-link

- [signalr.md](signalr.md) — source của exam violate event.
- [server-files.md](server-files.md) — lưu snapshot.
- [../04-database-schema.md](../04-database-schema.md#103-logservice-logging-db)
