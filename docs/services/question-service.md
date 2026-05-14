# QuestionService — Deep Dive

> Path: `CLS4.0-QuestionService/src/Services/Question/Question.API/`. Aggregate root: `Question`, `Exam`, `Test`, `TestCode`, `CourseContentTest`, `Survey`. **59 bảng SQL** — service phức tạp nhất về schema.

## 1. Use case — module QuestionApp

### CreateQuestionCommand
| Mục | Nội dung |
|---|---|
| Actor | Giảng viên / Admin |
| Input | `{TopicId, QuestionLevelId, Questions[]{Content, TypeId, Answers[]{Content, IsCorrect}, ...}, IsSent, IsArchive}` |
| Validator | `QuestionLevelId` tồn tại; `QuestionTypeId` enum hợp lệ |
| Business rule | (a) Có thể tạo `QuestionGroup` cho nhiều question chia sẻ context (passage reading); (b) Bulk insert với graph (Question + Answer cùng lúc); (c) Nếu `IsArchive` true → đẩy sang content archive (SharedServices) |
| Side-effect | Bulk insert; log per question `QuestionActionType.Create` |

### UpdateQuestionCommand
| Business rule | Question tồn tại + thuộc Portal; nếu đã được dùng trong `TestCodeQuestion` đang `Approved` → không cho sửa core content (avoid corrupted exam) |

### DeleteQuestionCommand
| Input | `{ListId[]}` |
| Business rule | Reject question đang ở status không cho xoá; soft-delete `IsDeleted=true`; check không bị reference trong TestCode active |
| Side-effect | Bulk soft-delete + log per question |

### CreateQuestionExcelJsonCommand (bulk import Excel)
| Handler size | 25.9KB — handler phức tạp |
| Business rule | Parse Excel với template chuẩn; validate row-by-row (TopicId, LevelId, TypeId, Answers); skip duplicate Content; bulk create với answer |

## 2. Use case — module Exam

### CreateExamCommand
| Input | `{Name, Code, TimeLimit, StartDate, EndDate, IsShuffle, TestCodes[], ...}` |
| Business rule | Date range hợp lệ; `Code` unique per Portal; status default = Draft |
| Side-effect | Insert `Exam` + `Test` + `TestCode` chính (nếu compose ngay) |

### ApproveExamCommand
| Actor | Approver |
| Business rule | (a) `StatusId = Pending`; (b) User là approver; (c) Compose hợp lệ (đủ câu hỏi mỗi level theo `TestCodeStructure`) |
| Side-effect | Set `Approved`; publish `ExamApprovedIntegrationEvent` (TrainingRoute notify) |

### RecallExamCommand
| Business rule | Revert `Approved → Editing`; phải còn trước `StartDate` (chưa thi) |

### DeleteExamCommand
| Side-effect | Soft-delete + cascade `Test`, `TestCode`, `TestLearner` |

## 3. Use case — module TestCode (mã đề)

### AddTestCodeFromArchiveCommand
| Business rule | Lấy question từ archive theo `TestCodeStructure` (số câu / level); random theo seed |
| Side-effect | Insert `TestCode` + `TestCodeQuestion` + `TestCodeQuestionAnswer` |

### AssignTestCodeCommand (gán TestCode vào Shift)
| Side-effect | Insert `TestCodeShift` map TestCode ↔ Shift |

### TestCodeQuestion — assign question manually
| Business rule | Question phải Approved + cùng PortalId; check không trùng trong TestCode |

## 4. Use case — module CourseContentTest (test gắn course content)

### CreateCourseContentTestCommand
| Input | `{CourseContentId, IsShuffleQuestion, MinuteOfWork, PointRatio, IsDisplayResult, IsFullScreen, IsAllowPageSwitch, ViolationLimit, IsCompleteOnSufficientPoints}` |
| Business rule | CourseContent type phải là Quiz; chỉ 1 test per content (upsert) |
| Side-effect | Insert/update `CourseContentTest` + `CourseContentTestQuestion` |

### Settings đáng chú ý
- `IsFullScreen` — bắt fullscreen để giảm cheat.
- `IsAllowPageSwitch` — false = mỗi violation tab-switch trừ điểm.
- `ViolationLimit` — quá ngưỡng → auto submit + đánh dấu vi phạm.
- `IsCompleteOnSufficientPoints` — đạt điểm pass thì auto end exam.

## 5. Use case — module MyExam (học viên làm bài)

### ExamineeStartExamCommand
| Actor | Học viên |
| Input | `{ExamineeId, ShiftId}` |
| Business rule | (a) Đúng thời gian shift; (b) Học viên thuộc shift `TestLearner`; (c) Nhận `TestCode` theo random rule; (d) Init timer |
| Side-effect | Insert `CourseContentTestUser` hoặc `TestLearnerMap`; publish `UserStartedExamIntegrationEvent` (SignalR + LogService) |

### SaveAnswerCommand (auto-save khi user chọn)
| Input | `{TestUserId, QuestionId, AnswerIds[], EssayContent}` |
| Business rule | Test đang chạy + chưa hết giờ; question thuộc TestCode đó |
| Side-effect | Upsert `CourseContentTestUserQuestionAnswer` / `CourseContentTestUserQuestionEssay` |

