# CourseService — Deep Dive

> Path: `CLS4.0-CourseService/src/Services/Course/Course.API/`. Aggregate root: `Course`, `Plan`, `TrainingPlan`. **44 bảng SQL**.

## 1. Use case — module CourseApp

### AddCourseInformationCommand (CreateCourse)
| Mục | Nội dung |
|---|---|
| Actor | Giảng viên / Admin |
| Input | `{Name, Code, About, Thumbnail, FormOfStudy, AuthorIds[], BranchIds[], ...}` |
| Validator | Name/Code/Thumbnail ≤ 500/250 chars; `FormOfStudy` enum hợp lệ |
| Business rule | (a) `Name` + `Code` unique trong `PortalId`; (b) Author phải tồn tại trong UserService; (c) Chính chủ là 1 trong các Author |
| Side-effect | Insert `Course` + `CourseBranch` + `CourseAuthor`; log `CourseActionType.Create`; **không** publish ngay (chờ approve) |
| Output | `JsonResponse<int>` = CourseId |

### UpdateCourseInformationCommand
| Business rule | Course tồn tại; vẫn check name/code unique excluding self; cho phép update author list (replace) |
| Side-effect | Update; log activity |

### DeleteCourseCommand
| Input | `{Ids[]}` |
| Business rule | Course phải ở status cho phép xoá (Draft/Declined); soft-delete `IsDeleted=true` |
| Side-effect | Soft-delete; cascade publish 3 event: `DeleteProficiencyLevelCourseIntegrationEvent` (UserService xoá proficiency liên kết), `DeleteCourseIntegrationEvent` (TrainingRoute/Certification cleanup), `DeleteExamTestEvent` (Question xoá exam liên kết) |

### RejectCourseCommand
| Actor | Approver |
| Business rule | (a) User là approver của course; (b) Status chuyển `Editing → Declined`; (c) `Description` = lý do reject |

### RecallRequestCourseCommand
| Business rule | Reset Course từ `Pending` → `Editing`; chỉ owner có quyền recall |

## 2. Use case — module CourseContent

### CreateCourseContentCommand
| Input | `{CourseId, Type (SCORM/Video/PDF/Essay/Quiz), Order, FilePath, Settings{}}` |
| Business rule | Course phải tồn tại + thuộc Portal; `Order` tự tăng nếu rỗng; type Quiz cần liên kết QuestionService |
| Side-effect | Insert `CourseContent`; nếu SCORM, gọi ServerFiles unzip; publish `CourseContentCreatedIntegrationEvent` |

### UpdateCourseContentCommand
| Business rule | Content tồn tại; giữ `Order` hoặc cho reorder; nếu thay file, ghi đè |

### DeleteCourseContentCommand
| Side-effect | Soft-delete `CourseContent`; cascade xoá `CourseContentUser`, `CourseContentTest` (xuyên QuestionService qua event) |

## 3. Use case — module CourseUser (enrollment)

### AddUserCourseUserCommand
| Input | `{CourseId, UserIds[]}` |
| Business rule | Course active; user không trùng `CourseUser`; respect `CourseRequire` (chứng chỉ điều kiện, proficiency điều kiện) |
| Side-effect | Insert `CourseUser` + `CourseUserRegister`; publish `AddUserToCourseNotificationIntegrationEvent` (SignalR push), `SendEmailCourseIntegrationEvent` (MailSender) |

### DeleteUserToCourseUserCommand
| Side-effect | Remove `CourseUser`; publish event để cập nhật route/plan progress |

### AgreeAnomyousCourseUserCommand (self-enroll)
| Actor | Học viên |
| Business rule | Course mở public; check `CourseRequire`; trừ credit nếu có cost |
| Side-effect | Insert `CourseUser`; trừ credit; publish enrollment event |

## 4. Use case — module CourseRequest (xin học course share)

### CreateCourseRequestCommand
| Input | `{CourseId, Description}` |
| Side-effect | Copy course metadata sang `CourseRequest` (chờ duyệt); log activity |

### ApproveCourseRequestCommand / DenyCourseRequestCommand
| Business rule | Chỉ admin gốc của course duyệt |
| Side-effect | Update status; nếu approve → tạo `CourseShare` |

