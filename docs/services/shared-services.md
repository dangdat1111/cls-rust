# SharedServices — Deep Dive

> Path: `CLS-SharedServices/src/Services/Shared/Shared.API/`. Aggregate root: `Article`, `Library`, `Certification`, `Gift`, `Theme`, `Topic`, `Appellation`. **40 bảng SQL**.
>
> Tài nguyên dùng chung cross-service (article, library, certification, gift, theme, topic, rating scale).

## 1. Use case — module ArticleApp

### CreateArticleCommand
| Mục | Nội dung |
|---|---|
| Input | `{Name, Content, TopicIds[], StatusId, IsDisplay, UrlFiles[]}` |
| Business rule | `StatusId ∈ ArticleEnum`; `TopicIds` tồn tại; auto-approve nếu user là Admin và có Forum permission (bit 128); ngược lại pending nếu setting default tắt auto-approve |
| Side-effect | Insert `Article` + `ArticleTopic` (map topics); publish `ArticleCreatedIntegrationEvent` |
| Dependency | `IArticleRepository`, `ITopicRepository`, `SettingSystemQueries`, `HttpContextAccessor` |

### UpdateArticleCommand
| Business rule | Article tồn tại; user có quyền edit (Admin hoặc owner); refresh topics |

### ApproveArticleCommand
| Validator | Requires Forum permission bit 128 cho Approve/Decline; Edit bit 4 cho Pending |
| Side-effect | Bulk update `ApproveStatus` + `UserId` (approver) + `Note` |

### DeleteArticleCommand
| Side-effect | Bulk soft-delete; cascade clean `ArticleTopic`, `ArticleComment` |

## 2. Use case — module ArticleComment / Reaction

### CreateCommentArticleCommand
| Input | `{ArticleId, ParentId, Content}` |
| Business rule | Article tồn tại + active; `ParentId > 0` → child comment (threading) |
| Side-effect | Insert `ArticleComment`; notification cho article owner |

### UpdateCommentArticleCommand / DeleteCommentArticleCommand
| Business rule | Chỉ owner hoặc admin |

### Article(Comment)Reaction — like/heart
| Side-effect | Toggle reaction per user |

## 3. Use case — module Library

### CreateContentToLibraryCommand
| Input | `{Name, Content, TopicId, ContentTypeId, StatusId, OrganizationalStructureIds[], IsDisplayHome, DisplayId}` |
| Business rule | `ContentTypeId ∈ LibraryContentTypeEnum`; OrgStruct list = visibility scope; `DisplayId` = visibility flag |
| Side-effect | Insert `Library` + `LibraryOrganizationalStructure`; publish `LibraryCreatedIntegrationEvent` |

### UpdateContentLibraryCommand
| Side-effect | Update + refresh OrgStruct map |

### InspectContentLibraryCommand (audit access)
| Side-effect | Insert audit entry; update `Library.InspectedDate` |

### RecallContentLibraryCommand
| Business rule | Soft-recall, preserve history |
| Side-effect | `IsRecalled = true`, `RecalledDate` |

### DeleteContentLibraryCommand
| Side-effect | Hard-delete; cleanup `LibraryComment`, `LibraryReaction`, `LibraryUserRating` |

### LibraryBookmark, LibraryComment, LibraryUserRating
- Tương tác user, follow pattern tương tự Article.

## 4. Use case — module Certification

### CreateCourseCertificationCommand
| Input | `{Name, Description, TemplateId, ResourceId (CourseId/RouteId), TypeId}` |
| Business rule | (a) `TemplateId` tồn tại trong `CertificationTemplate`; (b) `ResourceId` tồn tại cross-service (Course/Route) |
| Side-effect | Insert `Certification`; publish `CertificationSavedIntegrationEvent` |

### CreateTrainingRouteCertificationCommand
Tương tự nhưng `ResourceType = TrainingRoute`.

### UpdateCertificationCommand
| Business rule | TemplateId immutable sau khi đã cấp cho user (giữ history) |

### DeleteCertificationCommand
| Business rule | Cảnh báo nếu đã `CertificationUser` đang dùng — soft-delete, không hard-delete |

## 5. Use case — module CertificationTemplate

### CreateCertificationTemplateCommand
| Input | `{Title, Background, Layout, Fields[]}` |
| Business rule | Template structure (vị trí name, course, date, signature) |
| Side-effect | Insert `CertificationTemplate`; immutable sau khi có certificate dùng template này |

### SetDefaultCertificationTemplateCommand
| Business rule | Chỉ 1 default per organization; cascade clear default cũ |

## 6. Use case — module CertificationUser