### SubmitExamCommand
| Business rule | Test đang chạy; sau submit không cho sửa |
| Side-effect | Auto-grade question choice/fill; essay đẩy sang giảng viên (`CourseContentTestUserQuestionEssay`); compute partial score; publish `ExamCompletedIntegrationEvent` |

## 6. Use case — module MarkPoint (chấm điểm)

### MarkPointEssayCommand
| Actor | Giảng viên |
| Input | `{Items[]{QuestionEssayId, UserPoint (0-100), Comment}}` |
| Business rule | (a) Giảng viên thuộc Test; (b) Essay tồn tại + chưa chấm xong; (c) Recalculate `TestLearner.TotalPoint` |
| Side-effect | Update `CourseContentTestUserQuestionEssay.Point`; trigger pass/fail; nếu pass + đủ progress → publish `SetCertificationIntegrationEvent` |

### AutoMark — tự động chấm question loại single/multi/fill khi user submit, trong cùng `SubmitExamCommand` handler.

## 7. Use case — module Survey

### CreateSurveyCommand
| Input | `{Content, TopicId, TypeId, Answers[], TitleRating, CategoryRatings[], LevelRatings[]}` |
| Business rule | TypeId enum (multiple choice / rating / matrix / open); answer cần ≥1 nếu không phải rating |
| Side-effect | Insert `Survey` + `SurveyChoice` + `SurveyCategoryRating` + `SurveyLevelRating` |

### ApproveSurveyCommand
| Business rule | Status chuyển `Pending → Approved`; chỉ approver mới được duyệt |

### CourseContentSurvey — gắn survey vào course content
| Side-effect | Insert `CourseContentSurvey` + question/answer link |

## 8. Use case — module ExamSupervising (giám sát thi)

### ReportViolationCommand
| Trigger | SignalR `ExamHub` push (tab-switch, no-face, multi-face, screen exit) |
| Business rule | `ViolationLimit` đếm; quá ngưỡng → auto end exam + đánh dấu vi phạm |
| Side-effect | Insert `Supervision` log; publish `ViolateDetectedIntegrationEvent` (LogService lưu Mongo + ServerFiles lưu snapshot) |

## 9. Integration events publish

| Event | Trigger |
|---|---|
| `ExamCreatedIntegrationEvent` | CreateExam |
| `ExamApprovedIntegrationEvent` | ApproveExam |
| `ExamCompletedIntegrationEvent` | SubmitExam, UserCompleted |
| `UserStartedExamIntegrationEvent` | StartExam |
| `QuestionImportedIntegrationEvent` | ImportExcel |
| `ViolateDetectedIntegrationEvent` | ReportViolation |
| `SetCertificationIntegrationEvent` | Đủ điều kiện chứng chỉ |
| `MarkPointUpdatedIntegrationEvent` | Chấm essay |
| `SendExamResultEmailIntegrationEvent` | Sau submit (MailSender) |

## 10. Integration events subscribe

- `DeleteCourseIntegrationEvent` — xoá test/exam liên kết course.
- `DeleteExamTestEvent` — bị trigger từ Course Service.
- `UserAssignedToExamIntegrationEvent` — từ TrainingRoute (route step exam).
- `RenewedTestCodeIntegrationEvent` — re-issue TestCode khi user disconnect+reconnect.

## 11. Logic đặc thù

### Random TestCode mỗi học viên
- Mỗi `Test` có `TestCodeStructure` định số câu/level.
- Mỗi `TestCode` là 1 instance random từ archive.
- Khi học viên start → assign 1 TestCode (random từ pool theo seed = userId+shiftId).
- Cùng ca thi nhưng mỗi người TestCode khác → chống quay cóp.

### Anti-cheat
- SignalR `ExamHub` push event tab-switch / no-face mỗi 5s.
- Backend tích luỹ violation; quá `ViolationLimit` → auto submit + log.
- Webcam snapshot lưu qua ServerFiles, path lưu `ViolateImageUser` (LogService Mongo).

### Auto-grade
- Single/Multi choice: so sánh `AnswerIds[]` với `IsCorrect` flag → match = đủ điểm.
- Fill-in: so sánh text (case-insensitive, trim).
- Essay: defer cho giảng viên chấm tay.

## 12. Dependency

- SQL Server (Question DB, 59 bảng).
- RabbitMQ.
- ServerFiles (đính kèm question + snapshot vi phạm).
- LogService (audit violate).
- CourseService (sync course + content).
- UserService (validate learner).
- SignalR ExamHub.

## 13. Gap / improvement

- `UpdateQuestion` nên hoàn toàn cấm sửa nếu question đang dùng exam đang chạy (tránh inconsistency).
- Random TestCode hiện seed-based — nên có pre-compute pool size validation để chắc đủ câu.
- Auto-grade fill-in chỉ exact match — nên hỗ trợ regex hoặc fuzzy.
- Anti-cheat backend không phát hiện được DevTools mở; cần thêm signal frontend.

## 14. Cross-link

- [course-service.md](course-service.md) — course content test.
- [signalr.md](signalr.md) — ExamHub.
- [log-service.md](log-service.md) — violate log.
- [server-files.md](server-files.md) — snapshot webcam.
- [../04-database-schema.md](../04-database-schema.md#4-questionservice-db)
