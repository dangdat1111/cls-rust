# NotificationService — Deep Dive

> Path: `CLS4.0-Notification/src/MobileNotification/MobileNotification.API/`.
>
> Mobile push notification qua Firebase Cloud Messaging (FCM). Persistent vào MongoDB.

## 1. Cấu trúc thư mục

```
MobileNotification.API/
├── Controllers/
│   └── NotificationController.cs
├── Application/NotificationApp/
│   ├── Commands/
│   │   ├── RegisterCommand.cs
│   │   ├── UnregisterCommand.cs
│   │   ├── SubscribeTopicCommand.cs
│   │   ├── UnsubscribeTopicCommand.cs
│   │   ├── NotificationSeenCommand.cs
│   │   └── DeleteNotificationCommand.cs
│   ├── Queries/
│   │   ├── GetListNotificationQuery.cs
│   │   └── GetNotificationsQuery.cs
│   └── IntegrationEvents/EventHandling/
│       └── NotificationIntegrationEventHandler.cs
├── Services/
│   └── IPushNotificationService.cs (FCM wrapper)
└── AutofacModules/
```

## 2. Controllers + endpoints

| Endpoint | Method | Mục đích | Auth |
|---|---|---|---|
| `/api/notification/register` | POST | Đăng ký device FCM token | Authorized |
| `/api/notification/unregister` | POST | Hủy đăng ký device | Authorized |
| `/api/notification/subscribe-topic` | POST | Subscribe topic | Authorized |
| `/api/notification/unsubscribe-topic` | POST | Unsubscribe topic | Authorized |
| `/api/notification/get-notifications` | GET | Danh sách notification của user | Authorized |
| `/api/notification/set-seen` | PUT | Đánh dấu đã đọc | Authorized |
| `/api/notification/delete` | DELETE | Xoá notification | Authorized |

## 3. Use case — Commands

### RegisterCommand
| Input | `{UserId, DeviceToken, Platform (iOS/Android/Web), DeviceId}` |
| Business rule | Mỗi user có nhiều device; upsert nếu trùng `DeviceToken`; clean token cũ của user khác (cho trường hợp share device) |
| Side-effect | Update `UserDeviceGroup` Mongo doc |

### UnregisterCommand
| Side-effect | Xoá token khỏi `UserDeviceGroup` |

### SubscribeTopicCommand / UnsubscribeTopicCommand
| Input | `{Topic, DeviceToken}` |
| Side-effect | Gọi FCM API `subscribeToTopic` / `unsubscribeFromTopic` |
| Use case | Gửi notification hàng loạt theo topic (ví dụ: topic `portal:42` cho mọi user portal 42) |

### NotificationSeenCommand
| Input | `{NotificationIds[]}` |
| Side-effect | Update `NotificationUsers.IsRead = true`, `ReadAt = now` |

### DeleteNotificationCommand
| Side-effect | Soft-delete trong `NotificationUsers` (set `IsDeleted`); không xoá notification gốc (giữ cho user khác) |

## 4. Integration event handler

### NotificationIntegrationEventHandler
```
Subscribe: NotificationIntegrationEvent
Payload: { 
  PortalId, 
  EventType, 
  Title, 
  Body, 
  ImageUrl, 
  ActionUrl, 
  Recipients[] // userIds 
}

Logic:
1. Lookup template từ API Helper (SystemService) — render Title/Body theo language user
2. Lookup device token của recipients trong UserDeviceGroup (Mongo)
3. Insert Notification doc (Mongo) — 1 doc, nhiều recipient
4. Insert NotificationUsers per recipient (cho UI list)
5. BackgroundJob.Enqueue(() => FCM.SendMultiAsync(tokens, payload))
6. Nếu user có topic subscription → cũng push qua topic
```

## 5. FCM integration

- Package: `Firebase.Notifications` hoặc HTTP REST direct.
- Credential: Service account JSON, đường dẫn trong env.
- Capability:
  - Single device send (notification + data payload)
  - Multi-device batch (max 500/request)
  - Topic broadcast
- Retry: Hangfire auto retry nếu FCM API fail.

## 6. Logic đặc thù

### Mobile vs Web
- Web push chỉ phục vụ qua SignalR (xem [signalr.md](signalr.md)).
- FCM lo mobile + có thể web nếu app dùng Firebase Web SDK.

### Topic-based bulk push
- Server publish vào topic, FCM broadcast tới mọi subscriber.
- Tiết kiệm so với gửi từng device.

### Read state
- `Notifications` doc: notification gốc.
- `NotificationUsers` doc: per-user read state — query feed nhanh.

## 7. Dependency

- MongoDB (`Notifications`, `UserDeviceGroups`, `NotificationUsers`).
- RabbitMQ.
- FCM (Firebase credential JSON).
- SystemService (lấy template).
- Hangfire (enqueue FCM send).

## 8. Gap

- `Notification` doc nhiều recipient → khó query "feed của user X" hiệu quả; phải JOIN với `NotificationUsers`. Index quan trọng: `NotificationUsers(UserId, CreatedAt desc)`.
- FCM token timeout: device token có thể invalid khi user xoá app. Cần cleanup token định kỳ (FCM trả response indicating invalid token).
- Không có statistics open/click — chỉ track `IsRead` từ user action trong app.

## 9. Cross-link

- [signalr.md](signalr.md) — web push fallback.
- [system-service.md](system-service.md) — NotificationConfig + EmailTemplate.
- [log-service.md](log-service.md) — audit notification log.
- [../04-database-schema.md](../04-database-schema.md#102-notificationservice-notifications-db)
