# Bằng chứng khảo sát và nguồn tham khảo

## 1. Mốc và cách đọc kết quả

- **Ngày khảo sát:** 28/10/2026.
- **Repository:** `chung6a8m/dotnet-starter-kit`.
- **Nhánh đọc:** `main`.
- **Commit:** [a967537526f3b769c7d268da3ed2a6920af65ce9](https://github.com/chung6a8m/dotnet-starter-kit/commit/a967537526f3b769c7d268da3ed2a6920af65ce9).
- **Commit source gần nhất tại mốc này:** 09/10/2026; nội dung `chore(identity): clarify the profile ETag's scope after #1425 (#1433)`.
- **Phương pháp:** đọc source, cấu hình, quy tắc repo, test và CI; truy vết một số đường gọi; đối chiếu tài liệu chính thức.
- **Không thực hiện:** build/chạy ứng dụng, chạy test sản phẩm, security scan toàn bộ dependency, benchmark, restore drill hoặc xác nhận tuân thủ pháp lý.

Mọi link source trong bộ tài liệu cố định commit để người đọc có thể kiểm lại ngay cả khi `main` thay đổi. Tên phiên bản package được lấy từ file thật, không suy từ README của upstream hoặc các phiên bản starter cũ.

Bộ tài liệu phân biệt ba loại phát biểu:

1. **Quan sát source:** code/config/test có nội dung được dẫn.
2. **Suy luận kỹ thuật:** tác động có thể xảy ra khi đi qua chuỗi code; cần runtime regression test trước khi định lượng ảnh hưởng.
3. **Đề xuất:** giải pháp, mục tiêu hoặc backlog mới; không mô tả như tính năng đã có.

Không dùng số lượng test quảng bá trong README để khẳng định mọi đường Database-per-Tenant đã được chứng minh.

## 2. Bản đồ source trọng yếu

### R01 — Cấu trúc, phiên bản và quy tắc

- [AGENTS.md](https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/AGENTS.md): modular monolith, Contracts, bốn điểm đăng ký module, quy tắc BuildingBlocks.
- [global.json](https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/global.json): SDK `10.0.100`, roll-forward `latestFeature`.
- [Directory.Build.props](https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/src/Directory.Build.props): `net10.0` và cấu hình build.
- [Directory.Packages.props](https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/src/Directory.Packages.props): EF Core `10.0.12`, Finbuckle `10.1.0`, Npgsql EF `10.0.3`, Hangfire `1.8.25` và các dependency khác tại mốc khảo sát.
- [LICENSE](https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/LICENSE): license của repo; không thay cho license dependency.

### R02 — Tenant routing và ranh giới database

- [AppTenantInfo](https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/src/BuildingBlocks/Shared/Multitenancy/AppTenantInfo.cs): connection, activation, validity, plan/quota.
- [MultitenancyModule](https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/src/Modules/Multitenancy/Modules.Multitenancy/MultitenancyModule.cs): claim/header/query, cache/store, root override, lifecycle guard.
- [BaseDbContext](https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/src/BuildingBlocks/Persistence/Context/BaseDbContext.cs): tenant connection override và default-on isolation.
- [TenantDbContext](https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/src/Modules/Multitenancy/Modules.Multitenancy/Data/TenantDbContext.cs): catalog chính.
- [BillingDbContext](https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/src/Modules/Billing/Modules.Billing/Data/BillingDbContext.cs): central billing có tenant ownership tường minh.
- [IdentityDbContext](https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/src/Modules/Identity/Modules.Identity/Data/IdentityDbContext.cs): identity theo tenant, database override riêng.

### R03 — Provisioning và migration

- [CreateTenantCommandHandler](https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/src/Modules/Multitenancy/Modules.Multitenancy/Features/v1/CreateTenant/CreateTenantCommandHandler.cs).
- [TenantProvisioningJob](https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/src/Modules/Multitenancy/Modules.Multitenancy/Provisioning/TenantProvisioningJob.cs).
- [TenantProvisioningService](https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/src/Modules/Multitenancy/Modules.Multitenancy/Provisioning/TenantProvisioningService.cs).
- [TenantInitialPasswordBuffer](https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/src/Modules/Multitenancy/Modules.Multitenancy/Services/TenantInitialPasswordBuffer.cs).
- [IdentityDbInitializer](https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/src/Modules/Identity/Modules.Identity/Data/IdentityDbInitializer.cs).
- [TenantService](https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/src/Modules/Multitenancy/Modules.Multitenancy/Services/TenantService.cs).
- [DbMigrator Program](https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/src/Host/FSH.Starter.DbMigrator/Program.cs).
- [TenantMigrationsHealthCheck](https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/src/Modules/Multitenancy/Modules.Multitenancy/TenantMigrationsHealthCheck.cs).
- [GetTenantMigrationsQueryHandler](https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/src/Modules/Multitenancy/Modules.Multitenancy/Features/v1/GetTenantMigrations/GetTenantMigrationsQueryHandler.cs).

### R04 — Background routing, transaction và cache

- [FinbuckleEventTenantScope](https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/src/Modules/Multitenancy/Modules.Multitenancy/Services/FinbuckleEventTenantScope.cs): tenant context chỉ có ID/Identifier.
- [FinbuckleEventingDrainScope](https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/src/Modules/Multitenancy/Modules.Multitenancy/Services/FinbuckleEventingDrainScope.cs): drain context có connection.
- [TenantStoreDrainTargetProvider](https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/src/Modules/Multitenancy/Modules.Multitenancy/Services/TenantStoreDrainTargetProvider.cs): default DB và dedicated active targets.
- [InMemoryEventBus](https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/src/BuildingBlocks/Eventing/InMemory/InMemoryEventBus.cs): scope → handler → inbox.
- [EfCoreOutboxStore](https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/src/BuildingBlocks/Eventing/Outbox/EfCoreOutboxStore.cs): enlist transaction và PostgreSQL claiming.
- [ScopedDbConnectionProvider](https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/src/BuildingBlocks/Persistence/ScopedDbConnectionProvider.cs).
- [UserPermissionService](https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/src/Modules/Identity/Modules.Identity/Services/UserPermissionService.cs): load quyền theo home tenant của caller khi root override.
- [CacheKeys](https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/src/BuildingBlocks/Caching/CacheKeys.cs): permission key theo user ID, theme/idempotency có tenant.
- [SqlAuditSink](https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/src/Modules/Auditing/Modules.Auditing/Persistence/SqlAuditSink.cs): full-tenant lookup trước resolve audit context.

### R05 — Bằng chứng test, không phải kết quả test đã chạy

| Test source | Chứng minh chủ đích/fixture nào | Không được suy rộng thành |
|---|---|---|
| [PurgeJobsPerTenantTests](https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/src/Tests/Integration.Tests/Tests/Files/PurgeJobsPerTenantTests.cs) | Tạo database riêng thật, provisioning/login và file jobs | Mọi module/consumer đều đúng ở dedicated DB |
| [TenantHeaderOverrideTests](https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/src/Tests/Integration.Tests/Tests/Multitenancy/TenantHeaderOverrideTests.cs) | Claim/header và root override, gồm cache lạnh | Mọi root endpoint đúng attribution trên hai DB riêng |
| [TenantProvisioningFailureTests](https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/src/Tests/Integration.Tests/Tests/Multitenancy/TenantProvisioningFailureTests.cs) | Failure ở migrations; guard reactivation | Tenant mới mặc định inactive đến Ready |
| [TenantMigrationsHealthCheckTests](https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/src/Tests/Integration.Tests/Tests/Health/TenantMigrationsHealthCheckTests.cs) | Fixture TenantDbContext có pending/lỗi | Health check đã dò mọi module/dedicated DB |
| [OutboxAtomicityTests](https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/src/Tests/Integration.Tests/Tests/Eventing/OutboxAtomicityTests.cs) | Shared connection và commit/rollback trong transaction root | Mọi use case đều có transaction hoặc consumer exactly-once |

### R06 — Vận hành, sản phẩm và frontend

- [Backend CI](https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/.github/workflows/backend.yml), [Frontend CI](https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/.github/workflows/frontend.yml).
- [Compose](https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/deploy/docker/docker-compose.yml), [Deployment README](https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/deploy/docker/README.md).
- [Admin package](https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/clients/admin/package.json), [Dashboard package](https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/clients/dashboard/package.json).
- [SupportedCultures](https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/src/BuildingBlocks/Core/Localization/SupportedCultures.cs): `en-US` và `pt-BR`, chưa có `vi-VN`.
- [UsageReporter](https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/src/Modules/Billing/Modules.Billing/Services/UsageReporter.cs), [MonthlyInvoiceJob](https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/src/Modules/Billing/Modules.Billing/Services/MonthlyInvoiceJob.cs), [RedisQuotaService](https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/src/BuildingBlocks/Quota/RedisQuotaService.cs): đối chiếu logic kỳ usage.
- [InvoicePdfRenderer](https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/src/Modules/Billing/Modules.Billing/Services/InvoicePdfRenderer.cs): QuestPDF Community setting.

## 3. Nguồn bên ngoài

<a id="nguon-ben-ngoai"></a>

Các tài liệu dưới đây là nguồn chính thức được kiểm tra ngày 28/10/2026. Link không pin nội dung theo ngày; khi triển khai cần đối chiếu lại phiên bản đang sử dụng.

| Mã | Nguồn | Mục đích sử dụng |
|---|---|---|
| E01 | [Microsoft EF Core: Multi-tenancy](https://learn.microsoft.com/en-us/ef/core/miscellaneous/multitenancy) | Phân biệt connection theo tenant và query filter, lifetime cấu hình/context |
| E02 | [Microsoft: Multitenant storage and data](https://learn.microsoft.com/en-us/azure/architecture/guide/multitenant/approaches/storage-data) | Trade-off isolation, chi phí và tự động hóa nhiều database |
| E03 | [Microsoft: Architectural approaches overview](https://learn.microsoft.com/en-us/azure/architecture/guide/multitenant/approaches/overview) | Cụm triển khai phục vụ một hoặc nhiều tenant |
| E04 | [Microsoft EF Core: Transactions](https://learn.microsoft.com/en-us/ef/core/saving/transactions) | Transaction mặc định và điều kiện chia sẻ transaction giữa contexts |
| E05 | [Finbuckle: EF Core data isolation](https://www.finbuckle.com/MultiTenant/Docs/v10.1.5/EFCore) | Đối chiếu cơ chế riêng database và shared-database filtering |
| E06 | [Npgsql: Connection string parameters](https://www.npgsql.org/doc/connection-string-parameters) | Pooling, minimum/maximum và vòng đời connection |
| E07 | [PostgreSQL 18: Continuous archiving/PITR](https://www.postgresql.org/docs/18/continuous-archiving.html) | Phân biệt recovery cả cluster và khôi phục tenant bằng môi trường tạm |
| E08 | [PostgreSQL 18: File system backup](https://www.postgresql.org/docs/18/backup-file.html) | Điều kiện nhất quán khi backup files/volume |
| E09 | [.NET support policy](https://dotnet.microsoft.com/en-us/platform/support/policy) | .NET 10 LTS, lịch hỗ trợ và cập nhật bản vá |
| E10 | [SDK, MSBuild và Visual Studio matrix](https://learn.microsoft.com/en-us/dotnet/core/porting/versioning-sdk-msbuild-vs) | Điều kiện toolchain cho `net10.0` |
| E11 | [QuestPDF: License Selection Guide](https://www.questpdf.com/license/guide.html) | Kiểm điều kiện Community/paid license cho dependency PDF |

**Lưu ý phiên bản Finbuckle:** repo pin `10.1.0`. URL documentation phiên bản 10.1.0 đã chuyển hướng sang 10.1.5 lúc tra cứu. Bộ tài liệu dùng source repo để kết luận hành vi cụ thể; không lấy API/hành vi phiên bản docs mới hơn thay cho code đã pin.

**Toolchain Windows:** repo target `net10.0`. Ma trận Microsoft gắn SDK `10.0.1xx` với Visual Studio `18.0`; target runtime mới trên Visual Studio cũ không được hỗ trợ đầy đủ. Nếu đội còn dùng Visual Studio 2022, cần nâng IDE phù hợp hoặc dùng CLI/VS Code với SDK được hỗ trợ; không tự hạ target framework của repo như một thay đổi cấu hình nhỏ. Theo policy tại ngày tra cứu, .NET 10 LTS được hỗ trợ đến 14/11/2028.

## 4. Cách cập nhật tài liệu khi code thay đổi

1. Ghi commit mới và phạm vi diff so với mốc khảo sát.
2. Với mỗi K01–K06 và O01, chỉ đánh dấu đã xử lý khi có thay đổi code liên quan và bằng chứng kiểm chứng đúng dedicated-database scenario.
3. Chuyển một đề xuất thành quyết định bằng ADR/nghiệm thu rõ; giữ lịch sử lý do và trade-off.
4. Tách trạng thái “đã có test”, “test đã chạy pass” và “đã kiểm trên môi trường pilot”.
5. Điều chỉnh roadmap theo phát hiện thực nghiệm; không giữ nguyên ước lượng nếu identity, engine hoặc phạm vi nghiệp vụ thay đổi.

Các kiểm tra tài liệu cần thực hiện trước commit: phạm vi diff chỉ gồm `docs/sme/ideas`, encoding UTF-8/LF, Markdown không có lỗi whitespace, link nội bộ và path source hợp lệ, nguồn dẫn cố định đúng commit. Việc kiểm tra này không thay thế build/test sản phẩm.
