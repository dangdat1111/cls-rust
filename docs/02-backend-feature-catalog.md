# 02 — Backend Feature Catalog

> Catalog đầy đủ tính năng của từng backend service. Dùng để PM/PO duyệt scope nghiệp vụ và dev khi cần rebuild/migrate. Mỗi service mô tả: vai trò, aggregate chính, các module Command/Query (Application/), danh sách controller + endpoint chính, integration event, dependency.
>
> **Để xem chi tiết Command/Validator/Business rule per use case, đọc file deep-dive trong [`services/`](services/README.md)**.
>
> Convention chung mọi service:
> - Mọi endpoint sau Gateway có dạng `/<service>/api/<controller>/<action>`.
> - JWT bắt buộc trừ login/health.
> - Response chuẩn `JsonResponse<T>` (success/error/data/messages).
> - Authorization filter: `[AuthorizeFilter]`, `[AuthorizeFilter("Module", level)]`.

---

## 1. IdentityServer (`CL4.0-IdentityServer`)

### Vai trò nghiệp vụ
Authorization server theo OAuth2/OIDC. Cấp token, validate credential, tích hợp social login, persist grant.

### Aggregate
- IdentityServer4 config entities: `Client`, `ApiResource`, `ApiScope`, `IdentityResource`.
- `PersistedGrant`, `DeviceCode`.

### Endpoints chính (mặc định IdentityServer4)
| Endpoint | Mục đích |
|---|---|
| `/connect/token` | Cấp access_token + refresh_token (password, client_credentials, refresh_token grant). |
| `/connect/authorize` | Authorization code flow (social login). |
| `/connect/userinfo` | Trả về claims user. |
| `/connect/endsession` | Logout, xoá session. |
| `/.well-known/openid-configuration` | Discovery document. |
| `/.well-known/jwks` | Public key cho JWT validation. |

### Module Application
- `ResourceOwnerPasswordValidator` — validate username/password (gọi UserService DB), check captcha (recaptcha), trả về user identity nếu hợp lệ.
- `ProfileService` — bổ sung claims vào JWT (portal id, user type, feature list).
- `LogoutHandler` — xoá `PersistedGrant`.
- `SocialLoginHandler` — exchange Microsoft/Google token → tạo/match user cục bộ.

### Dependency
- SQL Server (IdentityServer schema + đọc User DB cho validate).
- (Không publish RabbitMQ event trong scope hiện tại.)

---

## 2. UserService (`CLS4.0-UserService`)

### Vai trò
Cốt lõi quản lý user, tổ chức, role/feature, portal/management, profile học vấn, HR, address. Là nguồn dữ liệu user mà IdentityServer query khi authenticate.

### Aggregate chính
`ApplicationUser`, `Group`, `OrganizationalStructure`, `Portal`, `Management`, `Feature`, `FeatureGroup`, `Domain` (alias domain portal), `JwtLog`, `Customer`, `RegistryConfiguration`, `Branch`, `Title`, `UserTitle`, `UserType`, `AcademicDegree`, `AcademicRank`, `Experience`, `Proficiency`, `ProficiencyLevel`, `PayrollScale`, `School`, `Level`, `Credit`.

### Module Application (37 module `*App`)
AcademicDegreeApp, AcademicRankApp, ActivityUserApp, AddressApp, AuthenticationApp, CalendarApp, CourseApp, CreditApp, CustomerApp, DashboardApp, DegreeApp, DomainApp, ExperienceApp, ExperienceUserApp, FeatureApp, FollowUserApp, HistoryApp, HRMApp, LearnerApp, LevelApp, ManagementApp, MenuApp, OrganizationalStructureApp, PayrollScaleApp, PortalApp, ProficiencyApp, ProfileApp, RegistryConfigurationApp, ReportApp, SchoolApp, SupperAdminApp, TitleApp, UserApp, UserGroupApp, UserReportApp, UserTitleApp, UserTypeApp, UserTypeSupperAdminApp.

Mỗi module có cấu trúc `Commands/<X>Command.cs` + `Commands/Handlers/<X>Handler.cs` và `Queries/<X>Query.cs`.

