# 03 — Frontend Feature Catalog

> Catalog đầy đủ chức năng của 2 ứng dụng Vue 2: `CLS4.0-WebUI` (cổng học viên/giảng viên/admin portal) và `CLS4.0-WebUI-Admin` (cổng super-admin). Dùng để PM/PO duyệt scope; dev khi cần sửa hoặc rebuild frontend.

## 1. Kiến trúc chung 2 ứng dụng

| Khía cạnh | Cấu hình |
|---|---|
| Framework | Vue 2.6.12 + Vue Router 3 + Vuex 3 |
| Build tool | Vue CLI 4 (webpack 4) |
| Package manager | Yarn 1.x |
| UI library | Bootstrap-Vue 2, vue-toastification, vue-sweetalert2, vee-validate |
| Charts/Media | apexcharts, echarts, @fullcalendar/*, quill (rich text), video.js |
| HTTP | Axios (có interceptor JWT) |
| Realtime | @microsoft/signalr |
| Push notification | Firebase messaging (chỉ WebUI) |
| Social login | @azure/msal-browser (Microsoft) — WebUI |
| State | Vuex modules theo từng feature |
| i18n | vue-i18n (đa ngôn ngữ VN/EN) |

### Cấu trúc thư mục chung

```
src/
├── @core/           <- core component, mixin, layout của theme
├── auth/            <- guard, JWT helper, login helpers
├── libs/            <- third-party config (signalr, firebase, axios)
├── router/
│   ├── index.js     <- router root, guard check JWT + permission
│   └── routes/      <- 1 file per module (course.js, test.js, ...)
├── store/           <- Vuex root + module
├── views/           <- screen component theo module
├── components/      <- shared component
├── layouts/         <- vertical / horizontal layout
└── assets/
```

### Cấu hình env (cả 2 app)

| Var (runtime) | Var (build arg Docker) | Ý nghĩa |
|---|---|---|
| `VUE_APP_BASE_API` | `CI_VUE_APP_BASE_API` | URL APIGateway (Ocelot) |
| `VUE_APP_BASE_SERVER_FILE` | `CI_VUE_APP_BASE_SERVER_FILE` | URL ServerFiles |
| `VUE_APP_BASE_URL` | `CI_VUE_APP_BASE_URL` | URL public của portal |
| `VUE_APP_BASE_SERVER_SIGNAL` | `CI_VUE_APP_BASE_SERVER_SIGNAL` | URL SignalR Hub |
| `VUE_APP_BASE_API_CLS` (Admin) | `CI_VUE_APP_API_CLS` | URL admin gateway |
| - | `CI_APP_VERSION` | Hiển thị version trong footer |

---

## 2. CLS4.0-WebUI — cổng người dùng

> Đây là cổng đa-role: học viên, giảng viên, admin portal đều dùng chung; menu/feature hiện theo permission từ JWT.

### 2.1 Authentication & Profile

Folder: `views/pages/authentication/`, `views/pages/account-setting/`.

| Màn hình | Route | Mô tả |
|---|---|---|
| Login | `/login` | Đăng nhập username/password. Có nút social: Microsoft (MSAL), Google. Có nút "Quên mật khẩu". |
| Portal chọn | `/select-portal` | Khi user thuộc nhiều portal, chọn portal trước khi vào hệ thống. |
| Forgot password | `/forgot-password` | Gửi email reset (gọi event qua MailSender). |
| Reset password | `/reset-password?token=...` | Đặt mật khẩu mới. |
| Account settings | `/account-setting` | Thông tin cá nhân, avatar, đổi mật khẩu, security log (JwtLog). |

### 2.2 Dashboard

Folder: `views/dashboard/`.

| Màn hình | Mô tả |
|---|---|
| Student dashboard | Tổng quan khoá học đang học, deadline, thông báo, lộ trình tiến độ. |
| Lecturer dashboard | Lớp đang dạy, bài essay cần chấm, kỳ thi sắp tới. |
| Admin portal dashboard | Số user, số khoá học, hoạt động gần đây, top khoá học. |

### 2.3 Module Học tập (Student)

Folder: `views/students/`.

| Submodule | Folder | Mô tả |
|---|---|---|
| My courses | `list-courses/`, `list-learning-course/`, `detail-my-course/`, `myCourseModuleStore.js` | Danh sách khoá đã đăng ký, vào học, theo dõi progress |
| Course detail (study) | `course/` | Player nội dung: video, SCORM (iframe + tracking), PDF viewer, essay submission |
| Learning path | `learning-path/` | Hiển thị Training Route + LearningStep, mở/khoá step theo tiến độ |
| Library | `library/` | Thư viện tài liệu chia sẻ (không phải khoá học) |
| Take exam | `exam/` | Làm bài thi: countdown, navigator, anti-cheat (webcam snapshot, tab-switch log) |
| Org struct view | `org-struct/` | Xem sơ đồ tổ chức của portal |
| Notification condition | `noti-course-condition/` | Cấu hình thông báo cá nhân |

### 2.4 Module Khoá học (Quản trị nội dung)

Folder: `views/courses/`.

| Submodule | Folder | Mô tả |
|---|---|---|
| Course management | `course/` | CRUD khoá học, gán giảng viên, gán đối tượng học, recall request |
| Course share | `course-share/` | Khoá học chia sẻ giữa các portal |
| Content repository | `content-repository/` | Kho nội dung tái sử dụng cho khoá học |
| Topic | `topic/` | Chủ đề/category khoá học |
| Library admin | `library/` | Quản lý thư viện |
| Exam (course-level) | `exam/` | Bài kiểm tra gắn vào khoá học |
| Survey | `survey/` | Khảo sát cuối khoá |
| Cost | `cost/` | Chi phí khoá học |

### 2.5 Module Bài thi & Ngân hàng câu hỏi

Folder: `views/tests/`.

| Submodule | Folder | Mô tả |
|---|---|---|
| Question bank | `question/` | CRUD câu hỏi, phân loại theo độ khó/topic, import/export Excel |
| Survey question | `survey/` | Câu hỏi dạng survey (không có đáp án đúng) |

### 2.6 Module Lộ trình đào tạo

Folder: `views/training/`.

| Submodule | Folder | Mô tả |
|---|---|---|
| Training roadmap list | `list-training-roadmap/` | Danh sách lộ trình, status, người tạo |
| Edit roadmap | `edit-training-roadmap/` | Thiết kế lộ trình: kéo thả LearningStep, gán user/group |

### 2.7 Module Quản lý người dùng

Folder: `views/users/`.

| Submodule | Folder | Mô tả |
|---|---|---|
| User CRUD | `user/` | Tạo/sửa/xoá user trong portal |
| User group | `user-group/` | Nhóm user theo lớp/đợt đào tạo |
| Organizational structure | `organizational-structure/` | Cây tổ chức (phòng ban) |
| Education management | `education-management/` | Bằng cấp, học vị của user |
| Proficiency | `proficiency-management/` | Kỹ năng/proficiency |
| Branch | `branch/` | Chi nhánh |
| User type | `user-type/` | Phân loại user |
| User title | `user-title/` | Chức danh (Appellation) |
| Activity status | `activity-status/` | Trạng thái user (active/inactive/violate) |

### 2.8 Module Báo cáo

Folder: `views/report/` (dùng chung `reportModuleStore.js`, ~76KB — đây là module nặng nhất frontend).

Các loại báo cáo:

- **Overview** (`overview/`) — dashboard tổng hợp.
- **Course report** (`course/`) — tỷ lệ hoàn thành, điểm trung bình, drop-out.
- **User report** (`user/`) — tiến độ học từng user.
- **User group report** (`user-group/`) — theo nhóm.
- **Exam report** (`exam/`) — phân phối điểm, top câu hỏi sai.
- **Survey report** (`survey/`) — kết quả khảo sát.
- **Lecturer report** (`lecturer/`) — hiệu suất giảng viên.
- **Cost report** (`cost/`) — chi phí đào tạo.
- **Training plan report** (`training-plans/`) — tiến độ kế hoạch.
- **Training (route) report** (`training/`) — tiến độ lộ trình.
- **Branch / Position / Qualification / Org-struct** — báo cáo theo chi nhánh, vị trí, bằng cấp, cây tổ chức.

### 2.9 Module Cấu hình hệ thống (Portal-level)

Folder: `views/system-management/`.

| Submodule | Mô tả |
|---|---|
| Homepage config | Cấu hình trang chủ portal (banner, widget) |
| Notifications config | NotificationConfig theo event |
| Setting | Setting chung portal (theme, logo, working hour, etc.) |
| System history | Log thay đổi cấu hình portal |

### 2.10 Module Giảng viên

Folder: `views/teacher/`.

| Submodule | Mô tả |
|---|---|
| Mark course (chấm essay) | Chấm tay essay học viên nộp |
| Mark exam | Chấm tay essay trong bài thi |
| Attendance list | Điểm danh học viên (offline/online class) |
| Question-answer | Q&A trong khoá học, tương tác giảng viên-học viên |

### 2.11 Module Apps (lịch + cộng đồng + chứng chỉ)

Folder: `views/apps/`.

| Submodule | Mô tả |
|---|---|
| Calendar | Lịch khoá học, lịch thi, lịch dạy |
| User-calendar | Lịch cá nhân |
| Meeting | Tích hợp video meeting (Webmeeting/mediasoup) |
| Forum | Diễn đàn thảo luận trong khoá học |
| Notifications | Trung tâm thông báo (đọc/xoá/đánh dấu) |
| Badge | Huy hiệu gamification |
| Gift | Đổi quà bằng credit |
| Certification | Tường chứng chỉ cá nhân, tải PDF certificate |
| File | Trình quản lý file (upload tài liệu cá nhân) |
| Training-plans | Quản lý/duyệt training plan |
| Coming soon | Placeholder cho feature WIP |

### 2.12 State store (Vuex)

Folder `store/`:

- `app/`, `app-config/` — config app, theme, layout.
- `vertical-menu/` — menu items render theo permission.
- `connections/` — SignalR connection state, online user list.
- `notifications/` — notification feed + counter.
- `server-file/` — upload progress state.
- `theme-config/` — theme dark/light, layout.
- `share/` — cross-module shared state.

---

## 3. CLS4.0-WebUI-Admin — cổng super-admin

> Cổng riêng cho super-admin (cấp Management). Ít feature hơn WebUI nhưng có chức năng cấu hình Portal/Feature toàn hệ thống.

Cấu trúc: `views/apps/system/` chứa các màn hình admin chính, `views/pages/` chứa auth/account settings.

### 3.1 Authentication

Tương tự WebUI: `views/pages/authentication/` (login, forgot, reset).

### 3.2 Module Quản trị hệ thống

Folder `views/apps/system/`.

| Màn hình | File chính | Mô tả |
|---|---|---|
| **Admin management** | `Admin.vue` (25KB) | CRUD tài khoản admin, gán role/feature, lock/unlock |
| **User type** | `UserType.vue`, `UserTypeEdit.vue`, `user-type/` | Định nghĩa loại user (Học viên/Giảng viên/Quản trị/HR/...) |
| **Feature group** | `FeatureGroup.vue` | Nhóm feature thành group để quản lý dễ hơn |
| **Feature** | `Feature.vue` (13.7KB) | Bật/tắt từng feature cho từng Portal |
| **Portal** | `portal/` | Tạo/sửa/xoá Portal, set domain, theme, parent management |
| **Credit** | `credit/` | Cấu hình hệ thống credit (cấp/trừ điểm, đổi quà) |
| **Turnover** | `Turnover.vue` (16KB) | Quy trình bàn giao admin nghỉ việc (chuyển user/role/asset) |
| **Dashboard** | `dashboard/` | Dashboard cấp Management (tổng số portal, user, traffic) |
| **Users admin** | `users-admin/` | Quản lý user cấp Management |
| **Modal** | `modal/` | Modal reusable trong các màn hình admin |

### 3.3 State store (Vuex)

Folder `store/` (rút gọn hơn WebUI):

- `app/`, `app-config/` — config app.
- `vertical-menu/` — menu admin.
- `server-file/` — upload state.

### 3.4 Module Account setting

Folder `views/pages/account-setting/` — đổi mật khẩu admin, profile.

---

## 4. Realtime — SignalR client

Cả 2 app dùng `@microsoft/signalr`. Cấu hình tại `libs/signalr.js` (hoặc tương tự).

Connection:
- URL: `VUE_APP_BASE_SERVER_SIGNAL`
- Auth: gửi JWT qua query/header
- Auto-reconnect bật

Hub events lắng nghe:
- `Notification` — thông báo mới → cập nhật Vuex `notifications` + toast.
- `UserOnline` / `UserOffline` — cập nhật trạng thái online.
- `ChatMessage` — tin nhắn mới (nếu module chat bật).
- `ExamEvent` — thông báo sự kiện thi (vào ca, kết thúc).

## 5. HTTP layer — Axios interceptor

Cả 2 app dùng axios với interceptor:

- Request: tự thêm `Authorization: Bearer {access_token}` từ localStorage.
- Response 401: gọi `/connect/token` với `refresh_token` để renew, retry request. Nếu refresh fail → redirect `/login`.
- Response error: toast lỗi (chỉ với lỗi không-401).

## 6. Build & deploy

### 6.1 Local

```bash
cd CLS4.0-WebUI
yarn install
yarn serve   # dev server (Vue CLI HMR)
yarn build   # production build (dist/)
yarn lint
```

Lưu ý Node 16/18 có thể cần `NODE_OPTIONS=--openssl-legacy-provider` do webpack 4 + Node mới.

### 6.2 Docker

Build với build-args (xem section env phía trên). Kết quả: image Nginx serve static `dist/`.

```bash
docker build \
  --build-arg CI_VUE_APP_BASE_API="https://lmsapi.doffice.com.vn/api" \
  --build-arg CI_VUE_APP_BASE_SERVER_FILE="https://lmssf.doffice.com.vn" \
  --build-arg CI_VUE_APP_BASE_URL="https://portal.doffice.com.vn" \
  --build-arg CI_VUE_APP_BASE_SERVER_SIGNAL="https://lmssignal.doffice.com.vn" \
  --build-arg CI_APP_VERSION="v1.4.20" \
  -t webui:prod CLS4.0-WebUI
```

## 7. Gap & technical debt frontend

- **Vue 2.6 đã end-of-life** (Dec 2023). Nên nâng cấp Vue 3 + Vite trước hoặc song song với migration backend.
- `reportModuleStore.js` 76KB — quá lớn, nên tách module.
- Một số folder typo: `commingsoon` (đúng là `comingsoon`) — đã merge nhưng không sửa.
- Thiếu test (chưa thấy `tests/` ở root webui).
- Bundle size lớn do nhập nhiều thư viện chart/media. Có thể lazy-load.

Khi migrate backend sang Rust, **frontend giữ nguyên** và không thay đổi contract (xem [07-rust-migration-plan.md](07-rust-migration-plan.md) section B). Việc nâng Vue 3 là một improvement track riêng.
