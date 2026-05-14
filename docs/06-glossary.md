# Bảng thuật ngữ (Glossary)

> Tài liệu này giải thích các thuật ngữ nghiệp vụ LMS và kỹ thuật được dùng trong toàn bộ bộ tài liệu CLS 4.0. Khi gặp thuật ngữ lạ trong các file `01..07`, tra cứu ở đây trước.

## 1. Thuật ngữ nghiệp vụ LMS

| Thuật ngữ | Giải nghĩa |
|---|---|
| **LMS** | Learning Management System — hệ thống quản lý đào tạo doanh nghiệp. |
| **Portal** | Một "cổng" độc lập trong hệ thống multi-tenant: mỗi doanh nghiệp/đơn vị có thể có 1 portal riêng với cấu hình, domain, theme, danh sách feature khác nhau. |
| **Management** | Đơn vị quản trị cấp trên của Portal — một Management có thể chứa nhiều Portal. |
| **OrganizationalStructure** | Sơ đồ tổ chức (phòng ban, đơn vị) — User được gắn vào một hoặc nhiều node của cây tổ chức. |
| **Group** | Nhóm người dùng theo chức năng/lớp học/đợt đào tạo; khác Organizational Structure (theo phòng ban). |
| **Feature** | Một chức năng có thể bật/tắt theo Portal (ví dụ: bật "Forum" cho portal A, tắt cho portal B). |
| **Course** | Khoá học — đơn vị nội dung cơ bản, có nhiều CourseContent. |
| **CourseContent** | Một bài/phần trong khoá học: SCORM, video, PDF, essay, quiz, offline. |
| **CourseAuthor** | Giảng viên/người tạo khoá học. Một Course có thể có nhiều Author. |
| **CourseUserRegister** | Bản ghi đăng ký khoá học của 1 user (có thể là tự đăng ký, được gán, hoặc recall). |
| **TrainingPlan** | Kế hoạch đào tạo tổng thể, chứa nhiều khoá học (PlanCourse) trong khoảng thời gian xác định. |
| **TrainingRoute** | Lộ trình học — chuỗi LearningStep tuần tự mà học viên phải hoàn thành theo thứ tự. |
| **LearningStep** | Một bước trong TrainingRoute, có thể là 1 Course, 1 Exam, hoặc 1 cột mốc kiểm tra. |
| **LearningStepExam** | Loại LearningStep dạng bài thi/đánh giá. |
| **Exam / Test** | Bài thi/đề thi. Một Exam có thể có nhiều TestCode (mã đề). |
| **TestCode** | Một mã đề cụ thể (compose từ Question bank), gán cho từng học viên trong ca thi. |
| **Shift / Exam Session** | Ca thi — khung thời gian thi, danh sách user được assign, TestCode được phân ngẫu nhiên. |
| **Question** | Câu hỏi trong ngân hàng câu hỏi. Có loại: single-choice, multi-choice, essay, fill-in. |
| **Answer** | Đáp án cho Question (Question có nhiều Answer, đáp án đúng được đánh dấu). |
| **Survey** | Khảo sát — dùng cùng Question bank nhưng không có đáp án đúng, dùng để thu thập ý kiến. |
| **Certification** | Chứng chỉ học viên đạt được khi hoàn thành Course/Route. |
| **CertificationTemplate** | Mẫu chứng chỉ (HTML/PDF template) dùng để render certificate. |
| **Article** | Bài viết tin tức/blog đăng trên portal. |
| **Library** | Thư viện tài liệu chia sẻ chung (không phải khoá học chính thức). |
| **Gift / Reward** | Phần quà/điểm thưởng cấp cho học viên (gamification). |
| **Widget** | Block UI cấu hình được — header, footer, course block — hiển thị trên portal. |
| **Theme** | Giao diện màu sắc/branding của portal. |
| **Turnover** | Quá trình chuyển giao quyền/dữ liệu giữa admin (ví dụ admin nghỉ việc, bàn giao cho người kế nhiệm). |
| **ViolateUser** | User bị ghi nhận vi phạm trong khi thi (chuyển tab, không có mặt webcam, v.v.). |
| **ViolateImageUser** | Snapshot webcam khi phát hiện vi phạm — dùng làm bằng chứng. |
| **JwtLog** | Log token JWT cấp ra — dùng audit/đăng xuất cưỡng bức. |
| **Roll-call** | Điểm danh học viên trong buổi học offline. |
| **Rating Scale** | Thang đánh giá (ví dụ 1–5 sao, hoặc thang chữ A/B/C). |
| **Appellation** | Danh xưng/chức danh (Mr./Ms./Thầy/Cô/PGS./TS./...). |
| **Custom Report** | Báo cáo do admin tự cấu hình filter/column. |
| **EmailEvent** | Sự kiện sẽ trigger email (ví dụ: user-registered, course-completed). |
| **NotificationConfig** | Cấu hình điều kiện push notification theo từng event và từng portal. |

