# TrainingRouteService — Deep Dive

> Path: `CLS4.0-TrainingRouteService/src/Services/TrainingRoute/TrainingRoute.API/`. Aggregate root: `TrainingRoute`, `LearningStep`. **11 bảng SQL**.
>
> Lộ trình học tuần tự — chuỗi LearningStep (Course / Exam) bắt buộc theo thứ tự.

## 1. Use case — module TrainingRouteApp (lifecycle)

### CreateTrainingRouteCommand
| Mục | Nội dung |
|---|---|
| Actor | Admin portal / Manager |
| Input | `{Name, Description, StartDate, EndDate, CourseIds[], CertificationId?, OrganizationalStructureIds[]}` |
| Business rule | (a) `Name` unique trong portal; (b) `StartDate ≤ EndDate`; (c) `CourseIds` tồn tại + Approved (cross-service Course); (d) `CertificationId` tồn tại (Shared) |
| Status | Default `Draft` |
| Side-effect | Insert `TrainingRoute` + `LearningStep` per Course; publish `TrainingRouteCreatedIntegrationEvent` (cache + search index) |
| Dependency | `TrainingRouteRepository`, CourseService (qua HTTP/cache), `EventBus`, `Logger` |

### EditTrainingRouteCommand
| Business rule | Status phải là `Draft` hoặc `Approved` (không edit `Pending/Completed`); update Name, CourseIds, OrgStructIds |
| Side-effect | Update + refresh LearningStep list |

### DeleteTrainingRouteCommand
| Business rule | Chỉ xoá `Draft` hoặc `Declined`; cascade soft-delete `TrainingRouteUser`, `LearningStep`, `LearningStepUser` |
| Side-effect | Soft-delete; audit trail |

### PendingTrainingRouteCommand
| Business rule | Chuyển `Draft → Pending` (đợi approve) |
| Side-effect | Update status + `PendingDate` |

### ApproveTrainingRouteCommand
| Input | `{Ids[], IsAutomaticAddToCourses, IsAutomaticAddToExams}` (bulk) |
| Business rule | (a) Status = Pending; (b) User là approver |
| Side-effect | Set `Approved` + `ApprovedBy/Date`; nếu flag set → auto-enroll user route vào courses/exams; publish `TrainingRouteApprovedIntegrationEvent` (trigger CourseService/QuestionService enroll) |

### DenyTrainingRouteCommand
| Input | `{Ids[], Description (lý do)}` |
| Business rule | Status = Pending |
| Side-effect | Set `Declined` + Description |

### RecallRequestTrainingRouteCommand
| Business rule | Revert `Approved → Pending` để sửa; track `RequestedBy/Date`; preserve approval history |
| Side-effect | Update status + publish `TrainingRouteRecalledIntegrationEvent` |

### RecallTrainingRouteCommand
- Tương tự nhưng do approver gọi.

### SetCompleteTrainingRouteCommand
| Business rule | Mọi `TrainingRouteUser` đã complete → set `Status = Completed` cho route |
| Side-effect | Update `CompletedDate`; publish `TrainingRouteCompletedIntegrationEvent` |

## 2. Use case — module TrainingRouteApp (assignment)

### AddGroupToTrainingRouteCommand
| Input | `{TrainingRouteId, GroupIds[]}` |
| Business rule | Resolve `GroupId` → user list (qua UserService hoặc cache); create `TrainingRouteUser` per member |
| Side-effect | Insert `TrainingRouteGroupUser` + `TrainingRouteUser` (per user); cascade enroll cross-service |

### RemoveGroupFromTrainingRouteCommand
| Side-effect | Delete group mapping + cascade `TrainingRouteUser` removal cho member của group đó |

### AddUserToTrainingRouteCommand
| Input | `{TrainingRouteId, UserIds[], IsAutomaticAddToCourses, IsAutomaticAddToExams}` |
| Side-effect | Insert `TrainingRouteUser`; optional auto-enroll cross-service; publish `UserAddedToTrainingRouteIntegrationEvent` |

### RemoveUserFromTrainingRouteCommand
| Side-effect | Delete `TrainingRouteUser`; cascade disenroll khỏi course/exam (nếu flag) |

### UpdateOrganizationalStructureFromTrainingRouteCommand
| Side-effect | Update OrgStructIds → re-filter eligible users; remove user không còn thuộc org |

