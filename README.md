# CLS-Rust — Migrate LMS từ C# sang Rust

Đây là workspace Rust cho dự án **CLS 4.0 (LMS)** — rewrite toàn bộ backend .NET Core 3.1 sang Rust theo chiến lược Strangler-fig, giữ nguyên frontend Vue 2 và contract API/JWT/event không đổi.

---

## Hệ thống cũ (C# / .NET Core 3.1)

### Tổng quan

**CLS 4.0** là Learning Management System phục vụ đào tạo doanh nghiệp, multi-tenant theo Portal (mỗi khách hàng có portal riêng với domain, theme, feature toggle riêng).

Production: `lmsapi.doffice.com.vn` · `lmsadmin.doffice.com.vn` · `lmssignal.doffice.com.vn` · `lmssf.doffice.com.vn`

### Tính năng cốt lõi

- Quản lý khoá học + nội dung học (SCORM / video / PDF / essay / offline)
- Ngân hàng câu hỏi, đề thi, ca thi, chấm điểm tự động + chấm tay
- Lộ trình học (Training Route) tuần tự + Training Plan tổng thể
- Khảo sát, chứng chỉ, gift/reward gamification
- Báo cáo đa chiều (user / course / exam / lecturer / cost) + custom report
- Realtime: chat, notification, giám sát thi (anti-cheat)

### Danh sách 16 microservice

| # | Service | Vai trò | Độ phức tạp |
|---|---|---|---|
| 1 | IdentityServer | OAuth2/OIDC, cấp JWT, social login | Trung |
| 2 | APIGateway (Ocelot) | Route + JWT middleware + Swagger aggregation | Thấp |
| 3 | UserService | User / Portal / Group / Org / HR / Role | Cao |
| 4 | CourseService | Khoá học + nội dung + enrollment + plan | Cao |
| 5 | QuestionService | Ngân hàng câu hỏi + đề thi + ca thi + chấm điểm | Rất cao |
| 6 | SharedServices | Article / Library / Certification / Gift / Theme | Trung |
| 7 | SystemService | EmailTemplate / Widget / Custom Report | Trung |
| 8 | TrainingRouteService | Lộ trình học + LearningStep | Trung |
| 9 | SignalR | Hub realtime (notification/chat/exam) — Redis backplane | Cao |
| 10 | CommunicationService | Chat store + test log + exam log (Mongo) | Thấp |
| 11 | NotificationService | Mobile FCM push | Thấp |
| 12 | LogService | System activity / violate user / audit (Mongo) | Thấp |
| 13 | ServerFiles | Upload + SCORM serve + file transfer | Trung |
| 14 | MailSender | Hangfire background gửi mail | Thấp |
| 15 | ReportService | Báo cáo tổng hợp | Trung |
| 16 | Admin (Internal Gateway) | Cổng riêng cho admin | Thấp |

### Stack công nghệ C#

| Lớp | Công nghệ |
|---|---|
| Runtime | .NET Core 3.1 (EOL 12/2022) |
| Web framework | ASP.NET Core Web API |
| DI | Autofac |
| ORM / Query | EF Core 3.1 + Dapper + MongoDB.Driver |
| Mediator | MediatR + pipeline behaviors |
| Auth | IdentityServer4 + JWT Bearer |
| Gateway | Ocelot |
| Event Bus | RabbitMQ (custom `IEventBus` trong CLS4.0-Core) |
| Service Discovery | Steeltoe.Discovery.Eureka |
| Config Server | Consul KV |
| Realtime | SignalR + Redis backplane |
| Logging | Serilog + Elasticsearch sink |
| Background Job | Hangfire |
| DB | SQL Server + MongoDB + Redis |
| Frontend | Vue 2.6 + Vuex 3 + Vue Router 3 |
| Infra | Docker + Kubernetes + NGINX Ingress + GitLab CI |

---

## Lý do migrate sang Rust