## 2. Thuật ngữ kỹ thuật

| Thuật ngữ | Giải nghĩa |
|---|---|
| **DDD** | Domain-Driven Design — phân tầng theo nghiệp vụ; Aggregate, Repository, Domain Event là khái niệm cốt lõi. |
| **Aggregate / Aggregate Root** | Cụm entity được nhóm lại, chỉ truy cập qua một entity gốc (root) để đảm bảo invariant. |
| **CQRS** | Command Query Responsibility Segregation — tách model ghi (Command) và model đọc (Query). |
| **Clean Architecture** | Phân tầng API → Application → Domain → Infrastructure; dependency hướng vào trong. |
| **MediatR** | Library .NET implement Mediator pattern, dùng dispatch Command/Query đến Handler tương ứng. |
| **Autofac** | DI container cho .NET, dùng trong toàn bộ service CLS để wire dependency. |
| **EF Core** | Entity Framework Core — ORM .NET, dùng cho ghi/đọc transactional. |
| **Dapper** | Micro-ORM .NET, dùng cho query đọc nhanh (Query handler). |
| **IdentityServer4** | OAuth2/OIDC framework cho .NET, vận hành như authorization server. |
| **JWT** | JSON Web Token — token xác thực cấp bởi IdentityServer, gửi trong header Bearer. |
| **Ocelot** | API Gateway cho .NET, định tuyến từ frontend tới các microservice phía sau. |
| **Eureka** | Service registry của Netflix OSS, dùng để service discover lẫn nhau. |
| **Consul** | KV store + service registry — CLS dùng làm config server (load appsettings từ Consul KV). |
| **EventBus** | Lớp trừu tượng publish/subscribe — implementation hiện tại trên RabbitMQ. |
| **RabbitMQ** | Message broker AMQP, dùng cho integration event giữa các service. |
| **Integration Event** | Event được publish ra EventBus cho service khác consume (ví dụ `UserRegisteredIntegrationEvent`). |
| **Domain Event** | Event nội bộ trong cùng aggregate, không ra ngoài bounded context. |
| **SignalR** | Library realtime của Microsoft — websocket + SSE + long-polling, có protocol riêng. |
| **Redis Backplane** | Cách scale SignalR ra nhiều node: tất cả node publish/subscribe message qua Redis. |
| **Hangfire** | Background job scheduler cho .NET, dùng cho MailSender và scheduled tasks. |
| **Serilog** | Structured logging cho .NET, có sink ghi ra Elasticsearch ở production. |
| **Polly** | Library retry/circuit-breaker cho .NET, dùng trong EventBusRabbitMQ. |
| **FluentValidation** | Library validation cho .NET, viết rule kiểu fluent. |
| **Steeltoe** | Bộ thư viện .NET Spring-equivalent, CLS dùng `Steeltoe.Discovery.Eureka` để register vào Eureka. |
| **Strangler Pattern** | Pattern migrate dần dần: dựng service mới song song service cũ, chuyển dần endpoint qua, rồi retire cũ. |
| **Dual-write** | Trong giai đoạn cutover: ghi cả 2 hệ thống cùng lúc để có thể rollback nếu hệ mới lỗi. |
| **Parity Test** | Test so sánh output của 2 implementation (cũ vs mới) cho cùng input — đảm bảo behavior y hệt. |
| **Bounded Context** | Ranh giới nghiệp vụ trong DDD — mỗi microservice CLS là một bounded context. |
| **Soft Delete** | Xoá logic (set `IsDeleted = true`) thay vì DELETE thực sự — CLS implement qua interface `ISoftDelete`. |
| **SCORM** | Sharable Content Object Reference Model — chuẩn đóng gói khoá học e-learning (zip có manifest XML). |
| **Tiberius** | Driver Rust cho Microsoft SQL Server (TDS protocol). |
| **SQLx** | Library Rust query DB với compile-time check. |
| **Axum** | Web framework Rust trên nền Tokio. |
| **Tokio** | Async runtime của Rust. |
| **lapin** | RabbitMQ client cho Rust. |
| **Keycloak** | Identity provider OSS Java — có thể thay thế IdentityServer4. |
| **Pingora** | Reverse proxy framework Rust của Cloudflare. |