### Controllers tiêu biểu
| Controller | Endpoints chính |
|---|---|
| UserController | `/api/user/change-password`, CRUD user |
| ProfileController | `change-password`, `get-detail`, `get-detail-update`, `get-list-branch` |
| DashboardController | `get-top-5-org-structure`, `scoreboard-user` |
| HRMController | `update-structure`, `update-title`, `update-user`, `course/get-list-teacher`, `user/create` |
| LearnerController | `check-course-proficiency-require` |
| AcademicDegreeController | combobox, CRUD, search, import/export Excel |
| AcademicRankController | combobox, CRUD, export Excel |
| AddressController | Country/Province/District/Ward CRUD + combobox |
| FeatureController, FeatureGroupController | CRUD + gán feature cho portal |
| FollowUserController | follow/unfollow user |
| ManagementController, PortalController | CRUD Portal + Management (multi-tenant) |
| OrganizationalStructureController | CRUD cây tổ chức + add/remove user |
| TitleController, UserTitleController, AppellationController | Chức danh, danh xưng |
| UserGroupController | Tạo/xoá nhóm + add/remove user |
| UserTypeController, UserTypeSuperAdminController | Loại user |
| ProficiencyController, ProficiencyLevelController, GroupProficiencyController | Kỹ năng + cấp độ |
| ExperienceController, ExperienceUserController, DegreeController, SchoolController | HR: kinh nghiệm, bằng cấp |
| PayrollScaleController | Bảng lương |
| LevelController | Cấp bậc |
| CreditController | Credit/điểm thưởng |
| CategoryTitleController | Danh mục chức danh |
| CourseController (proxy) | Liên thông CourseService (lấy course của user) |
| CustomerController | Approve/Create/Delete/Deny customer (multi-tenant onboarding) |
| DomainController | Cấu hình domain cho portal |
| EventController | Calendar event cá nhân |
| HistoryController | Lịch sử thay đổi |
| MenuController | Menu hiển thị theo permission |
| RegistryConfigurationController | Cấu hình đăng ký |
| ReportController, UserReportController | Báo cáo user |
| SupperAdminController | Super-admin operations |
| HRM/* (AuthenticationController, HrmGroupController, HrmOrgController, HrmUserController) | Sub-API cho HRM client |

### Integration events
- **Publish**: `UserCreatedIntegrationEvent`, `UserUpdatedIntegrationEvent`, `UserDeletedIntegrationEvent`, `UserAssignedToGroupIntegrationEvent`, `PortalCreatedIntegrationEvent`, `RoleChangedIntegrationEvent`.
- **Subscribe**: events từ Course/Question để update progress/scoreboard.

### Dependency
- SQL Server (UserService DB, 64 bảng theo migration snapshot).
- IdentityServer (cho authenticate flow).
- RabbitMQ (`micro_event_bus`).
- Captcha key (`Captcha:SecretKey`).
- SecretKey (`SecretKey:HashPassword`, `SecretKey:EmailConfirm`).

---

## 3. CourseService (`CLS4.0-CourseService`)

### Vai trò
Lifecycle khoá học: tạo, gán giảng viên, đăng ký học viên, theo dõi tiến độ, đánh giá, training plan, chấm điểm essay.

### Aggregate chính
`Course`, `CourseContent`, `CourseContentDiscuss`, `CourseAuthor`, `CourseUserRegister`, `CourseGroupUsers`, `CourseBranchs`, `TrainingPlan`, `PlanCourse`, `Cost`, `CourseRate`, `CourseLike`, `CourseEvaluation`, `ContentArchive`, `ContentArchiveType`, `ContentOffline`, `ContentRollCall`, `TopicCourse`, `TrainingCategory`, `TrainingType`.

### Module Application (32 module)
ContentArchiveApp, ContentArchiveTypeApp, ContentOfflineApp, ContentRollCallApp, CostApp, CourseApp, CourseContentApp, CourseContentDiscussApp, CourseDetailApp, CourseEvaluationApp, CourseLikeApp, CourseRateApp, CourseReportApp, CourseRequestApp, CourseShareApp, CourseUserApp, DashboardApp, GroupApp, HomeApp, HRMApp, LearnerApp, LearningProgressApp, MarkPointCourseApp, OrganizationalStructureApp, ParticipantApp, PlanApp, PlanCourseApp, ProfileApp, ReportApp, TopicCourseApp, TrainingCategoryApp, TrainingPlanApp, TrainingTypeApp, UserApp.

### Controllers tiêu biểu
| Controller | Mục đích |
|---|---|
| CourseController, CourseDetailController | CRUD khoá học, detail |
| CourseContentController | Quản lý bài học/content trong khoá |
| ContentArchiveController | Kho content tái sử dụng — Create, CreateExamTestArchive, Delete, GetAll, GetAuthor, GetByCourseId, GetById, GetPagingArchive, GetFilesInCourse, Update, UpdateExamTestArchive, ExportReport |
| ContentArchiveTypeController | Combobox loại archive |
| CostController | Chi phí khoá học (CRUD + export, total, exam cost) |
| CourseLikeController, CourseRateController, CourseEvaluationController | Tương tác đánh giá khoá học |
| CourseReportController, ReportController | Báo cáo khoá học |
| CourseRequestController | Yêu cầu đăng ký (Create/Approve/Deny/Delete) |
| CourseShareController | Khoá học chia sẻ giữa portal |
| CourseUserController, ParticipantController | Quản lý học viên trong khoá (add/remove, paging) |
| DashboardController | Dashboard nghiệp vụ khoá học |
| HomeController | Trang chủ public/portal |
| HRMController | Liên kết HR |
| LearnerController | Truy cập khoá học của học viên |
| MarkPointCourseController | Chấm điểm khoá học |
| PlanController, PlanCourseController, TrainingPlanController | Training plan + map plan-course |
| ProfileController | Hồ sơ học viên trong context khoá học |
| TeacherController | Quản lý giảng viên cấp khoá học |
| TrainingCategoryController, TrainingTypeController | Categorize khoá học |

### Integration events
- **Publish**: `CourseCreatedIntegrationEvent`, `CourseUpdatedIntegrationEvent`, `CourseDeletedIntegrationEvent`, `UserEnrolledIntegrationEvent`, `UserCompletedCourseIntegrationEvent`, `CourseAssignedIntegrationEvent`, `CourseRecalledIntegrationEvent`, `CertificationConditionMetIntegrationEvent`.
- **Subscribe**: `UserUpdatedIntegrationEvent` (sync user data), `ExamCompletedIntegrationEvent` (cập nhật progress khoá học có exam).

### Dependency
- SQL Server (Course DB, 44 bảng).
- RabbitMQ.
- ServerFiles (lưu file SCORM, video).
- UserService (qua HTTP/event).
- QuestionService (cho course exam).
- SharedServices (cho cấp chứng chỉ khi hoàn thành).

---

## 4. QuestionService (`CLS4.0-QuestionService`)

### Vai trò
Ngân hàng câu hỏi, đề thi/ca thi, chấm điểm, khảo sát, giám sát thi (anti-cheat).

### Aggregate chính
`Question`, `Answer`, `QuestionLevel`, `Topic`, `Exam`, `Test`, `TestCode`, `TestCodeQuestion`, `TestCodeArchive`, `Survey`, `SurveyExam`, `SurveyTest`, `SurveyTestCode`, `SurveyTestCodeArchive`, `CourseContentTest`, `CourseContentSurvey`, `MarkPoint`, `MarkPointTest`, `Cost`, `ExamSupervising`.

### Module Application (28 module)
CostApp, CourseApp, CourseContentSurveyApp, CourseContentTestApp, CourseServiceApp, DashboardApp, ExamApp, ExamLearnerApp, ExamLecturer, ExamReportApp, ExamSupervisingApp, ExamTestApp, HRMApp, LearnerApp, MarkPointApp, MarkPointTestApp, ProfileApp, QuestionApp, QuestionLevelApp, QuestionReport, ReportApp, SurveyApp, SurveyManagerApp, SurveyReportApp, TestCodeApp, TestCodeArchiveApp, TestCodeQuestionApp, TestCodeSurveyApp, TopicApp.

### Controllers tiêu biểu
| Controller | Mục đích |
|---|---|
| QuestionController, QuestionLevelController | Ngân hàng câu hỏi, độ khó |
| QuestionReportController | Báo cáo phân tích câu hỏi |
| ExamController | CRUD đề thi/kỳ thi |
| ExamTestController | Map Exam ↔ Test |
| ExamReportController | Báo cáo kỳ thi |
| ExamSupervisingController | Giám sát thi (anti-cheat: webcam, tab-switch) |
| SurveyController, SurveyReportController, SurveyTestController, SurveyExamController, SurveyTestCodeController, SurveyTestCodeArchiveController | Khảo sát |
| TestCodeController, TestCodeQuestionController, TestCodeSurveyController, TestCodeArchiveController | Compose đề từ ngân hàng |
| CourseContentTestController | Test gắn vào CourseContent — GetDetail, GetPagingList, Create, Update, Delete, GetPagingLearner, GetTotalPoint, ExportReport |
| CourseContentSurveyController | Survey gắn vào CourseContent — GetDetail, GetPagingList, Create, Update, Delete |
| MyExamController | Học viên làm bài thi (submit, view) |
| MySurveyController | Học viên làm khảo sát |
| MarkPointController, MarkPointCourseController | Chấm điểm (auto + manual essay) |
| CostController | Chi phí thi |
| DashboardController, ReportController, ProfileController, HRMController, LearnerController, CourseController | Các phụ trợ |

### Logic đặc thù
- **Auto-grading**: với question loại single-choice/multi-choice/fill-in, hệ thống chấm tự động khi học viên submit. Essay gửi sang giảng viên chấm tay.
- **Anti-cheat**: trong khi làm bài, frontend gửi tin hiệu (tab-switch, webcam-snapshot) qua SignalR → QuestionService consume → tạo `ViolateUser` log (qua LogService).
- **Random TestCode**: mỗi học viên trong cùng ca thi nhận TestCode khác nhau (rút từ pool TestCode của Exam) để chống quay cóp.
- **Import/Export Excel**: hỗ trợ bulk import câu hỏi bằng template Excel (EPPlus).

### Integration events
- **Publish**: `ExamCreatedIntegrationEvent`, `ExamCompletedIntegrationEvent`, `QuestionImportedIntegrationEvent`, `UserStartedExamIntegrationEvent`, `ViolateDetectedIntegrationEvent`.
- **Subscribe**: `UserAssignedToExamIntegrationEvent` từ Course/Route.

### Dependency
- SQL Server (Question DB, 59 bảng).
- RabbitMQ.
- ServerFiles (lưu file đính kèm câu hỏi).
- LogService (ghi violate).

---

## 5. SharedServices (`CLS-SharedServices`)

### Vai trò
Tài nguyên dùng chung cross-service: bài viết, thư viện tài liệu, chứng chỉ, gift, theme, topic, rating scale.

### Aggregate chính
`Article`, `ArticleComment`, `ArticleCommentReaction`, `ArticleReaction`, `ArticleBookmark`, `Library`, `LibraryComment`, `LibraryCommentReaction`, `LibraryUserRating`, `LibraryBookmark`, `LibraryImage`, `Certification`, `CertificationTemplate`, `CertificationUser`, `ExternalCertification`, `Appellation`, `GroupAppellationOrganizationalStructure`, `Gift`, `GiftUser`, `Theme`, `Topic`, `RatingScale`, `RatingScaleLevel`, `CostType`, `Reel`, `SettingSystem`.

### Module Application (30 module)
AppellationApp, ArticleApp, ArticleBookmarkApp, ArticleCommentApp, ArticleCommentReactionApp, ArticleReactionApp, CertificationApp, CertificationTemplateApp, CertificationUserApp, CostTypeApp, ExternalCertificationApp, GiftApp, GiftUserApp, GroupAppellationOrganizationalStructureApp, LibraryApp, LibraryBookmarkApp, LibraryCommentApp, LibraryCommentReactionApp, LibraryImageApp, LibraryUserRatingApp, RatingScaleApp, RatingScaleLevelApp, ReelApp, SettingSystemApp, ThemeApp, ToolApp, TopicApp.

### Controllers tiêu biểu
| Controller | Mục đích |
|---|---|
| ArticleController | CRUD bài viết, approve, get-new, get-total |
| ArticleBookmarkController | Bookmark bài viết |
| ArticleCommentController + ArticleCommentReactionController + ArticleReactionController | Bình luận + react |
| LibraryController | Quản lý thư viện (AddContent, GetList, Inspect, Recall, GetAuthor, GetLearningSteps) |
| LibraryBookmarkController, LibraryCommentController, LibraryCommentReactionController, LibraryImageController, LibraryUserRatingController | Tương tác thư viện |
| CertificationController | CRUD certification + assign user (CountCertificateSuccess, CreateCourseCertification, CreateTrainingRouteCertification, etc.) |
| CertificationTemplateController | Mẫu chứng chỉ |
| CertificationUserController | Cấp chứng chỉ cho user, import/export Excel, ranking |
| ExternalCertificationController | Chứng chỉ ngoài hệ thống do user khai báo |
| AppellationController, GroupAppellationOrganizationalStructureController | Danh xưng |
| GiftController, GiftUserController | Gift/reward |
| CostTypeController | Loại chi phí dùng chung |
| RatingScaleController, RatingScaleLevelController | Thang đánh giá |
| ReelController | Video reel (TikTok-style content) |
| SettingSystemController | Setting hệ thống cấp portal |
| SharedController | Báo cáo report tổng hợp user (`get-user-report`, `get-user-report-course-report`, `get-user-report-by-learn-time`) |
| ThemeController | CRUD theme, set-active, combobox |
| ToolController | Tool DBA (`ToolAsync` — chạy raw SQL, nên giới hạn role) |
| TopicController | Cây topic dùng cho course/question/route — search, paging, combobox theo nhiều scope, import/export |

### Integration events
- **Publish**: `CertificationIssuedIntegrationEvent`, `ArticlePublishedIntegrationEvent`.
- **Subscribe**: `CertificationConditionMetIntegrationEvent` (từ Course/Route) → tự cấp certificate.

### Dependency
- SQL Server (Shared DB, 40 bảng).
- RabbitMQ.
- ServerFiles (file đính kèm article/library, render PDF certificate).

---

## 6. SystemService (`CLS4.0-SystemService`)

### Vai trò
Quản lý email template, cấu hình notification theo event, widget UI, custom report, event category.

### Aggregate chính
`EmailTemplate`, `EmailEvent`, `NotificationEvent`, `NotificationConfig`, `Widget`, `ConfigEvent`, `CustomReport`.

### Module Application tiêu biểu
- `EmailTemplateApp` — CRUD template (subject, body HTML, variables).
- `EmailEventApp` — định nghĩa các event sẽ trigger email (user-registered, course-completed...).
- `NotificationConfigApp` — bật/tắt notification cho từng event theo portal.
- `WidgetApp` — quản lý widget header/footer/course block trên portal.
- `CustomReportApp` — builder cho custom report (filter, columns).
- `ConfigEventApp` — phân loại event hệ thống.

### Endpoints chính
- `/api/emailtemplate/*` — CRUD template, preview render.
- `/api/notificationconfig/*` — get config theo portal + event, update toggle.
- `/api/widget/*` — list widget theo portal.
- `/api/customreport/*` — define report, run query, export.

### Integration events
- **Publish**: thường không publish (đây là cấu hình).
- **Subscribe**: không nhiều — chủ yếu được service khác query.

### Dependency
- SQL Server (System DB, 22 bảng).
- Service khác query qua HTTP / cached.

---

## 7. TrainingRouteService (`CLS4.0-TrainingRouteService`)

### Vai trò
Quản lý lộ trình học tuần tự (TrainingRoute) — chuỗi LearningStep (Course/Exam) mà học viên phải hoàn thành theo thứ tự.

### Aggregate chính
`TrainingRoute`, `LearningStep`, `LearningStepExam`, `TrainingRouteUser`, `TrainingRouteGroup`, `TrainingRouteOrganizationalStructure`.

### Module Application (4 module)
- `TrainingRouteApp` — CRUD route, approve/deny/pending, add/remove group/user/org-struct, recall request, set complete; reports đa dạng.
- `TrainingRouteUserApp` — operations cho user trên route.
- `LearnerApp` — query route của learner.
- `LearningStepApp` — query step.

### Controllers tiêu biểu
| Controller | Endpoints chính |
|---|---|
| TrainingRouteController | (rất nhiều — xem chi tiết) |
| LearnerController | `GetListAsync` |
| LearningStepController | `GetLearningStepsByIdAsync` |

**TrainingRouteController** có ~60 endpoint, tóm tắt theo nhóm:
- Lifecycle: Create, Edit, Delete, Pending, Approve, Deny, Recall, RecallRequest, SetComplete.
- Assign: AddGroup, RemoveGroup, AddUser, RemoveUser, UpdateOrganizationalStructure.
- Query detail: GetDetail, GetShortDetail, GetDetailUser, GetCalendar, GetTimeline, GetListByIds.
- Query paging: GetPaging, GetPagingPending, GetPagingGroup, GetPagingUser, GetPagingUserOther, GetPagingReport.
- Combobox: GetComboboxCreator, GetComboboxCreatorPending, GetComboboxTrainingRoute.
- Report: TrainingRouteCostReport, TrainingRoutesReport, TrainingRoutesNumberReport, TrainingRouteCourseExamCostReport, TrainingRouteStepReport, TraningRouteStepUserReport, UserTrainingRouteReport, UserTrainingRouteStepReport, UserTrainingRouteStepCompleteReport, UserTrainingRouteByIdReport, UserReportByLearnTime, GetReportOrStructureDashboardExam.
- Export Excel: ExportTrainingRouteCostReport, ExportTrainingRoutesReport, ExportExcelUserTrainingRouteReport, ExcelTemplateAddUsers.
- Import: ImportListUserFromEmail.
- Khác: GetListCourseCertificationAsync, ListUserIdFromTrainingRouteAsync, GetListStatusSurveyTestOfUserAsync, GetTop5JoinPercent, GetDashboardTrainingUser, GetTrainingRouteInforNotification.

### Integration events
- **Publish**: `TrainingRouteCreatedIntegrationEvent`, `TrainingRouteUserAssignedIntegrationEvent`, `TrainingRouteCompletedIntegrationEvent`, `LearningStepCompletedIntegrationEvent`.
- **Subscribe**: `CourseCompletedIntegrationEvent`, `ExamCompletedIntegrationEvent` để cập nhật progress step.

### Dependency
- SQL Server (TrainingRoute DB, 11 bảng).
- Course/Question/SharedServices (cross-service query).
- RabbitMQ.

---

## 8. SignalR (`CLS4.0-SignalR`)

### Vai trò
Hub realtime gửi thông báo/chat/exam event xuống browser. Tách khỏi service nghiệp vụ để chịu load WebSocket riêng.

### Hub chính
| Hub | Mục đích |
|---|---|
| `ConnectionHub` | Track user online (connect/disconnect), gửi join/leave event. |
| `NotificationHub` | Push notification đẩy realtime. |
| `ExamHub` | Sự kiện trong ca thi: bắt đầu, hết giờ, snapshot webcam, tab-switch alert. |
| `ChatHub` (nếu enable) | Chat 1-1, group. |

### Logic
- Người dùng login → frontend connect SignalR với JWT.
- Hub map `connectionId` ↔ `userId` lưu trong Redis (để scale ra nhiều node).
- Integration event từ RabbitMQ → Handler trong SignalR service → tìm `connectionId` của user đích → push qua hub.
- Redis pub/sub là backplane cho multi-instance.

### Endpoints
- `/hubs/connection`, `/hubs/notification`, `/hubs/exam`, `/hubs/chat`.
- WebSocket + Long Polling + SSE (do SignalR protocol tự xử lý).

### Integration events subscribe
- Tất cả event push xuống user: notification, course-assigned, exam-event, chat-message, violate-detected, certification-issued.

### Dependency
- Redis (`Redis:Host`) — backplane + user-online map.
- RabbitMQ — consume integration event.
- IdentityServer (validate JWT khi handshake).

---

## 9. CommunicationService (`CLS4.0-CommunicationService`)

### Vai trò
Lưu trữ realtime data (chat, exam log) vào MongoDB; tách hot-write ra khỏi SQL.

### Collections Mongo
- `Chats` — tin nhắn 1-1 + group.
- `TestLogs` — log hành động học viên trong khi làm test (xem câu, chọn đáp án, change).
- `StudentExamLogs` — log toàn diện kỳ thi (mở bài, submit, vi phạm).

### Module Application
- `MessageApp` (Chat) — gửi/edit/xoá message, paging, search.
- `TestLogApp` — write-only (Hub gửi event), query report.
- `StudentExamLogApp` — write từ ExamHub, query reconstruct exam session.

### Endpoints
- `/api/chat/*` — get conversation, send message, mark read.
- `/api/testlog/*`, `/api/studentexamlog/*` — query report.

### Dependency
- MongoDB.
- RabbitMQ (subscribe event từ Question/SignalR).

---

## 10. NotificationService (`CLS4.0-Notification`)

### Vai trò
Mobile push notification qua FCM (Firebase Cloud Messaging). Tách khỏi SignalR vì FCM async + có queue.

### Aggregate
- `Notification` (Mongo doc).
- `UserDeviceGroup` — map user ↔ FCM token (mobile device).
- `NotificationUsers` — recipients per notification.

### Module Application
- `NotificationApp` — tạo, push, mark read, paging.
- `UserDeviceGroupApp` — register device token.
- `NotificationUserApp` — query per-user feed.

### Endpoints
- `/api/notification/*` — CRUD, register device, get feed, mark-read.

### Logic
- Service khác publish `NotifyUserIntegrationEvent` → Notification consume → lưu Mongo + gọi FCM HTTP API push tới device.

### Dependency
- MongoDB.
- RabbitMQ.
- Firebase credentials.

---

## 11. LogService (`CLS4.0-LogService`)

### Vai trò
Audit log toàn hệ thống + giám sát vi phạm thi.

### Collections Mongo
- `SystemActivities` — timeline action (login, edit, delete).
- `Notifications` — copy notification gửi user (audit).
- `ViolateUsers` — user vi phạm khi thi.
- `ViolateImageUsers` — snapshot webcam khi vi phạm (binary trên ServerFiles, path lưu Mongo).

### Module Application
- `LoggingApp` — write activity, query timeline.
- `DashboardApp` — dashboard hoạt động.
- `UserApp` — query user activity.
- `ExamMongoApp` — exam-specific log query.
- `ViolateUserApp` — list violate, view evidence.

### Endpoints
- `/api/log/*`, `/api/violateuser/*`, `/api/dashboard/*`.

### Dependency
- MongoDB.
- RabbitMQ (subscribe mọi event nghiệp vụ để write activity).
- ServerFiles (lưu snapshot ảnh).

---

## 12. ServerFiles (`CLS4.0-ServerFiles`)

### Vai trò
File upload/download, SCORM serving, batch file processing, transfer file giữa node.

### Aggregate
`File`, `FileType`, `Config`, `Exam` (local), `User` (local).

### Endpoints
- `/api/file/upload`, `/api/file/download/{id}`, `/api/file/delete/{id}`.
- `/api/scorm/{coursecontentid}/{file}` — serve unzipped SCORM content.
- `/api/file/batch` — bulk upload.
- `/api/filetype/*` — CRUD loại file cho phép.

### Logic
- Upload: lưu file vào disk (mount K8s PV), record metadata vào SQL.
- SCORM: unzip tự động khi upload `.zip`, expose qua iframe.
- File size limit 500MB (xem Ingress config).
- Hỗ trợ presigned URL hoặc redirect cho file lớn.

### Dependency
- SQL Server (small DB cho metadata).
- Disk (K8s PV).

---

## 13. MailSender (`CLS4.0-MailSender`)

### Vai trò
Background worker tiêu thụ EmailEvent từ RabbitMQ, render template (lấy từ SystemService), gửi SMTP.

### Logic
1. Service khác publish `SendEmailIntegrationEvent` với `{event_type, recipient, data}`.
2. MailSender consume → query SystemService lấy `EmailTemplate` theo `event_type` + portal.
3. Render template (variables thay bằng `data`).
4. Gửi SMTP qua config `MailSettings:*`.
5. Hangfire job retry nếu fail.
6. (Tuỳ chọn) publish `EmailSentIntegrationEvent`.

### Dependency
- RabbitMQ.
- SystemService (HTTP query template).
- SMTP server (`MailSettings:Server` `:Port` `:User` `:Password` `:From`).
- Hangfire (queue + retry + dashboard).

---

## 14. ReportService (`CLS4.0-ReportService`)

### Vai trò
Báo cáo tổng hợp cross-service. Đa số query trực tiếp các DB (qua Dapper), một số là proxy gọi service tương ứng.

### Loại báo cáo
- Course report: completion rate, drop-out, top khoá.
- User report: progress, learn time, certificate count.
- Exam report: phân phối điểm, question hot-fail.
- Cost report: tổng chi phí theo Course/Plan/Route.
- Custom report: từ SystemService.

### Endpoints
- `/api/report/course/*`, `/api/report/user/*`, `/api/report/exam/*`, `/api/report/cost/*`, `/api/report/custom/*`.

### Dependency
- Multi-DB read (đọc DB của Course/User/Question).
- SystemService cho custom report.

---

## 15. APIGateway (`CLS4.0-APIGateway`)

### Vai trò
Ocelot gateway: route + JWT middleware + Swagger aggregate + rate limit.

### Cấu hình
- `ocelot.json` chứa route definition:
  - `UpstreamPathTemplate`: `/<service>/api/...`.
  - `DownstreamPathTemplate`: `/api/...`.
  - `DownstreamHostAndPorts`: từng service (lấy từ Consul/Eureka).
  - `AuthenticationOptions`: JWT.
- Aggregate Swagger từ mỗi service: gateway gọi `/swagger/v1/swagger.json` của từng service.

### Middleware
- JWT validate (issuer = IdentityServer authority).
- CORS.
- Rate limit per client + per IP.
- Logging request/response.

### Endpoints
- `/.well-known/openid-configuration` (proxy IdentityServer).
- `/swagger` (aggregate UI).
- Mọi endpoint khác theo Ocelot route.

---

## 16. CLS4.0-Core (shared library)

### Vai trò
NuGet/internal library cung cấp foundation cho mọi service. Nằm tại `CLS4.0-Core/src/Core/`.

### Modules chính
| Module | Class chính | Mục đích |
|---|---|---|
| `Core.EventBus` | `IEventBus`, `IIntegrationEventHandler<T>` | Abstraction cho event bus |
| `Core.EventBus.RabbitMQ` | `EventBusRabbitMQ`, `DefaultRabbitMQPersistentConnection`, `IRabbitMQPersistentConnection` | Implementation RabbitMQ với Polly retry |
| `Core.Domain.SeedWork` | `Entity`, `AggregateRoot`, `Enumeration`, `IRepository<T>`, `IUnitOfWork`, `Repository<T>` | Base DDD |
| `Core.Domain.SeedWork` | `ISoftDelete` | Soft delete interface |
| `Core.Infrastructure` | `IDbContextCore`, `BaseDbContext` | DbContext abstraction |
| `Core.Infrastructure.Repositories` | `Repository<T>` với EFCore.BulkExtensions | Bulk insert/update/delete |
| `Core.API.Controllers` | `CustomController`, `ClsRequestHandler` | Base controller + handler |
| `Core.API.Middleware` | `ExceptionHandlerMiddleware`, `JwtMiddleware`, `CorrelationIdMiddleware` | Cross-cutting middleware |
| `Core.API.Filters` | `AuthorizeFilter` | Auth filter theo feature |
| `Core.API.Behaviors` | `LoggingBehavior`, `ValidationBehavior`, `TransactionBehavior` | MediatR pipeline |
| `Core.Authentication` | `JwtHandler`, `JwtConfigurationExtensions` | JWT setup |
| `Core.Discovery` | `EurekaConfigurationExtensions` | Service discovery setup |
| `Core.Configuration` | `ConfigurationBuilderExtensions.AddConfigurationFromHost()` | Consul loader |
| `Core.Logging` | `SerilogConfigurationExtensions` | Serilog + Elastic sink |
| `Core.Healthcheck` | health check extension | `/hc` endpoint |
| `Core.Models.Response` | `JsonResponse<T>` | Response wrapper chuẩn |
| `Core.Common` | Extension methods, Helper | String, Date, Crypto helpers |

### Sử dụng
Mọi service `.csproj` reference `CLS4.0-Core` (qua project reference hoặc NuGet package nội bộ). Trong `Startup.cs` thường gọi:

```csharp
services.AddCoreAuthentication(Configuration);
services.AddCoreEventBus(Configuration);
services.AddCoreLogging(Configuration);
services.AddCoreHealthcheck();
services.AddMediatR(...);
services.AddCoreMediatRPipeline();
```

---

## Quy ước cross-service

### Authentication flow
1. Client gọi IdentityServer `/connect/token` với username/password.
2. IdentityServer validate qua UserService DB → cấp JWT.
3. JWT chứa claims: `sub` (userId), `portalId`, `userTypeId`, `roles`, `features`.
4. Mọi request sau gửi `Authorization: Bearer <jwt>`.
5. Gateway validate JWT; service nhận JWT đã validated, đọc claims qua `HttpContext.User`.

### Soft delete
Mọi aggregate kế thừa `ISoftDelete` đều có `IsDeleted` (bool). Repository tự filter `WHERE IsDeleted = false`.

### Audit field
Mọi aggregate có: `CreatedDate`, `CreatedBy`, `UpdatedDate`, `UpdatedBy`, `IsActive`, `IsDeleted`.

### Pagination
Convention input: `{ pageIndex, pageSize, search, orderBy, orderDirection, filter: {...} }`.
Convention output: `{ items: T[], total: int, pageIndex, pageSize }`.

### Integration event base
```csharp
public abstract class IntegrationEvent {
    public Guid Id { get; }
    public DateTime CreationDate { get; }
    public Guid PortalId { get; }   // tenant
    public Guid? UserId { get; }    // actor
    public string CorrelationId { get; }
}
```

### Multi-tenant
Mọi data đều có `PortalId`. Query/Command tự thêm filter `WHERE PortalId = @currentPortalId` từ JWT claim.

---

## Tham chiếu

- Schema chi tiết per-service: [04-database-schema.md](04-database-schema.md).
- Tổng quan kiến trúc: [01-system-overview.md](01-system-overview.md).
- Thuật ngữ: [06-glossary.md](06-glossary.md).
- Migration sang Rust: [07-rust-migration-plan.md](07-rust-migration-plan.md).