## 3. Use case — module TrainingRouteUserApp

### ImportUserFromEmailCommand
| Input | Excel file với column email |
| Business rule | Parse email → call `UserService.GetUserByEmail` → bulk insert `TrainingRouteUser` |
| Side-effect | Bulk insert; trả lỗi per row nếu email không tồn tại |

## 4. Use case — module LearningStep

### Tạo LearningStep
- Step tự động sinh khi thêm Course/Exam vào route.
- Type: `LearningStepCourse` hoặc `LearningStepExam`.
- `Order` quyết định thứ tự bắt buộc.

### LearningStepExamCourseRequire
- Định nghĩa course prereq trước exam step.
- Học viên chỉ unlock exam step khi đã hoàn thành course require.

### Progress tracking
- `LearningStepUser` — progress per user per step.
- `TrainingRouteReportUser` — materialized cho query nhanh.
- Update khi nhận event `CourseCompletedIntegrationEvent` / `ExamCompletedIntegrationEvent`.

## 5. Use case — module LearnerApp

### GetListTrainingRouteLearner
| Query | Trả route mà user được assign + progress hiện tại |
| Logic | JOIN `TrainingRoute` × `TrainingRouteUser` × `LearningStepUser` |

## 6. Reports (TrainingRouteApp queries)

Nhiều report endpoint:
- `TrainingRouteCostReport` — chi phí route
- `TrainingRouteStepReport` — progress per step
- `UserTrainingRouteReport` — per user
- `UserTrainingRouteStepCompleteReport` — bước hoàn thành
- `GetReportOrStructureDashboardExam` — dashboard
- `GetDashboardTrainingUser` — dashboard user
- `ExportTrainingRoutesReport` / `ExportTrainingRouteCostReport` — Excel export
- `ExcelTemplateAddUsers` — template để import user

## 7. Integration events publish

| Event | Trigger |
|---|---|
| `TrainingRouteCreatedIntegrationEvent` | Create |
| `TrainingRouteApprovedIntegrationEvent` | Approve (trigger cross-service enroll) |
| `TrainingRouteRecalledIntegrationEvent` | Recall |
| `TrainingRouteCompletedIntegrationEvent` | Complete |
| `UserAddedToTrainingRouteIntegrationEvent` | Add user |
| `LearningStepCompletedIntegrationEvent` | User complete 1 step |
| `SetCertificationIntegrationEvent` | Đủ điều kiện cấp cert sau route |

## 8. Integration events subscribe

- `CourseCompletedIntegrationEvent` — update LearningStepUser nếu step là course đó.
- `ExamCompletedIntegrationEvent` — tương tự cho exam step.
- `UserDeletedIntegrationEvent` — cleanup `TrainingRouteUser`.
- `CourseDeletedIntegrationEvent` — invalidate LearningStep liên quan.

## 9. Dependency

- SQL Server (TrainingRoute DB, 11 bảng).
- RabbitMQ.
- CourseService (validate course tồn tại + auto-enroll).
- QuestionService (auto-enroll exam).
- UserService (resolve user/group, import email).
- SharedServices (CertificationId reference).

## 10. Logic đặc thù

### Lock step theo thứ tự
- User chỉ thấy step kế tiếp sau khi complete step trước.
- Backend enforce ở `LearnerApp` query (filter step `Order ≤ currentMaxCompleteOrder + 1`).

### Auto-enroll cross-service
- Khi approve route + flag `IsAutomaticAddToCourses`: publish event → CourseService consume → tự `CourseUser` cho mọi user trong route.
- Tương tự cho exam.

### Báo cáo cost
- Cost route = sum cost của course/exam trong route.
- Query cross-service (qua Dapper read DB Course/Question).

## 11. Gap / improvement

- Không lock route khi đang chạy → admin có thể thay course giữa chừng, gây inconsistency với user đang học.
- Step prereq chỉ check 1-1 (course → exam) — chưa support DAG phức tạp.
- Cascade auto-enroll có thể chạy lâu khi route có nhiều user × nhiều course → nên chuyển sang background job.

## 12. Cross-link

- [course-service.md](course-service.md)
- [question-service.md](question-service.md)
- [shared-services.md](shared-services.md) — certification.
- [user-service.md](user-service.md) — group/org/user.
- [../04-database-schema.md](../04-database-schema.md#7-trainingrouteservice-db)
