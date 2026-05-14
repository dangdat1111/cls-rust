# CommunicationService — Deep Dive

> Path: `CLS4.0-CommunicationService/src/Communication.API/`.
>
> Persist chat message + test log + exam log vào MongoDB. Tách hot-write khỏi service nghiệp vụ chính.

## 1. Cấu trúc thư mục

```
Communication.API/
├── Controllers/
│   ├── MessageController.cs
│   └── ExamController.cs
├── Application/MessagingApp/
│   ├── Commands/
│   │   ├── CreateMessageCommand.cs
│   │   └── ReadMessageCommand.cs
│   ├── Queries/
│   │   ├── GetUserChatQuery.cs
│   │   ├── GetAmountNewChatQuery.cs
│   │   └── GetNewChatNotificationQuery.cs
│   └── IntegrationEvents/EventHandling/
│       ├── SendMessageIntegrationEventHandler.cs
│       └── ReadMessageIntegrationEventHandler.cs
└── Domain/Models/MessagingAggregate/
    └── Chat.cs
```

## 2. Collections Mongo

| Collection | Schema | Mục đích |
|---|---|---|
| `Chats` | `_id, FromUserId, ToUserId, Type (1-1/group), GroupId, TestId?, Message, Attachments[], CreatedAt, IsRead, ReadAt` | Tin nhắn |
| `TestLogs` | `_id, TestId, UserId, Action (start/answer/submit/violate), QuestionId?, AnswerId?, Timestamp, Metadata{}` | Action user khi làm test |
| `StudentExamLogs` | `_id, ExamId, ShiftId, TestCodeId, UserId, Events[{type, timestamp, data}], StartedAt, SubmittedAt, Score` | Full exam session log |

## 3. Endpoints — MessageController

| Endpoint | Method | Mục đích |
|---|---|---|
| `/api/message/get-amount-new-chat?toUserId=X&fromUserId=Y&testId=Z` | GET | Đếm tin nhắn chưa đọc |
| `/api/message/get-chat?...&pageNumber=&pageSize=&isSeen=` | GET | Chat history phân trang, filter `isSeen` |
| `/api/message/get-chat-notification?toUserId=X&testId=Y` | GET | List chat notification |
| `/api/message/get-new-chat-notification?toUserId=X&testId=Y` | GET | Count chat mới |
| `/api/message/read-message` | POST | Mark messages read |
| `/api/message/send-message` | POST | Gửi tin nhắn (qua API thay vì SignalR) |

## 4. Use case — Commands

### CreateMessageCommand
| Input | `{FromUserId, ToUserId, Message, TestId?, Attachments[]}` |
| Business rule | (a) `FromUserId == current user`; (b) `ToUserId` tồn tại; (c) Nếu trong context exam (TestId set) → validate user thuộc test |
| Side-effect | Insert `Chat` doc; publish `SendMessageIntegrationEvent` → SignalR push realtime |

### ReadMessageCommand
| Input | `{MessageIds[]}` |
| Side-effect | Update `IsRead=true`, `ReadAt=now`; publish `MessageSeenIntegrationEvent` → SignalR notify sender |

## 5. Use case — Queries

### GetUserChatQuery
| Input | `{FromUserId, ToUserId, TestId?, PageNumber, PageSize, IsSeen?}` |
| Logic | Filter `Chats` theo cặp user (cả 2 chiều) + optional `TestId`; sort `CreatedAt desc`; paginate |
| Output | `{Items[], Total}` |

### GetAmountNewChatQuery
| Logic | Count `Chats WHERE ToUserId=X AND FromUserId=Y AND IsRead=false` |

### GetNewChatNotificationQuery
| Logic | Aggregate group by `FromUserId` count unread → cho UI hiển thị "n cuộc chat chưa đọc" |

## 6. Integration event handlers

### SendMessageIntegrationEventHandler
```
Subscribe: SendMessageIntegrationEvent
1. Create Chat doc (Mongo)
2. Republish event để SignalR push xuống ToUserId
3. Nếu ToUserId offline → trigger NotificationService FCM push (gián tiếp qua chain event)
```

### ReadMessageIntegrationEventHandler
```
Subscribe: ReadMessageIntegrationEvent
1. Update IsRead=true cho list messages
2. Publish MessageSeenIntegrationEvent để SignalR notify sender
```

## 7. ExamController + test/exam log

### Receive event ghi exam log
- `UserStartedExamIntegrationEvent` → insert `StudentExamLogs` với `StartedAt`.
- `SaveAnswerEvent` → push event vào `Events[]` array.
- `ExamCompletedIntegrationEvent` → update `SubmittedAt`, `Score`.
- `ViolateDetectedEvent` → push violation event vào `Events[]`.

### Reconstruct exam session
- Query `StudentExamLogs` theo `(ExamId, UserId)` → replay events thấy được hành trình làm bài.
- Dùng cho audit, dispute resolution.

## 8. Logic đặc thù

### Chat trong exam (TestId)
- Chat giữa giám thị ↔ examinee trong lúc thi.
- Filter theo `TestId` để cô lập conversation theo ca thi.
- Tự xoá hoặc archive sau khi ca thi kết thúc + N ngày.

### Phân trang Mongo
- Dùng skip/limit (đơn giản nhưng chậm với offset lớn).
- **Improvement**: dùng cursor-based pagination với `CreatedAt`.

## 9. Dependency

- MongoDB.
- RabbitMQ.
- SignalR (consumer của event re-publish).
- NotificationService (gián tiếp qua event chain).
- UserService (validate FromUserId/ToUserId).

## 10. Gap

- Không có encryption at rest cho chat content — privacy concern.
- Skip/limit pagination chậm khi messages > 100k.
- Không có "delete message" command (chỉ archive).
- `TestLogs` và `StudentExamLogs` chưa có TTL → DB phình.

## 11. Cross-link

- [signalr.md](signalr.md) — realtime push.
- [question-service.md](question-service.md) — provider exam event.
- [log-service.md](log-service.md) — overlap với violate log.
- [../04-database-schema.md](../04-database-schema.md#101-communicationservice-chats-db)
