# UserService — Deep Dive

> Path: `CLS4.0-UserService/src/User.API/`. Aggregate root: `ApplicationUser`, `Portal`, `Management`, `Group`, `OrganizationalStructure`, `Feature`, `Customer`. **64 bảng SQL**.
>
> Service trung tâm: mọi service khác query/sync user qua đây. IdentityServer cũng đọc DB này.

## 1. Use case bảng — module UserApp

### CreateUserCommand
| Mục | Nội dung |
|---|---|
| Actor | Admin portal |
| Tiền điều kiện | Admin có quyền `Module=User, Level=Create` |
| Input | `{FirstName, LastName, UserName, Email, Password, StatusId, UserTypeId, Workplace, Address, GroupIds[], OrgStructureIds[], AcademicDegreeId?, LevelId?, PayrollScaleId?, ...}` |
| Validator | FirstName/LastName max 50, UserName 4-50, Password ≥6, Email format, max 100; required: StatusId, UserTypeId |
| Business rule | (a) Duplicate email + username trong cùng `ManagementId`; (b) Auto-gen `UserCode` nếu rỗng; (c) Validate `UserTypeId`, `StatusId`, `AcademicDegreeId`, `LevelId`, `PayrollScaleId` tồn tại; (d) Check quyền assign user vào org-structure/group |
| Output | `JsonResponse<int>` = UserId |
| Side-effect | Insert `ApplicationUser` (password đã hash); insert `ManagementUser`, `GroupUser`, `OrganizationalStructureUser`; log `UserActionType.Create`; publish `UserCreatedIntegrationEvent` |

### UpdateUserCommand
| Mục | Nội dung |
|---|---|
| Validator | Tương tự Create + Id required |
| Business rule | User tồn tại + thuộc `ManagementId`; email phải vẫn unique nếu đổi; update education/experience sub-list |
| Side-effect | Merge update; refresh GroupUser/OrgStructUser; log `UserActionType.Update`; publish `UserUpdatedIntegrationEvent` |

### DeleteUserCommand
| Mục | Nội dung |
|---|---|
| Input | `{ListId[]}` |
| Business rule | (a) Không cho xoá user đang tạo lệnh (current user); (b) Không cho xoá user gốc tạo Portal; (c) Soft-delete `StatusId=4` + `IsDeleted=true`; (d) Cascade xoá `GroupUser`, `OrganizationalStructureUser` |
| Side-effect | Soft-delete; log per user; publish `UserDeletedIntegrationEvent` |

### ChangePasswordCommand
| Mục | Nội dung |
|---|---|
| Validator | Password mới min 6 max 50, phải có chữ hoa/thường/số/ký tự đặc biệt, `ConfirmPassword == NewPassword` |
| Business rule | Hash + so sánh password hiện tại; reject nếu `NewPassword == OldPassword` |
| Side-effect | Update `User.Password` (hash mới). **Không** revoke JWT đã cấp — đây là gap, nên publish event để IdentityServer revoke `PersistedGrant`. |

### ResetPasswordCommand
| Mục | Nội dung |
|---|---|
| Input | `{SecurityCode, Password, ConfirmPassword}` |
| Business rule | Lookup user theo `SecurityCode` (token gửi qua mail); reject nếu code invalid/expired/đã dùng |
| Side-effect | Update password hash; clear `User.SecurityCode`; publish `ResetPasswordSuccess` event |

### ConfirmEmailCommand
| Mục | Nội dung |
|---|---|
| Input | `{Email, ConfirmCode}` |
| Business rule | Validate code khớp (tính từ `SecretKey:EmailConfirm`); chưa confirm trước đó |
| Side-effect | `User.EmailConfirmed = true` |

### ExcelCreateUsersCommand (bulk import)
| Mục | Nội dung |
|---|---|
| Input | `{Users[], IsSave (false=validate only, true=commit)}` |
| Business rule | Bulk validate trước, trả lỗi per-row; nếu IsSave thì insert; auto-gen UserCode; skip duplicate |
| Side-effect | Bulk insert; ManagementUser link; publish event per user; log batch |
| Output | `List<ExcelCreateUserModel>` kèm lỗi từng dòng |

## 2. Use case — module AuthenticationApp

### LoginCommand
| Mục | Nội dung |
|---|---|
| Input | `{Email, Password, PortalDomain (từ Referer)}` |
| Business rule | (a) Case-insensitive email; (b) Verify password hash; (c) `StatusId = active`; (d) Validate portal/domain từ Referer; (e) Optional device tracking |
| Side-effect | Insert `LoginHistory`; update `User.LastLogin`; insert `ActivityUser`; gen JWT (gọi IdentityServer hoặc cấp local nếu có); publish `LoginSuccessIntegrationEvent` |
| Output | `LoginResultDto{accessToken, refreshToken, user, portal, role}` |

### RefreshTokenCommand
| Business rule | Validate signature + expiry refresh token; user vẫn active; gen new access token |

### AuthenticationCommand (social/external)
| Input | `{Token, Url, Provider}` |
| Business rule | (a) Validate external token với provider (MSAL/Google); (b) Lookup external identity hoặc auto-create user nếu portal cho phép self-register; (c) Link external identity |
| Side-effect | Insert/update `ExternalLogin`; create user nếu mới; cấp token nội bộ |

## 3. Use case — module Group / UserGroup

### CreateGroupCommand
| Business rule | `Code` unique per `ManagementId`; validate `GroupTypeId` |
| Side-effect | Insert `Group`; publish `GroupCreatedIntegrationEvent` |

### UserAddGroupCommand (add multiple users into group)
| Business rule | Group tồn tại + cùng `ManagementId`; skip user đã trong group; optional flag cascade `IsCourse/IsExam/IsTraining` để gán user vào course/exam/route gắn group đó |
| Side-effect | Bulk `GroupUser`; cascade trigger qua các service tương ứng; publish event per user |