## 5. Use case — module ContentArchive (kho content tái sử dụng)

### SaveArchiveToCourseContentCommand
| Side-effect | Insert `ContentArchive`; có thể là test (`CreateExamTestArchive`) hoặc resource thường |

### Events
`ContentArchiveCreatedIntegrationEvent`, `ContentArchiveUpdatedIntegrationEvent`.

## 6. Use case — module CourseRate / Like / Evaluation

### CreateCourseRateCommand
| Input | `{CourseId, Score (1-5), Comment}` |
| Business rule | 1 user 1 rate/course (upsert); chỉ user đã `CourseUser` mới rate được |
| Side-effect | Insert/update `CourseRate` |

### CourseLike (toggle), CourseEvaluation (sau khoá), CourseComment + Reaction
- Tương tác user trong khoá học.

## 7. Use case — module Plan / PlanCourse / TrainingPlan

### CreatePlanCommand
| Business rule | Date range hợp lệ; user là portal admin |
| Side-effect | Insert `Plan`; publish event |

### AddCourseToPlan / RemoveCourseFromPlan
| Business rule | Course phải ở status Approved; không duplicate `(PlanId, CourseId)` |
| Side-effect | Insert/Remove `PlanCourse`; recalculate plan progress |

## 8. Use case — module MarkPointCourse (chấm điểm khoá)

### MarkPointEssayCommand
| Input | `{CourseContentUserId, UserPoint (0-100), Comment}` |
| Validator | Point range 0-100 |
| Business rule | Content phải là essay; user phải có bài nộp; cập nhật `CourseUser.TotalPoint` tổng hợp |
| Side-effect | Update `CourseContentUser`; nếu đạt điểm pass + đủ progress → publish `SetCertificationIntegrationEvent` (SharedServices cấp chứng chỉ); update proficiency của user qua event |

## 9. Use case — module ContentRollCall (điểm danh offline)

### RollCallStudentCommand
| Input | `{ContentRollCallId, UserIds[], Status (present/absent)}` |
| Side-effect | Insert `ContentRollCallUser`; cập nhật attendance % |

## 10. Use case — module CourseShare

| Cách dùng | Khi 2 portal chia sẻ chung 1 khoá học gốc |
| Business rule | Source portal admin approve share request |

## 11. Integration events publish

| Event | Trigger |
|---|---|
| `CourseCreatedIntegrationEvent` | Sau approve |
| `CourseUpdatedIntegrationEvent` | Update sau approve |
| `DeleteCourseIntegrationEvent` | Delete |
| `DeleteProficiencyLevelCourseIntegrationEvent` | Delete cascade |
| `DeleteExamTestEvent` | Delete cascade → Question |
| `AddUserToCourseNotificationIntegrationEvent` | Add user (SignalR push) |
| `SendEmailCourseIntegrationEvent` | Add user (MailSender) |
| `CourseCompletedIntegrationEvent` | User hoàn thành |
| `SetCertificationIntegrationEvent` | Đủ điều kiện cấp chứng chỉ |
| `CourseRecalledIntegrationEvent` | Recall |
| `ContentArchiveCreated/UpdatedIntegrationEvent` | Archive |

## 12. Integration events subscribe

- `UserCreated/Updated/DeletedIntegrationEvent` — sync user reference local.
- `ExamCompletedIntegrationEvent` — cập nhật progress khoá học có exam.
- `CertificationIssuedIntegrationEvent` — cập nhật certificate count.
- `GroupUserAdded/Removed` — auto enrol/un-enrol user vào course gắn group.

## 13. Dependency

- SQL Server (Course DB, 44 bảng).
- RabbitMQ.
- UserService (qua event hoặc HTTP query).
- QuestionService (cho course test).
- ServerFiles (lưu SCORM/video/PDF).
- SharedServices (cấp certificate, library link).

## 14. Cross-link
- [user-service.md](user-service.md)
- [question-service.md](question-service.md)
- [training-route-service.md](training-route-service.md)
- [../04-database-schema.md](../04-database-schema.md#3-courseservice-db)
