# SignalR Service — Deep Dive

> Path: `CLS4.0-SignalR/src/SignalR.API/`. Endpoint public: `lmssignal.doffice.com.vn:8015`.
>
> Vai trò: realtime hub xử lý chat, notification, exam supervising, user online tracking. Scale qua **Redis backplane**.

## 1. Hub architecture (6 hubs)

| Hub | Endpoint | Trách nhiệm |
|---|---|---|
| `ConnectionHub` | `/hubs/connection` | User online/offline, group join, inactivity check |
| `NotificationHub` | `/hubs/notification` | Push notification realtime |
| `ExamHub` (lớn nhất, ~994 dòng) | `/hubs/exam` | Exam proctoring: test room, examinee/supervisor, violation, identity verify |
| `ChatHub` | `/hubs/chat` | Tin nhắn 1-1 + group, persist Mongo |
| `CourseHub`, `ForumHub`, `AuthenticationHub` | (định nghĩa, ít implementation) | Course/forum event, auth flow |

## 2. ConnectionHub — lifecycle

### OnConnectedAsync
```
1. Read `auth_time` từ JWT claim
2. Cross-check với MongoDB (audit) → reject nếu auth_time outdated (đã đổi pass + logout cưỡng bức)
3. Join group `mgmt:{managementId}` để broadcast trong tenant
4. Redis SET user online (key `online:{userId}` → managementId)
5. Lấy pending notification + push
```

### OnDisconnectedAsync
```
1. Push signal `RequestOnlineVerification` về client (3s grace period)
2. Nếu client không reply trong 3s → coi như mất kết nối thật
3. Redis DEL `online:{userId}`
4. Broadcast `UserOffline` cho group `mgmt:{managementId}`
```

### Inactivity tracking
- Static `_listInactiveUserIds` phân biệt disconnect tạm thời vs thật sự offline.
- Khi user reload trang → disconnect + reconnect nhanh → không broadcast offline.

## 3. ExamHub — proctoring deep mechanics

### TestManager (in-memory static)
- Maintains exam session state: `Examinee` list + `Supervisor` list per test room.
- Lưu `ConnectionId`, `FirstConnectionTime`, `Status (Connected/DoingExam/Submitted/Disconnected)`.

### Methods chính

| Method | Trigger | Logic |
|---|---|---|
| `ConnectToExam(examId, examineeId)` | Client connect vào exam | Tạo `Examinee` object; status=Connected; gắn vào TestManager |
| `StartExam(examineeId)` | User bấm bắt đầu | Status=DoingExam; reset timer; signal supervisor |
| `DisConnectToExam` | Mất kết nối giữa thi | Save disconnect time vào MongoDB `ReStartExamUserDto`; wait reconnection trong timeout |
| `ReconnectExam` | User reload trang | Restore state từ Mongo; resume timer |
| `HandleViolation(type, payload)` | Tab-switch/no-face/identity-fail | Publish `CreateTestLogIntergrationEvent` (type=11 suspend, 12 minus points); signal cả examinee + supervisors |
| `IdentifyUser(examineeId)` | Supervisor yêu cầu xác minh | Send `RequestIdentity` signal về examinee |
| `SubmitIdentity(imageUrl)` | Examinee upload ảnh webcam | Send `IdentitySubmitted` về supervisor |
| `VerifyIdentity(examineeId, isVerified)` | Supervisor xác nhận | Update status; nếu fail → violation type=identity-fail |
| `EndExam` | Hết giờ hoặc submit | Cleanup TestManager; signal complete |

### Violation handling
- Type code: 11 (suspend), 12 (minus points), 13 (tab-switch), 14 (no-face), 15 (multi-face).
- Publish `CreateTestLogIntergrationEvent` → LogService consume → save Mongo `ViolateUser`.
- Snapshot webcam (nếu có) → upload qua ServerFiles → path lưu `ViolateImageUser`.

## 4. ChatHub