### CreateCourseCertificationAsync (issue khi user complete)
| Trigger | Event `SetCertificationIntegrationEvent` từ Course/Question/TrainingRoute |
| Input | `{UserId, CertificationId, GrandFrom, ExpiryDate?}` |
| Business rule | Check user chưa có cert (hoặc cho phép re-issue); generate code unique |
| Side-effect | Insert `CertificationUser`; publish `CertificationIssuedIntegrationEvent`; trigger MailSender gửi cert |

### AssignExternalCertificationAsync (cert ngoài)
| Input | `{UserId, ExternalCertificationName, Company, ImportDate}` |
| Side-effect | Insert `ExternalCertification` |

### ImportListUserToCertificationFromExcelAsync
| Side-effect | Batch import từ Excel; gọi UserService validate userId; bulk insert |

## 7. Use case — module Gift / GiftUser

### CreateGiftAppellationCommand
| Input | `{AppellationId, Name, Number (stock), Notification}` |
| Side-effect | Insert `Gift` |

### AddGiftForUserCommand
| Input | `{GiftId, UserIds[]}` |
| Business rule | Stock đủ (`Number ≥ count(UserIds)`); decrement stock |
| Side-effect | Insert `GiftUser`; publish `GiftAssignedIntegrationEvent` |

### UpdateStatusGiftUserCommand
| `StatusId ∈ {Pending, Accepted, Rejected, Expired}` |

## 8. Use case — module Theme

### CreateThemeCommand
| Input | `{Name, Items (color palette KeyValue), ExamImages}` |
| Side-effect | Insert `Theme` + `ThemeDetail` |

### SetActiveThemeCommand
| Business rule | 1 active per portal; cascade clear active cũ |
| Side-effect | Update `ThemePortal.IsActive` |

## 9. Use case — module Topic (tree topic dùng chung)

### CreateTopicCommand
| Input | `{Name, ParentId, TypeId, Icon}` |
| Business rule | Tree hierarchy; `Name` unique cùng cấp + cùng `TypeId` |
| Side-effect | Insert `Topic`; publish `TopicCreatedIntegrationEvent` (cache invalidate) |

### ImportExcelTopicCommand
| Side-effect | Batch import với hierarchy validation |

## 10. Use case — module Appellation / GroupAppellation

### CreateAppellationGroupCommand
| Side-effect | Insert `AppellationGroup` |

### UpdateAppellationCommand / DeleteAppellationCommand
- CRUD đơn giản.

### GroupAppellationOrganizationalStructure — map danh xưng cho OrgStruct
- Cấu hình danh xưng mặc định cho phòng ban.

## 11. Use case — module RatingScale

### CreateRatingScaleCommand
| Input | `{Name, Description, Levels[]{Name, Value}}` |
| Side-effect | Insert `RatingScale` + `RatingScaleLevel` |

## 12. Use case — module Reel (TikTok-style video)

### CreateReelCommand
| Side-effect | Insert `Reel`; reaction + comment ngắn |

## 13. Use case — module SettingSystem

### UpdateSettingDefaultCommand
| Business rule | Setting cấp portal, key-value pair, override default toàn hệ |

## 14. Use case — module CostType, ExternalCertification, Tool
- Tool: `ToolAsync` chạy raw SQL — **giới hạn role admin only** (security concern, hiện chưa thấy auth filter).

## 15. Integration events publish

| Event | Trigger |
|---|---|
| `ArticleCreatedIntegrationEvent` | Create article |
| `ArticlePublishedIntegrationEvent` | Approve article |
| `LibraryCreatedIntegrationEvent` | Add library content |
| `CertificationSavedIntegrationEvent` | CRUD certification |
| `CertificationIssuedIntegrationEvent` | Issue user cert |
| `GiftAssignedIntegrationEvent` | Add gift for user |
| `TopicCreatedIntegrationEvent` | Create topic |

## 16. Integration events subscribe

- `SetCertificationIntegrationEvent` từ Course/Route → tự cấp `CertificationUser`.
- `UserDeletedIntegrationEvent` → cleanup user reference trong Article/Library/Cert.
- `CourseCompletedIntegrationEvent` → trigger condition check cấp cert.

## 17. Dependency

- SQL Server (Shared DB, 40 bảng).
- RabbitMQ.
- ServerFiles (file đính kèm article/library, render PDF certificate).
- UserService (validate user cross-service).
- CourseService / TrainingRoute (ResourceId reference cert).
- SystemService (setting default).

## 18. Gap

- `ToolController.ToolAsync` chạy SQL raw không có auth check rõ ràng → security risk.
- Library `RecallContent` chỉ flag, không có UI để rollback recall.
- Topic hierarchy có thể vô tận → khuyến nghị giới hạn depth.

## 19. Cross-link

- [course-service.md](course-service.md)
- [training-route-service.md](training-route-service.md)
- [system-service.md](system-service.md)
- [../04-database-schema.md](../04-database-schema.md#5-sharedservices-db)
