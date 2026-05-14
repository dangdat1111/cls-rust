# ServerFiles — Deep Dive

> Path: `CLS4.0-ServerFiles/src/Services/ServiceFile/ServerFile.API/`. Endpoint public: `lmssf.doffice.com.vn:8021`.
>
> File upload/download, SCORM serving, video transcode, PDF encryption. Body limit 500MB.

## 1. Cấu trúc thư mục

```
ServerFile.API/
├── Controllers/
│   ├── FileController.cs (~2000+ dòng, nhiều endpoint)
│   └── ViewController.cs (SCORM + Video viewer)
├── Services/
│   ├── FileHelper.cs
│   └── ProcessFileHelper.cs (background processing)
├── Application/UserApp/
│   ├── Commands/
│   │   ├── UpLoadFileCommand.cs
│   │   ├── UploadExamJsonCommand.cs
│   │   └── FileCommand.cs
│   └── Queries/
│       ├── FileQuery.cs
│       └── GetFileByFolderNameQuery.cs
└── Helpers/
    └── ProcessFileHelper.cs
```

## 2. Endpoints — FileController

### Upload
| Endpoint | Method | Mục đích |
|---|---|---|
| `/api/file/upload` | POST (multipart) | Upload file chính (form-data, auth) |
| `/api/file/upload-cdn` | POST | Upload video từ CDN URL |
| `/api/file/uploadQuestion` | POST | Upload file đính kèm câu hỏi |
| `/api/file/exam` | POST | Upload exam JSON |
| `/api/file/exams` | POST | Upload nhiều exam JSON |
| `/api/file/captureImage` | POST | Capture & save image từ base64 (snapshot webcam) |

### Download / View
| `/api/file/downloadDocument/{folderName}` | GET | Download (secure PDF, video m3u8) |
| `/api/file/downloadMultiFile` | POST | Download multi files as ZIP |
| `/api/file/copy/{folderName}` | GET | Copy file |
| `/api/file/info/{folderName}` | GET | File metadata |
| `/api/file/multiFileInfo?fds[]=...` | GET | Info multi-file |

### Stats
| `/api/file/files` | POST | List files (pagination) |
| `/api/file/total-size` | GET | Tổng dung lượng portal |
| `/api/file/total-size-by-portals` | GET | Dung lượng theo list portal |
| `/api/file/duration-cdn?cdn=&url=&secretKey=` | GET | Lấy duration video CDN |

## 3. Folder organization

```
/uploads/<year>/<month>/<day>/<portalId>/<userId>/<random>.<ext>
```

Phân hoạch theo thời gian + tenant + user → dễ backup + cleanup theo portal.

## 4. Loại file hỗ trợ

| Type | Xử lý |
|---|---|
| Image | Resize nhiều kích thước, optimize |
| Video | Transcode HLS (m3u8), multi-resolution (240p/480p/720p/1080p), encrypt key |
| Audio | Preserve format |
| PDF | Encrypt với password `folderName+folderName` |
| SCORM | Auto unzip khi upload `.zip`, serve files trong folder qua iframe |
| Resource | Generic — preserve as-is |

## 5. Use case — Commands

### UpLoadFileCommand
| Input | Multipart form: `{File, Type, PortalId, UserId}` |
| Business rule | (a) Validate file type via `FileType` table; (b) Check size limit; (c) Check quota Portal (`Memory` của `Credit`); (d) Generate folder path |
| Side-effect | Lưu file vào disk; insert `File` metadata (DB); nếu Video/PDF/SCORM → enqueue background processing |

### Background processing
- `ProcessFileBackgroundParam`: queue async (Hangfire fire-and-forget).
- `FireAndForgetOperationVideo`: download from CDN + transcode.
- Video: FFmpeg chạy nền convert m3u8 + encrypt.
- PDF: dùng iText / PdfSharp encrypt với password.
- SCORM: unzip vào folder con + parse `imsmanifest.xml` để biết entry point.

### UploadExamJsonCommand
| Input | JSON file exam (cấu trúc question/answer) |
| Side-effect | Parse + lưu, không xử lý nặng |

## 6. SCORM serving

- ViewController có endpoint `/api/view/scorm/{folderName}/{path*}` serve file từ folder unzip.
- Frontend mở SCORM trong iframe với src trỏ tới endpoint này.
- Tracking SCORM (`cmi.completion_status`, `cmi.score.raw`) gửi về CourseService qua API riêng.

## 7. Secure download

### PDF encrypted
- Password = `folderName + folderName` (concat).
- Frontend nhận password qua API sau khi auth → mở PDF.
- Mục đích: ngăn share link PDF công khai mà vẫn mở được.

### Video m3u8
- Encrypted segment key.
- Player (video.js) load key qua endpoint có JWT.

### JWT direct download
- Một số endpoint nhận `?token=...` để cho phép browser download (vì XHR không kích thước lớn được).

## 8. Quota management

- Mỗi Portal có `Credit.Memory` (MB max).
- Trước upload, check `SELECT SUM(Size) FROM File WHERE PortalId=X` < quota.
- Nếu vượt → reject 413 Payload Too Large.

## 9. Logic đặc thù

### m3u8 rewrite cho mobile/Safari
- Safari không support tất cả codec → backend rewrite m3u8 manifest để chọn variant tương thích.
- Adaptive bitrate dựa trên User-Agent.

### File transfer giữa node
- Khi cluster nhiều ServerFiles instance, có endpoint sync file (rsync-like) nếu chạy multi-node với storage không shared.
- Khuyến nghị: dùng K8s PV shared (NFS/Ceph) thay vì sync.

## 10. Dependency

- SQL Server (metadata, 5 bảng nhỏ).
- Disk (K8s PV, mount path config qua appsettings).
- Hangfire (background job transcode).
- FFmpeg (binary trên image — đảm bảo Dockerfile install).
- iText / PdfSharp (PDF processing).
- (Không dùng RabbitMQ — chủ yếu HTTP).

## 11. Gap / improvement

- Dùng disk K8s PV → khó scale + backup. Khuyến nghị migrate sang **S3-compatible** (MinIO, AWS S3, GCS) — Rust crate `aws-sdk-s3` rất tốt.
- PDF password đơn giản (folderName×2) — dễ guess. Cần random key + lưu DB.
- Quota check trước upload có race condition khi upload song song — dùng row lock hoặc transaction.
- Video transcode trên cùng pod ServerFiles tốn CPU → tách worker riêng (FFmpeg cluster).
- `captureImage` endpoint không có rate limit → có thể spam.

## 12. Migration Rust note

- Web framework: Axum + multipart support.
- File I/O async: `tokio::fs`.
- Image: `image` crate.
- Video transcode: vẫn dùng FFmpeg subprocess (Rust không thay được).
- PDF: `lopdf` / `printpdf` (đánh giá kỹ).
- SCORM: zip unzip với `zip` crate.
- Storage abstraction: trait `Storage` → impl `LocalDisk` + `S3` để dễ chuyển.

## 13. Cross-link

- [course-service.md](course-service.md) — SCORM tracking.
- [question-service.md](question-service.md) — webcam snapshot vi phạm.
- [log-service.md](log-service.md) — image vi phạm metadata.
- [../05-deployment-operations.md](../05-deployment-operations.md) — endpoint + body limit.
