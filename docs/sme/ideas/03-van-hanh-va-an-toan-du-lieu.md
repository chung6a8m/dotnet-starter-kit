# Vận hành và an toàn dữ liệu cho Database-per-Tenant

> Đây là yêu cầu vận hành và kế hoạch kiểm chứng đề xuất ngày 28/10/2026, không phải runbook đã chạy thành công. Hiện trạng source được khóa tại [a9675375](https://github.com/chung6a8m/dotnet-starter-kit/tree/a967537526f3b769c7d268da3ed2a6920af65ce9); xem [đánh giá repo](01-danh-gia-repo.md) và [kiến trúc đích](02-kien-truc-de-xuat.md).

## 1. Điều gì phải chứng minh được?

Một tenant chỉ được coi là đủ điều kiện pilot khi:

- Tạo, truy cập, xử lý nền và xuất dữ liệu đều vào đúng database.
- Request/job của tenant A không đọc/ghi dữ liệu B hoặc rơi về database mặc định.
- Có thể nâng cấp schema A, phát hiện lỗi và tiếp tục phục vụ B.
- Có thể restore A vào database mới và đối soát tệp, identity, sự kiện mà không ghi đè B.
- Runtime không có quyền cấp database hoặc sửa schema.
- Nhóm vận hành biết ai xử lý khi provisioning/migration/backup/event thất bại.

Đề xuất ban đầu để thảo luận là **RPO không quá 15 phút, RTO không quá 4 giờ cho một tenant pilot**. Đây là mục tiêu kỹ thuật cần thử nghiệm trên dữ liệu gần thực tế và cả backup control plane, không phải SLA đã đạt. Logical dump hàng đêm đơn lẻ không thể đáp ứng RPO 15 phút. Nếu chưa có ngân sách/cơ chế đáp ứng, phải đổi mục tiêu và thông báo điều kiện dịch vụ trước pilot.

## 2. Provisioning: có trạng thái bền và có thể phục hồi

Repo có provisioning attempts/steps và Hangfire job, nhưng step Database chưa cấp tài nguyên riêng, CacheWarm chưa thực hiện warm; tenant được tạo active sớm và mật khẩu admin đi qua buffer trong process. Các điểm này đã được dẫn chứng ở K03 của [tài liệu 01](01-danh-gia-repo.md).

### 2.1. Quy trình đích

| Giai đoạn | Việc thực hiện | Điều kiện chuyển bước |
|---|---|---|
| Đăng ký | Ghi tenant chưa Ready, placement dự kiến, operation ID duy nhất | Không có active attempt khác; idempotency key được lưu bền |
| Cấp phát | Tạo database/roles/grants bằng provisioner | Database identity đúng, runtime role chỉ có quyền dự kiến |
| Migration | Chạy version bộ module đã chọn | Mọi module cần thiết ở schema version tương thích |
| Seed | Dữ liệu hệ thống, role, invitation cho admin | Seed lặp an toàn; không copy tài khoản/mật khẩu demo |
| Kiểm chứng | Kết nối bằng runtime credential, kiểm quyền và read/write tối thiểu | Runtime hoạt động; không truy cập database tenant khác |
| Công bố | Ghi Ready, tăng routing version, làm mới cache, phát sự kiện hoàn tất | State update và event điều khiển được ghi bền cùng nơi |
| Thất bại | Ghi step, lỗi đã loại secret, lần thử và lịch retry | Không phục vụ data-plane; cho retry hoặc xử lý thủ công có audit |

Các action phải có hậu điều kiện kiểm được. Khi database đã được tạo nhưng worker chết trước khi ghi Completed, lần chạy lại phải nhận ra đúng tài nguyên của operation; không tạo database thứ hai hoặc xóa một database không thuộc operation đó.

### 2.2. Cạnh tranh và phục hồi

Cần unique constraint/optimistic concurrency hoặc lease theo tenant, cộng cơ chế fencing để worker mất lease không được tiếp tục ghi trạng thái. Ghi Pending rồi enqueue job là hai bước có thể thất bại giữa chừng; dùng durable enqueue/outbox hoặc reconciliation quét attempt bị treo.

Phân biệt:

- Tự retry cùng attempt: bỏ qua step đã hoàn tất sau khi kiểm hậu điều kiện.
- Retry thủ công: có liên kết attempt trước, quyền riêng và reason.
- Hủy provisioning: giữ lịch sử; dọn tài nguyên có ownership chắc chắn, không tự xóa dữ liệu đã phát sinh.
- Tenant bị suspend vì thanh toán: chưa phải tenant bị xóa; chính sách background maintenance/retention vẫn tồn tại.

Admin nên nhận invitation/set-password token dùng một lần. Nếu buộc dùng secret khởi tạo, lưu qua kho secret có TTL và reference bền; worker khác process vẫn lấy được theo quyền. Không đưa mật khẩu thô vào Hangfire payload/log, và không dùng mật khẩu seed chung làm fallback cho tenant khách hàng.

## 3. Phân tách credentials và giới hạn quyền

| Danh tính kỹ thuật | Quyền cần | Không nên có |
|---|---|---|
| API runtime tenant A | DML trên schema nghiệp vụ A, đọc metadata cần thiết | Tạo/drop database, thay schema, truy cập dữ liệu tenant B |
| Worker nghiệp vụ A | Tương tự API cho công việc được giao | Credentials migrator hoặc toàn quyền mọi database |
| Migrator | DDL trên database được chọn; đọc manifest và ledger | Thao tác tùy ý không xác định target |
| Provisioner | Tạo database/role/grant trong placement được phép | Chạy như identity của người dùng SME |
| Backup/restore operator | Quyền backup/restore có kiểm soát, truy cập kho backup | Xuất bản backup ra storage công khai |
| Control-plane runtime | Catalog, subscription, state vận hành theo quyền | Tự suy diễn quyền đọc mọi dữ liệu khách hàng |

MVP có thể dùng một workload identity truy cập các secret theo policy; điều đó vẫn phải được đánh giá về phạm vi ảnh hưởng. Một database/tenant nhưng dùng chung superuser cho mọi kết nối không đạt mục tiêu hạn chế ảnh hưởng khi credential bị lộ.

Hiện tenant connection string được giữ trong `AppTenantInfo`; job filter còn lưu tenant object vào job parameter. Đề xuất lưu descriptor/secret reference, trả alias và `HasDedicatedDatabase` cho UI, và chỉ gửi tenant ID trong job. Rotation phải tăng version, vô hiệu cache cũ và xử lý connection pool cũ; retry dài hạn phải tra registry mới nhất. [AppTenantInfo](https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/src/BuildingBlocks/Shared/Multitenancy/AppTenantInfo.cs), [FshJobFilter](https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/src/BuildingBlocks/Jobs/FshJobFilter.cs)

Không dùng việc mã hóa connection string trong catalog như giải pháp duy nhất: phải quản lý khóa, quyền đọc, bản backup và khả năng phục hồi kho secret.

## 4. Migration nhiều database

### 4.1. Tận dụng runner hiện có nhưng phải kiểm đúng mục tiêu

Repo đã có one-shot `DbMigrator`, catalog-first, advisory lock, `--tenant <id>` và exit code. Giữ cách tách migration khỏi API startup. Tuy nhiên `list-pending` hiện chỉ có báo cáo đầy đủ cho catalog; endpoint/health cũng resolve `TenantDbContext` thay vì toàn bộ module context của tenant. [DbMigrator Program](https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/src/Host/FSH.Starter.DbMigrator/Program.cs), [TenantMigrationsHealthCheck](https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/src/Modules/Multitenancy/Modules.Multitenancy/TenantMigrationsHealthCheck.cs)

Lệnh tham khảo hiện tại, phải chạy với cấu hình môi trường/target đã kiểm tra:

```bash
dotnet run --project src/Host/FSH.Starter.DbMigrator -- list-pending
dotnet run --project src/Host/FSH.Starter.DbMigrator -- apply --tenant TENANT_ID
```

Không coi lệnh đầu là bằng chứng mọi tenant module đã ở head. Với snapshot này dùng dạng `--tenant TENANT_ID`; parser có vấn đề với `--tenant=TENANT_ID` ở đối số cuối, có thể bỏ mất bộ lọc. Cần sửa parser và bắt buộc hiển thị/kiểm target manifest trước automation. [MigratorCommand](https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/src/Host/FSH.Starter.DbMigrator/MigratorCommand.cs)

### 4.2. Quy trình release đề xuất

1. Tạo manifest release: code version, danh sách module/migration, schema tương thích tối thiểu/tối đa, target databases và yêu cầu backup.
2. Kiểm preflight: target tồn tại, database identity đúng, credential DDL đúng, dung lượng/lock budget đủ.
3. Migrate catalog/control-plane trước theo dependency đã xác định.
4. Canary trên tenant nội bộ có database riêng và dữ liệu đại diện.
5. Chạy từng batch nhỏ, giới hạn concurrency theo DB server; ghi trạng thái trên từng database/module.
6. Khi target lỗi, lưu lỗi và giữ tenant đó ở chế độ thích hợp. Dừng batch hoặc tiếp tục các target độc lập theo policy rõ; không bỏ qua lỗi âm thầm.
7. Đối chiếu ledger với schema thật, kiểm query nghiệp vụ và outbox, rồi cho target nhận traffic phù hợp.
8. Chỉ kết thúc release khi mọi target dự kiến được phân loại Completed/Blocked/Deferred có lý do.

MVP ít tenant có thể chạy tuần tự như runner hiện tại. Cần ledger/retry theo target và báo cáo chính xác trước khi đầu tư parallel migration quy mô lớn.

### 4.3. Schema tương thích và rollback

Dùng chiến lược thêm cấu trúc trước, chuyển code/data, rồi mới xóa cấu trúc cũ trong release sau. Application phải hỗ trợ cửa sổ schema đã định trong quá trình rollout; không để API mới chạy query đòi cột chưa có ở tenant chưa migrate.

Rollback ứng dụng chỉ khả thi nếu schema còn tương thích. Với migration mất dữ liệu, ưu tiên forward fix hoặc restore theo runbook; không tự động chạy `Down` trên production như một bảo đảm khôi phục. Trước thao tác không đảo ngược phải có backup được kiểm và kế hoạch dừng ghi.

Không tạo migration fork riêng mỗi tenant. Custom fields nên đi qua schema/metadata được quản lý chung, có validation và index theo nhu cầu thực tế.

## 5. Backup, restore và chuyển tenant

### 5.1. Đặc điểm PostgreSQL cần hiểu đúng

PostgreSQL physical backup + WAL/PITR phục hồi cả cluster, không trực tiếp chỉ một database trong cluster. Muốn phục hồi riêng tenant tới một thời điểm, có thể phục hồi cluster vào môi trường tạm rồi trích xuất database tenant. Logical dump phục vụ phục hồi từng database nhưng không thay thế WAL/PITR. [Tài liệu PostgreSQL 18](https://www.postgresql.org/docs/18/continuous-archiving.html)

README Docker hiện minh họa `tar` volume; không có đủ quy trình bảo đảm nhất quán và restore drill. Không sử dụng việc copy volume đang ghi làm bằng chứng backup hợp lệ. Cần chọn phương pháp backup online đúng hoặc snapshot nhất quán theo cơ chế PostgreSQL/storage. [README của repo](https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/deploy/docker/README.md), [PostgreSQL: file system backup](https://www.postgresql.org/docs/18/backup-file.html)

### 5.2. Runbook restore tenant đề xuất

| Bước | Thao tác | Bằng chứng nghiệm thu |
|---|---|---|
| 1 | Xác định tenant, thời điểm, mục tiêu nghiệp vụ; khóa operation | Người có quyền, target và recovery point rõ |
| 2 | Đưa tenant vào maintenance; dừng/fence write và side-effect jobs | Không còn ghi mới vào database cũ sau cutover boundary |
| 3 | Restore vào database mới, trong môi trường cô lập | Backup đọc được, checksum/manifest hợp lệ, không overwrite tenant đang chạy |
| 4 | Kiểm schema, tenant ID, row counts, tổng tiền/sổ kho và constraints | Báo cáo chênh lệch được chấp nhận |
| 5 | Đối soát object versions, cache, sessions, outbox/inbox và hệ thống ngoài | Không phát lại payment/email/webhook lịch sử ngoài ý định |
| 6 | Đổi routing bằng compare-and-swap, tăng version/recovery epoch | API/worker nhận mapping mới; mapping cũ bị vô hiệu |
| 7 | Smoke test, mở dần traffic, theo dõi lỗi | Tenant A hoạt động; B không đổi; RPO/RTO đo được |
| 8 | Giữ database cũ trong thời hạn điều tra rồi dọn theo policy | Có biên bản và cơ chế quay lại khi còn khả thi |

Phải lưu receipts/idempotency và đối soát trạng thái bên ngoài: restore database về quá khứ có thể làm inbox “quên” một side effect đã xảy ra. Kho backup đơn thuần không giải quyết được điều đó.

Receipts của side effect đã phát ra ngoài phải nằm **ngoài phạm vi restore tenant**: ví dụ ledger điều phối bền ở control plane hoặc receipt/idempotency của nhà cung cấp ngoài, có tenant ID và operation ID ổn định. Không restore ngược kho receipts cùng tenant DB rồi kỳ vọng chống lặp vẫn đúng. Trước khi mở worker, đối chiếu các operation đã hoàn tất bên ngoài để quyết định giữ kết quả, bổ sung trạng thái hoặc thực hiện compensation; không tự phát lại toàn bộ outbox cũ.

Identity tenant-local phục hồi cùng DB có thể phục hồi session/refresh token hoặc password state cũ. Cần recovery epoch/thu hồi phiên thích hợp và audit thao tác phục hồi. Không tự reset tài khoản khách hàng bằng dữ liệu demo.

Recovery/revocation epoch phải tăng đơn điệu và lưu ngoài database tenant đang restore, chẳng hạn trong registry control plane. API và worker kiểm epoch/version trước chấp nhận token/job cũ; lưu epoch trong chính snapshot tenant sẽ không giải quyết được việc “hồi sinh” phiên. Khi phục hồi thảm họa cả control plane, cần quy trình thu hồi/đối soát riêng với nguồn receipts bên ngoài.

Object storage cần version/manifest để đối chiếu với metadata DB; chỉ backup database không đủ cho attachment. Control catalog, billing và secret references cần được backup riêng với quy trình phục hồi mối liên hệ tới tenant databases.

### 5.3. Chuyển placement và offboarding

Chuyển tenant sang server/cụm khác có quy trình tương tự restore: chuẩn bị target, đồng bộ hoặc dừng ghi, fence worker cũ, cutover registry, tăng version, kiểm truy cập và giữ đường quay lại có thời hạn. DNS/connection cache không được để API và worker cùng ghi hai database.

Offboarding gồm ngừng quyền truy cập, hoàn tất export được phép, đối soát billing, thu hồi secrets, dọn jobs/index/object storage theo chính sách lưu giữ đã chốt. Database chỉ được xóa sau khi phạm vi, người phê duyệt và thời hạn giữ đã rõ. Đây là quy trình sản phẩm cần thiết kế, chưa phải tính năng có sẵn được xác nhận.

## 6. Capacity, connection pool và tài nguyên dùng chung

Npgsql pooling được bật mặc định; cấu hình hiện hành có maximum pool size mặc định 100 và minimum 0. Đây là giới hạn pool, không có nghĩa 100 kết nối mở ngay. [Npgsql parameters](https://www.npgsql.org/doc/connection-string-parameters)

Một ước lượng trần connection cần xét theo **process và pool/database đang dùng**, không theo số DbContext:

```text
C_api_upper ≈ số_API_process × số_tenant_đồng_thời_nóng_mỗi_process × MaxPoolSize_mỗi_tenant
C_total_upper ≈ C_api_upper + pools_worker + pools_control + migrator + dự_phòng_vận_hành
```

Ví dụ minh họa: 3 API process, 30 tenant “nóng” mỗi process, max pool 5 tạo trần lý thuyết 450 connection chỉ cho phần tenant API; chưa tính worker/control/migrator. Đây không phải tải đã đo hoặc sizing khuyến nghị mặc định.

Repo có `ScopedDbConnectionProvider` để các context trong cùng scope chia sẻ connection; nó không giới hạn connection toàn bộ fleet. Default appsettings còn có `Minimum Pool Size=5`; không nên copy giá trị này cho hàng trăm tenant mà chưa đo. [ScopedDbConnectionProvider](https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/src/BuildingBlocks/Persistence/ScopedDbConnectionProvider.cs), [appsettings](https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/src/Host/FSH.Starter.Api/appsettings.json)

Các biện pháp cần đánh giá qua load test:

- Giới hạn concurrent requests và background work theo tenant; tránh tenant import lớn chiếm mọi worker.
- Pool minimum thấp, maximum phù hợp workload; dispose connection/context đúng; không giữ transaction khi gọi mạng.
- Index, pagination, giới hạn export và thời gian chạy báo cáo.
- Dùng PgBouncer khi có bài toán connection thực tế, sau test tương thích transaction/session; nó không xóa giới hạn CPU/IO hay số database.
- Chuyển tenant lớn sang placement khác trước khi gây ảnh hưởng nhóm tenant nhỏ.
- Một DI scope chia sẻ một connection không được chạy EF operations song song tùy ý; công việc độc lập dùng scopes riêng.

## 7. Giám sát và xử lý lỗi

### 7.1. Phân biệt các loại sức khỏe

- **Liveness:** process còn hoạt động.
- **Readiness của API/cụm:** cấu hình và các phụ thuộc chung thiết yếu sẵn sàng.
- **Tình trạng từng tenant:** routing, schema, quyền truy cập, database, backlog và hoạt động bảo trì.

Không thiết kế readiness của toàn cụm theo nguyên tắc “một tenant DB hỏng thì loại hết API”. Thay vào đó, cô lập tenant lỗi, trả lỗi dịch vụ có mã/correlation phù hợp, và hiển thị đầy đủ cho operator.

Endpoint công khai chỉ nên có trạng thái tổng quát; chi tiết database/tenant/errors dành cho kênh vận hành được xác thực. Snapshot hiện tại trả details qua health endpoints anonymous; cần rà mức thông tin hiển thị khi triển khai. [HealthEndpoints](https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/src/BuildingBlocks/Web/Health/HealthEndpoints.cs)

### 7.2. Các tín hiệu cần theo dõi

| Nhóm | Tín hiệu |
|---|---|
| Provisioning | Số Pending/Running quá hạn, retry, tuổi attempt, thời gian tới Ready |
| Database | Kết nối đang dùng/chờ, query latency, lock, CPU/IO, dung lượng tăng |
| Migration | Version theo database/module, số Blocked/Deferred, thời gian từng target |
| Event/jobs | Tuổi outbox lâu nhất, dead letters, retry, consumer lag, queue fairness |
| Recovery | Tuổi backup hợp lệ, WAL archive lag, lần restore thử gần nhất, RPO/RTO đo được |
| Bảo mật | Root/support access, thay quyền, failed auth, export lớn, secret rotation |
| Sản phẩm | Lỗi nghiệp vụ, conflict tồn kho, đối soát công nợ, thời gian xử lý chứng từ |

Tenant ID nên có trong log/traces với kiểm soát truy cập. Với metrics, tránh nhãn tenant vô hạn trên mọi time series; dùng tổng hợp theo placement/plan và drill-down có giới hạn.

Repo có OpenTelemetry nhưng phải kiểm dữ liệu thật tới collector. Chỉ cấu hình endpoint chưa bảo đảm exporter được bật: production settings hiện đặt OTLP `Enabled=false`; deployment cần cấu hình và kiểm traces/metrics thực tế. [Production settings](https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/src/Host/FSH.Starter.Api/appsettings.Production.json)

Outbox drain hiện chỉ thêm dedicated targets của tenant active. Cần quyết định riêng liệu tenant suspended/expired vẫn được xử lý các sự kiện thanh toán, retention hoặc hỗ trợ nào; tránh backlog bị bỏ quên nhưng cũng không tự tiếp tục mọi nghiệp vụ của tenant đã khóa. [TenantStoreDrainTargetProvider](https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/src/Modules/Multitenancy/Modules.Multitenancy/Services/TenantStoreDrainTargetProvider.cs)

## 8. Billing SaaS, quota và chi phí

### 8.1. Phát hiện O01: snapshot usage theo kỳ cần sửa trước tính tiền

Đây là suy luận từ call chain, chưa tái hiện runtime:

- `MonthlyInvoiceJob` tính tháng trước rồi yêu cầu sinh invoice.
- `UsageReporter.CaptureForPeriodAsync` nhận năm/tháng nhưng đọc `GetCurrentAsync`.
- `RedisQuotaService` tạo key `ApiCalls` theo tháng hiện tại và reset ở đầu tháng kế tiếp.
- Job định kỳ chạy lúc 00:05 UTC ngày 1.

Kết quả có thể là lấy bộ đếm tháng mới rồi ghi nhãn tháng cũ; bộ đếm cũ cũng không còn thời hạn lưu cho đối soát sau rollover. Không dùng đường này làm cơ sở usage billing trước khi có period-aware metering, dữ liệu lịch sử bền và test retry/backfill. [MonthlyInvoiceJob](https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/src/Modules/Billing/Modules.Billing/Services/MonthlyInvoiceJob.cs), [UsageReporter](https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/src/Modules/Billing/Modules.Billing/Services/UsageReporter.cs), [RedisQuotaService](https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/src/BuildingBlocks/Quota/RedisQuotaService.cs), [BillingModule](https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/src/Modules/Billing/Modules.Billing/BillingModule.cs)

**MVP đề xuất:** gói thuê bao cố định theo tier; quota phục vụ kiểm soát tài nguyên. Kể cả giá cố định vẫn phải test vòng đời subscription, term invoice, retry và idempotency.

Quota/ratelimit có code nhưng mặc định không đồng nghĩa đã bật. Cần kiểm `QuotaOptions.Enabled`, `QuotaOptions.Redis` và các lựa chọn fail-open/fail-closed cho từng loại hạn mức. Bộ đếm in-process không đáp ứng quota toàn cụm khi nhiều replica. [Quota registration](https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/src/BuildingBlocks/Quota/Extensions.cs)

### 8.2. Mô hình TCO để quyết định gói dịch vụ

```text
Chi_phí_tháng =
  compute_API_worker + database_compute_storage
  + backup_WAL_retention + object_storage_egress
  + cache_queue + observability
  + license_support + giờ_vận_hành

Chi_phí_bình_quân_tenant = tổng_chi_phí / số_tenant_trả_phí
```

Bình quân chỉ dùng cho hoạch định. Cần phân bổ thêm chi phí tenant lớn, tenant có nhiều tệp, báo cáo hoặc backup retention dài. Định giá tối thiểu phải bao phủ onboarding, hỗ trợ và restore, ngoài hosting.

Không có báo giá hạ tầng hay benchmark trong đợt này. Database-per-Tenant trên cùng PostgreSQL instance không đồng nghĩa mua một máy chủ riêng mỗi tenant; tuy nhiên mọi database vẫn tiêu thụ tài nguyên, migration và công vận hành.

Repo có MIT license nhưng dependency có điều kiện riêng. `InvoicePdfRenderer` đặt QuestPDF Community license; phải kiểm điều kiện áp dụng cho pháp nhân sản xuất và phương án phân phối trước thương mại hóa. Không suy rằng MIT của repo bao trùm mọi dependency. [Renderer](https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/src/Modules/Billing/Modules.Billing/Services/InvoicePdfRenderer.cs), [QuestPDF License Selection Guide](https://www.questpdf.com/license/guide.html)

## 9. Ma trận kiểm chứng ưu tiên

Fixture tối thiểu: một control/default DB, tenant A dùng database A, tenant B dùng database B, các credentials ít quyền riêng và ít nhất hai API/worker process cho các test yêu cầu. Có thể giữ thêm shared tenant để bảo đảm thay đổi không vô tình phá compatibility có chủ đích.

| Mã | Tình huống | Điều kiện đạt |
|---|---|---|
| T01 | A đọc/ghi/list/export với ID thuộc B; giả mạo header/query | B/default không bị ghi dữ liệu nghiệp vụ tenant trái phép; không trả nội dung B; tenant claim thắng; control audit hợp lệ vẫn được ghi |
| T02 | Tenant descriptor thiếu/sai, catalog cache cũ | Data-plane từ chối; không fallback về default |
| T03 | Job thực thi ở worker khác, sau rotate/move | Lookup descriptor mới; chỉ đụng target đã xác minh |
| T04 | A upload/finalize → outbox → webhook fanout → inbox | Subscription chỉ A vẫn sinh đúng delivery; B/default không có row event tương ứng |
| T05 | Rollback business transaction + outbox; crash trước/sau inbox | Cùng commit/rollback khi cần; retry không tạo chứng từ hoặc side effect trùng |
| T06 | Catalog ở head, DB A offline hoặc thiếu migration một module | Báo đúng tenant/module; B vẫn phục vụ; readiness chung không bị đánh đồng |
| T07 | Provisioning kill/restart ở từng step, hai attempt đồng thời | Một workflow có hiệu lực; không mất invitation; Pending được reconciliation |
| T08 | Runtime credential thử DDL và truy cập DB B | Bị từ chối; migrator đúng quyền vẫn thành công |
| T09 | Root support/impersonation, cache lạnh, grant bị thu hồi | Đúng database/user, actor/effective tenant đầy đủ; revoke có hiệu lực |
| T10 | Hai DB có cùng UserId do import/clone, quyền khác nhau | Permission cache/realtime/session không lẫn tenant |
| T11 | Restore A tới thời điểm trước một webhook/payment đã gửi | Không phát lại ngoài ý định; receipts và số dư được đối soát; B không đổi |
| T12 | Migration batch có một target lỗi, retry, rollback app | Target manifest/ledger chính xác, không migrate nhầm tất cả tenant |
| T13 | Hết tháng, job chạy muộn/retry, usage backfill | Metering đọc đúng kỳ, invoice không trùng, có dữ liệu đối soát |
| T14 | Một tenant import/report lớn và các tenant tải thường | Đáp ứng budget latency đã chốt; pool/worker không bị chiếm hết |
| T15 | Suspend, expired, offboarding trong lúc có job/event chờ | Policy nhất quán cho request, jobs, drain, dữ liệu giữ lại và export |
| T16 | Luồng bán hàng–xuất kho–thu tiền cạnh tranh/retry | Không âm kho ngoài policy, không nhân đôi phiếu, công nợ đối soát được |
| T17 | Tạo/gia hạn tenant → Billing trung tâm → event tiếp theo, có crash/retry | Catalog/Billing/outbox/inbox ở đúng control DB theo producer/consumer; local transaction đúng; không bị chuyển sang tenant DB khi sửa K01; invoice/subscription không trùng |

T04 đặc biệt quan trọng vì code hiện đã có drain theo dedicated database nhưng consumer có thể làm mất connection descriptor. Test chỉ kiểm outbox “Processed” sẽ bỏ sót mất side effect. Phải assert vị trí dữ liệu và kết quả nghiệp vụ cuối.

T17 bảo vệ trường hợp ngược lại: event có tenant ID nhưng producer/consumer sở hữu dữ liệu control-plane. Sửa tenant routing phải giữ ranh giới này, kể cả MVP chỉ dùng giá thuê bao cố định.

Các test hiện có như `PurgeJobsPerTenantTests` là nền tốt; chưa có kết quả chạy các test T01–T17 trong đợt tài liệu này. [PurgeJobsPerTenantTests](https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/src/Tests/Integration.Tests/Tests/Files/PurgeJobsPerTenantTests.cs)

## 10. Deployment và các cổng nghiệm thu

Repo có Compose, Terraform, backend CI, frontend CI và migrator smoke. Cần dùng chúng như điểm xuất phát, không như chứng nhận môi trường production. Compose hiện dùng PostgreSQL 18, trong khi một service smoke CI dùng PostgreSQL 17; phải thêm kiểm chứng đúng version/extension/cấu hình mục tiêu. [Compose](https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/deploy/docker/docker-compose.yml), [Backend CI](https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/.github/workflows/backend.yml)

| Cổng | Điều kiện |
|---|---|
| G0 — Hiểu nền tảng | Tái hiện K01/K02/K03; chốt engine, identity, tenant boundary và phạm vi MVP |
| G1 — Database-per-Tenant | T01–T10, T12, T15 và T17 đạt theo scope; secrets/privileges/migration/routing được review |
| G2 — Nghiệp vụ MVP | Luồng SME đầu-cuối đúng, data scopes, import/export và T16 đạt |
| G3 — Pilot dữ liệu thật | T11/T14 đạt, restore drill có biên bản, observability và người vận hành sẵn sàng |
| G4 — Usage billing nếu triển khai | T13 đạt với ledger/metering theo kỳ và quy trình đối soát |

Không cần hoàn thiện mọi tiện ích SaaS trước G1. Cần sửa những đường dữ liệu không thể bỏ qua; các tính năng như impersonation rộng, chat hoặc usage billing có thể tạm tắt cho tới khi đáp ứng gate tương ứng, với dependency được kiểm.

Audit kỹ thuật hiện có không thay thế sổ kho/sổ công nợ có tính toàn vẹn nghiệp vụ. Các chứng từ đã ghi sổ cần được điều chỉnh/đảo theo quy tắc, thay vì sửa/xóa và chỉ trông chờ log để tái dựng lịch sử.