| Lý do | Chi tiết |
|---|---|
| .NET Core 3.1 EOL | Microsoft kết thúc hỗ trợ 12/2022, không còn security patch |
| IdentityServer4 | Chuyển commercial từ v5 |
| SQL Server license | Đắt khi scale |
| Memory footprint | .NET ~150-300 MB/service vs Rust ~30-50 MB/service |
| Throughput | GC pause vs zero-cost abstraction |

---

## Mapping stack C# → Rust

| Thành phần C# | Tương đương Rust |
|---|---|
| ASP.NET Core | **Axum** (Tokio) |
| EF Core + SQL Server | **SQLx** + Tiberius driver |
| Dapper | SQLx raw query + `FromRow` derive |
| MongoDB.Driver | `mongodb` official crate |
| MediatR | Custom `CommandHandler<C, R>` trait + dispatcher |
| MediatR Behaviors | Tower middleware layers |
| Autofac DI | Axum application state + manual wiring |
| FluentValidation | `validator` crate |
| IdentityServer4 | **Keycloak** (out-of-process, Pha 6) |
| Ocelot Gateway | **Pingora** / Envoy (Pha 7) |
| SignalR Hub | `signalrs` community crate hoặc native WS |
| RabbitMQ EventBus | `lapin` + custom EventBus trait |
| Hangfire | **Apalis** + tokio-cron-scheduler |
| Serilog | `tracing` + tracing-subscriber |
| Elasticsearch sink | OpenTelemetry OTLP → **SigNoz** (tracing + log + metric) |
| Swashbuckle | `utoipa` |
| Consul config | `consul-rs` + `config` crate |
| Eureka | Bỏ — dùng K8s DNS service discovery |
| AutoMapper | `From` / `TryFrom` trait |

---

## Observability — SigNoz

SigNoz là nền tảng observability open-source, self-hosted, tích hợp ba pillar — **trace / log / metric** — trong một UI duy nhất. Thay thế bộ ba Elasticsearch + Kibana + Elastic APM của hệ thống cũ.

### Tại sao SigNoz

| Tiêu chí | Elastic APM / ELK | SigNoz |
|---|---|---|
| License | Elastic (non-SSPL với cloud) | Apache 2 / MIT |
| Self-hosted cost | RAM ~4-8 GB cho ELK stack | ~2 GB (ClickHouse backend) |
| OTLP native | Qua Logstash/Agent | Có sẵn, zero config |
| Correlate trace↔log | Manual | Built-in (trace_id tự động) |
| UI | Kibana tách biệt | Unified (Trace · Log · Metric · Alert) |

### Kiến trúc tích hợp

```
Rust Service
  │  (opentelemetry-otlp)
  ├── Trace (spans)  ──┐
  ├── Log  (tracing) ──┤─→ OpenTelemetry Collector ──→ SigNoz (ClickHouse)
  └── Metric (gauge) ──┘                                      │
                                                        SigNoz UI
                                                   (query + alert + dashboard)
```

Mỗi service Rust export OTLP qua gRPC `http://otel-collector:4317`. Collector chạy sidecar hoặc DaemonSet trên K8s.

### Crates sử dụng trong `cls-observability`

```toml
[dependencies]
tracing                     = "0.1"
tracing-subscriber          = { version = "0.3", features = ["env-filter", "json"] }
opentelemetry               = { version = "0.22", features = ["trace"] }
opentelemetry_sdk           = { version = "0.22", features = ["trace", "rt-tokio"] }
opentelemetry-otlp          = { version = "0.15", features = ["grpc-tonic", "logs", "metrics"] }
opentelemetry-semantic-conventions = "0.14"
tracing-opentelemetry       = "0.23"   # bridge tracing → OTel spans
```

### Khởi tạo trong service