### UserRemoveGroupCommand
| Side-effect | Soft-delete `GroupUser`; cascade remove course/exam/route assignment nếu được flag |

## 4. Use case — module OrganizationalStructure

### CreateOrganizationalStructureCommand
| Input | `{Name, Code, ParentId, AbilityId, OwnerId?}` |
| Business rule | `Code/Name` unique per `ManagementId`; `ParentId` tồn tại (tree); validate `AbilityId` framework |
| Side-effect | Insert; cập nhật left/right index nếu dùng nested set; publish event |

### UserAddOrganizationalStructureCommand
| Business rule | Org tồn tại; user thuộc ManagementId; flag `IsCourse/IsExam/IsTraining` để cascade |
| Side-effect | Insert `OrganizationalStructureUser`; cascade link cross-service |

### UserDeleteOrganizationalStructureCommand
Tương tự, soft-delete + cascade remove cross-service link nếu flag set.

## 5. Use case — module Customer (self-register flow)

### CreateCustomerCommand
| Actor | Người dùng tự đăng ký portal |
| Business rule | Tạo entity `Customer` (pending) — **chưa** tạo User; check email/username unique; lấy Portal từ Host header |
| Side-effect | Insert `Customer`; gửi email confirm; publish `CustomerRegisteredIntegrationEvent` |

### ApproveCustomerCommand
| Actor | Admin portal |
| Business rule | Tạo `ApplicationUser` từ data Customer; xoá Customer; assign UserType, OrgId, GroupId, TitleId optional |
| Side-effect | Insert User + ManagementUser + (OrgStruct/Group/Title) link; publish `CustomerApprovedIntegrationEvent` (sẽ trigger MailSender gửi mail welcome) |

## 6. Use case — module Feature (multi-tenant feature toggle)

### CreateFeatureGroupCommand / UpdateFeatureDefaultCommand
| Business rule | Title required; `Admin/Teacher/Learner` bit cho phép vai trò nào thấy; ParentId cho hierarchy |
| Side-effect | Insert/update `FeatureGroup`/`FeatureDefault`; publish `FeatureUpdatedEvent` để FE refresh menu |

### Toggle feature theo Portal
- Admin bật/tắt Feature cho từng Portal qua `UserTypeFeature` + `Feature` (per-portal record).
- Frontend đọc qua JWT claim `feature[]` để show/hide menu.

## 7. Use case — module Credit / Order

### CreateCreditCommand
| Validator | Name unique per `ManagementId`; `CurrencyId` tồn tại; numeric ≥0 (Memory, MonthPrice, YearPrice, TotalCourse, TotalUser) |
| Side-effect | Insert `Credit` (gói credit/pricing plan); publish event |

### UpdateCreditCommand
| Business rule | Lookup theo Id, chưa soft-delete, name unique excluding self |

## 8. Use case — module Calendar

### CreateEvent / Add Group/User vào event
| Side-effect | Insert `EventCalendar` + map `EventCalendarUser/Group/OrgStruct`; trigger notification qua RabbitMQ |

## 9. Use case — module Proficiency

### AssignProficiencyToUser
| Business rule | `Proficiency` + `ProficiencyLevel` tồn tại; gán cho user theo `ProficiencyUser` |
| Side-effect | Insert `ProficiencyUser`; publish `ProficiencyAssignedEvent` |

## 10. Integration events publish

| Event | Trigger |
|---|---|
| `UserCreatedIntegrationEvent` | CreateUser, ExcelCreateUsers, ApproveCustomer |
| `UserUpdatedIntegrationEvent` | UpdateUser |
| `UserDeletedIntegrationEvent` | DeleteUser |
| `UserAddedToGroupIntegrationEvent` | UserAddGroup |
| `UserRemovedFromGroupIntegrationEvent` | UserDeleteFromGroup |
| `UserAddedToOrgStructureEvent` | OrgStruct add user |
| `LoginSuccessIntegrationEvent` | Login |
| `CustomerRegisteredIntegrationEvent` | CreateCustomer |
| `CustomerApprovedIntegrationEvent` | ApproveCustomer |
| `PortalCreatedIntegrationEvent` | CreatePortal |
| `FeatureUpdatedEvent` | Update feature |
| `CreditCreated/UpdatedIntegrationEvent` | CRUD Credit |
| `ResetPasswordSuccess` | ResetPassword |

## 11. Integration events subscribe

- `CourseEnrolledIntegrationEvent` — cập nhật scoreboard/dashboard user.
- `CertificationIssuedIntegrationEvent` — count cert per user.
- `ExamCompletedIntegrationEvent` — cập nhật profile học vấn.

## 12. Dependency

- SQL Server (UserService DB, 64 bảng).
- RabbitMQ.
- IdentityServer (đọc UserService DB cho authenticate).
- Captcha key (recaptcha).
- Hash key (`SecretKey:HashPassword`, `SecretKey:EmailConfirm`).
- MailSender (gián tiếp qua event).

## 13. Gap / improvement

- ChangePassword không revoke token đã cấp → security gap.
- Soft delete `StatusId=4` không loại bỏ hẳn khỏi reference → cần index `IsDeleted` cho query.
- Captcha mới ở frontend, backend không enforce → bot có thể bypass.
- Bulk import 1000+ user dễ time-out HTTP → nên chuyển sang background job + return job-id.

## 14. Cross-link

- [identity-server.md](identity-server.md)
- [course-service.md](course-service.md) — sync user via event.
- [../02-backend-feature-catalog.md](../02-backend-feature-catalog.md#2-userservice-cls40-userservice)
- [../04-database-schema.md](../04-database-schema.md#2-userservice-db)