### Methods
- `Send(toUserId, message, testId?)` — gửi tin nhắn.
- `MarkAsRead(messageIds[])` — đánh dấu đọc.
- `JoinGroup(groupId)` — vào group chat.

### Logic
- Persist vào MongoDB `Chats` collection qua CommunicationService.
- Redis lưu `openChat:{userId}` để biết user đang mở conversation nào (cho read receipt).
- Push notification fallback nếu user offline (qua NotificationService).

## 5. NotificationHub

### Methods
- `Subscribe(topic)` — subscribe theo topic (course/exam/forum).
- `Acknowledge(notificationId)` — confirm đã nhận.

### Logic
- Subscribe RabbitMQ event → push xuống user theo `userId → connectionId` mapping qua Redis.
- Khi user offline → defer; lưu vào `NotificationUsers` để user thấy khi vào.

## 6. Integration events subscribe (~15)

| Event | Handler |
|---|---|
| `NotificationIntegrationEvent` | Push notification qua NotificationHub |
| `SendMessageIntegrationEvent` | Push chat message |
| `ReadMessageIntegrationEvent` | Update read status |
| `MessageSeenIntegrationEvent` | Push read receipt |
| `StartExamIntegrationEvent` | Notify examinee start |
| `ExamTimerStartIntegrationEvent` | Sync timer |
| `UserDisconnectFromExamIntegrationEvent` | Notify supervisor |
| `UserReconnectToExamIntegrationEvent` | Restore state |
| `SendViolateWhenUserShutdownScreenIntegrationEvent` | Violation |
| `RenewedTestCodeIntegrationEvent` | Re-issue TestCode |
| `SendMessageCompleteScormIntegrationEventHandler` | SCORM hoàn thành |
| `LogoutIntegrationEventHandler` | Xoá Redis online map của user |

## 7. Redis usage (StackExchange.Redis)

| Key pattern | Mục đích |
|---|---|
| `online:{userId}` | User online → managementId |
| `openChat:{userId}` | Conversation đang mở |
| `auth_time:{userId}` | Validate token chưa bị force-logout |
| `SC{userId}` | SCORM session mapping |
| `exam:{examId}:examinee:{id}` | Exam session state cache |

## 8. Logic đặc thù

### Multi-instance scaling
- Hệ thống có thể scale SignalR ra nhiều pod K8s.
- Redis backplane đảm bảo broadcast giữa các instance: 1 user connect tới instance A, 1 user khác tới instance B → vẫn nhắn tin được.

### Force-logout via auth_time
- Khi user đổi password / admin lock user, publish `LogoutIntegrationEvent`.
- SignalR handler: xoá Redis `online:{userId}` + disconnect tất cả connection của user.

### TestManager (in-memory state) — point of concern
- Lưu trong RAM của 1 instance → nếu instance restart, mất state thi.
- Reconnect logic dùng MongoDB làm fallback, nhưng vẫn rủi ro cho ca thi đang chạy.
- **Cải thiện**: chuyển state sang Redis hoặc DB.

## 9. Dependency

- Redis (`Redis:Host`) — backplane + state cache.
- RabbitMQ — consume integration event.
- MongoDB (qua CommunicationService) — persist chat + exam log.
- IdentityServer — validate JWT khi handshake.
- ServerFiles — upload snapshot webcam.

## 10. Gap

- `TestManager` static in-memory → restart instance = mất state.
- Không có rate-limit cho `Send` chat → user spam được.
- ConnectionHub broadcast `UserOffline` 3s grace có thể trễ trong UI khi user reload.
- SignalR protocol Microsoft riêng → khi migrate Rust, đây là điểm risk lớn (xem [../07-rust-migration-plan.md](../07-rust-migration-plan.md#pha-3--service-realtime--file-3-tháng--high-risk)).

## 11. Cross-link

- [identity-server.md](identity-server.md)
- [question-service.md](question-service.md) — ExamHub host xử lý exam.
- [communication-service.md](communication-service.md) — Mongo persist chat.
- [notification.md](notification.md) — FCM fallback.
- [log-service.md](log-service.md) — violate log.