```rust
// crates/cls-observability/src/lib.rs
pub fn init(service_name: &'static str) -> OtelGuard {
    let exporter = opentelemetry_otlp::new_exporter()
        .tonic()
        .with_endpoint("http://otel-collector:4317");

    let tracer = opentelemetry_otlp::new_pipeline()
        .tracing()
        .with_exporter(exporter)
        .with_trace_config(
            opentelemetry_sdk::trace::config()
                .with_resource(Resource::new(vec![
                    KeyValue::new("service.name", service_name),
                    KeyValue::new("deployment.environment", env_or("production")),
                ]))
        )
        .install_batch(opentelemetry_sdk::runtime::Tokio)
        .expect("OTel tracer");

    let subscriber = tracing_subscriber::registry()
        .with(EnvFilter::from_default_env())
        .with(tracing_subscriber::fmt::layer().json())        // log → stdout (collector scrape)
        .with(tracing_opentelemetry::layer().with_tracer(tracer));

    tracing::subscriber::set_global_default(subscriber).expect("set subscriber");

    OtelGuard   // Drop-guard gọi shutdown_tracer_provider()
}
```

Mỗi service gọi `cls_observability::init("mailsender")` ở đầu `main()`.

### Instrumentation Axum

```rust
// Tự động tạo span cho mỗi HTTP request
use axum_tracing_opentelemetry::middleware::OtelAxumLayer;

let app = Router::new()
    .route("/send", post(send_handler))
    .layer(OtelAxumLayer::default());
```

Crates bổ sung: `axum-tracing-opentelemetry = "0.18"`, `tower-http` (trace layer dự phòng).

### Structured log với trace correlation

```rust
#[tracing::instrument(skip(payload), fields(mail.to = %payload.to))]
async fn send_handler(payload: SendMailRequest) -> Result<()> {
    tracing::info!(template_id = payload.template_id, "sending mail");
    // trace_id tự động gắn vào log line → SigNoz tự correlate
}
```

### Deployment SigNoz trên K8s

```yaml
# helm install signoz signoz/signoz -n monitoring -f values.yaml
# Minimal values cho môi trường test:
clickhouse:
  resources:
    requests: { cpu: 500m, memory: 2Gi }
otelCollector:
  enabled: true
  config:
    receivers:
      otlp:
        protocols:
          grpc: { endpoint: "0.0.0.0:4317" }
          http: { endpoint: "0.0.0.0:4318" }
    exporters:
      clickhouse:
        endpoint: tcp://clickhouse:9000
```

Helm chart chính thức: `helm repo add signoz https://charts.signoz.io`

### Alerts mặc định cần cấu hình

| Alert | Condition | Severity |
|---|---|---|
| High error rate | `error_count / total > 1%` (5 phút) | Critical |
| Slow endpoint | P99 latency > 2s | Warning |
| Service down | Không có span trong 3 phút | Critical |
| RabbitMQ consumer lag | `consumer.lag > 1000` | Warning |

### Biến môi trường

```env
OTEL_EXPORTER_OTLP_ENDPOINT=http://otel-collector:4317
OTEL_SERVICE_NAME=mailsender          # override tên service
OTEL_RESOURCE_ATTRIBUTES=env=production,team=backend
RUST_LOG=info,cls_observability=debug
```

---

## Chiến lược Strangler-fig

1. **Không big-bang rewrite** — service Rust chạy song song .NET, cutover từng endpoint qua Gateway.
2. **Giữ nguyên contract** — JSON DTO (PascalCase), JWT claims, RabbitMQ event payload + routing key, database schema.
3. **Frontend không sửa** trong toàn bộ quá trình migrate.
4. **Parity test bắt buộc** trước mỗi cutover (cùng input → so sánh output .NET vs Rust).
5. **Feature flag / canary** ở Gateway: 1% → 10% → 50% → 100% traffic.

```
Frontend Vue (không sửa)
        ↓
  Ocelot Gateway ──── flag ────┬── .NET Service (legacy endpoints)
                               └── Rust Service (migrated endpoints)
                                        ↓
                              SQL Server / Mongo (SHARED)
                              RabbitMQ (SHARED)
```

---

## Lộ trình 8 pha (~18-24 tháng)

| Pha | Nội dung | Thời gian | Rủi ro |
|---|---|---|---|
| **0 — Chuẩn bị** | Dựng Rust workspace, CI/CD, coding standard, training team | 1 tháng | Thấp |
| **1 — PoC MailSender** | Validate pattern Rust (lapin + Apalis + lettre) | 1-2 tháng | Thấp |
| **2 — Stateless IO** | NotificationService (FCM) + LogService (Mongo) | 2-3 tháng | Thấp |
| **3 — Realtime + File** | ServerFiles + SignalR Hub ⚠️ | 3 tháng | Cao |
| **4 — Outer ring** | SharedServices + SystemService + CommunicationService + ReportService | 3-4 tháng | Thấp-Trung |
| **5 — Core nghiệp vụ** | TrainingRoute + Question + Course + UserService ⚠️ | 4-6 tháng | Rất cao |
| **6 — Replace IdentityServer** | Migrate sang Keycloak (multi-tenant realm) | 2 tháng | Trung |
| **7 — Replace Gateway** | Ocelot → Pingora/Envoy | 1-2 tháng | Trung |
| **8 — Cleanup** | Retire .NET images, Consul, Eureka; tối ưu K8s resource | 1 tháng | Thấp |

**Effort ước tính**: ~1920 mandays. Team 4 dev → ~24 tháng. Team 6 dev → ~16 tháng.

### Decision gates

| Gate | Thời điểm | Tiêu chí GO |
|---|---|---|
| G1 | Sau Pha 1 | Rust MailSender benchmark ≥ 2× .NET (RAM hoặc throughput) |
| G2 | Sau Pha 3 | SignalR cutover thành công hoặc phương án C (giữ .NET SignalR) chấp nhận được |
| G3 | Sau Pha 5 | Core 4 service ổn định ≥ 1 tháng, error rate ≤ baseline |

---

## Cấu trúc workspace Rust

```
cls-rust/
  crates/
    cls-core/           ← EventBus, JWT middleware, error types, observability
    cls-db/             ← SQLx pool + Tiberius wrapper
    cls-eventbus/       ← IEventBus trait + RabbitMQ impl (lapin)
    cls-auth/           ← JWT validation, claims parsing
    cls-observability/  ← tracing config + OpenTelemetry OTLP → SigNoz
  services/
    mailsender/         ← Pha 1
    notificationservice/← Pha 2
    logservice/         ← Pha 2
    serverfiles/        ← Pha 3
    signalr/            ← Pha 3
    ...
  docs/                 ← Bộ tài liệu hệ thống (01..07)
```

**Coding conventions:**
- Error handling: `thiserror` cho lib, `anyhow` cho binary
- Async runtime: Tokio only
- Logging: `tracing` macros
- Naming: `snake_case` trong Rust, map sang JSON `PascalCase` qua `serde rename`

---

## Tài liệu chi tiết

| File | Nội dung |
|---|---|
| [docs/01-system-overview.md](docs/01-system-overview.md) | Kiến trúc tổng quan, C4 diagram, luồng dữ liệu mẫu |
| [docs/02-backend-feature-catalog.md](docs/02-backend-feature-catalog.md) | Catalog tính năng backend theo từng service |
| [docs/services/README.md](docs/services/README.md) | Deep-dive per-service (16 file use case, validator, business rule) |
| [docs/03-frontend-feature-catalog.md](docs/03-frontend-feature-catalog.md) | Catalog tính năng WebUI + WebUI-Admin |
| [docs/04-database-schema.md](docs/04-database-schema.md) | Schema SQL Server + MongoDB chi tiết |
| [docs/05-deployment-operations.md](docs/05-deployment-operations.md) | K8s topology, CI/CD, runbook |
| [docs/06-glossary.md](docs/06-glossary.md) | Thuật ngữ nghiệp vụ + kỹ thuật |
| [docs/07-rust-migration-plan.md](docs/07-rust-migration-plan.md) | Kế hoạch migrate chi tiết, rủi ro, effort, decision gates |
