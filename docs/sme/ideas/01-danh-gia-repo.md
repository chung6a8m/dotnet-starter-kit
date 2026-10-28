# Đánh giá repo làm nền ứng dụng SME với Database-per-Tenant

## 1. Kết luận và phạm vi đánh giá

**Kết luận: nền tảng phù hợp có điều kiện, chưa chứng nhận production.**
Repo đã có định tuyến database riêng theo tenant, tenant isolation, migrations, provisioning và một số integration test dùng database riêng thực.
Phần cần bổ sung chủ yếu nằm ở tính nhất quán của các đường xử lý tenant, độ bền của provisioning và cách vận hành database riêng.
Không cần viết lại toàn bộ nền tảng để đạt mục tiêu SME; cần sửa các khoảng trống có bằng chứng trước khi cam kết cách ly dữ liệu với khách hàng.

| Thuộc tính | Giá trị |
|---|---|
| Repository khảo sát | [chung6a8m/dotnet-starter-kit](https://github.com/chung6a8m/dotnet-starter-kit) |
| Commit cố định | `a967537526f3b769c7d268da3ed2a6920af65ce9` |
| Ngày đánh giá | 2026-10-28 |
| Mục tiêu giả định | Mỗi SME là một tenant; dữ liệu nghiệp vụ chính nằm trong database PostgreSQL riêng |
| Phương pháp | Đọc source, cấu hình, migrations và test liên quan; đối chiếu luồng gọi |
| Giới hạn | Phần khảo sát này không chạy build/test, không đo tải và chưa tái hiện các failure window trong môi trường runtime |

Các liên kết nguồn ở cuối tài liệu đều trỏ tới commit cố định, không trỏ tới nhánh có thể thay đổi.
“Quan sát từ mã” mô tả hành vi thấy trực tiếp trong source; “cần tái hiện” là hệ quả cần integration test hoặc fault injection xác nhận.
Việc thấy một test trong repo không đồng nghĩa test đó đã chạy thành công trong phiên đánh giá này.

| Hạng mục | Verdict | Điều kiện sử dụng |
|---|---|---|
| Nền ứng dụng SME dạng modular monolith | Phù hợp | Giữ module boundaries, chọn phạm vi nghiệp vụ vừa đủ |
| Routing HTTP tới database riêng | Đã có nền tảng thực | Bổ sung chính sách bắt buộc dedicated cho tenant khách hàng |
| Tenant isolation theo hàng | Đã có và nên giữ | Kiểm tra mọi đường `IgnoreQueryFilters`, background job và entity global |
| Vòng đời provisioning nhiều process | Chưa hoàn chỉnh | Giải quyết password handoff, trạng thái Ready và retry đồng thời |
| Quan sát migration của database riêng | Có sai lệch trong implementation | Sửa K02 trước khi dùng làm readiness hoặc release gate |
| Eventing và nghiệp vụ cần tính nhất quán cao | Có building blocks, cần kiểm tra từng luồng | Sửa K01 và xác định transaction/idempotency theo K06 |
| Đưa production ngay theo cấu hình mẫu | Chưa đủ bằng chứng | Hoàn tất acceptance tests và vận hành trong tài liệu 03 |

## 2. Hiện trạng công nghệ và cấu trúc

Backend target `.NET 10`, dùng EF Core 10, Finbuckle 10.1, Mediator và Hangfire theo các project/property files [S01], [S02].
Đây là modular monolith: host lắp ráp module; mỗi module có runtime project và Contracts riêng, với quy tắc dependency được mô tả trong repo [S03].
Hai ứng dụng React/Vite tách vai trò operator và tenant; manifest admin có React 19, Vite 7, TypeScript, TanStack Query và Playwright [S04], [S58].
AppHost kết nối PostgreSQL, Valkey, object storage và migrator; đây là môi trường phát triển thuận tiện, không tự chứng minh cấu hình production tối thiểu quyền [S05].

| Thành phần dữ liệu | Vị trí hiện tại | Ý nghĩa đối với Database-per-Tenant |
|---|---|---|
| `TenantDbContext` | Database chính; bảng tenant, provisioning, theme, expiry notice | Control plane, dùng để tìm tenant trước khi chọn database nghiệp vụ [S06] |
| `BillingDbContext` | Cố ý nằm database chính; subscriptions, invoices, wallets, usage snapshots | Control plane phục vụ quản trị xuyên tenant; ownership lọc ở query services [S07] |
| `IdentityDbContext` | Default DB hoặc tenant connection | User, role, session có thể nằm trong DB riêng [S08] |
| Catalog, Tickets, Files, Chat, Notifications, Webhooks, Auditing | Các context kế thừa `BaseDbContext` | Có đường routing theo connection của tenant; cần kiểm tra cả các caller background [S09], [S10] |
| `EventingDbContext` | Kế thừa `BaseDbContext`, sở hữu outbox/inbox | Có thể đặt message cùng database nghiệp vụ để tham gia local transaction [S11] |
| Schema nghiệp vụ | Schema cố định theo module, ví dụ `catalog` | Tách schema theo module; không phải một schema được sinh riêng theo tenant [S10] |
| Migrations | Project `FSH.Starter.Migrations.PostgreSQL`, phân folder theo module | Cần nâng version schema cho mỗi DB vật lý và theo dõi kết quả [S12] |

Vì Billing và tenant registry được giữ tập trung, “database riêng” ở đây nên được định nghĩa là **database nghiệp vụ riêng cho mỗi SME**.
Nếu hợp đồng yêu cầu toàn bộ billing, registry, logs và object storage đều nằm trong hạ tầng riêng, phạm vi cần thiết lớn hơn khả năng routing hiện có.
Không nên coi việc Billing dùng DB trung tâm là bug; cần công khai ranh giới dữ liệu này trong thiết kế và chính sách lưu trữ.

Có nhánh `UseSqlServer` trong helper cấu hình, nhưng không đủ để xem SQL Server là lựa chọn thay thế chạy ngay [S13].
Migrator dùng `PostgresMigratorLock`, migrations được đóng gói cho PostgreSQL, và outbox claim có SQL `SKIP LOCKED` riêng cho PostgreSQL [S12], [S14], [S15].
Ngoài PostgreSQL, outbox fallback được code ghi rõ chỉ an toàn khi một dispatcher chạy; nên chọn PostgreSQL cho baseline SME này [S15].

## 3. Những phần nên tận dụng

**Tenant resolution đã có phân quyền hợp lý.** Strategy chain là claim, header rồi query; store chain là distributed cache rồi EF store [S16].
Caller thường không thể dùng header để ghi đè tenant trong JWT; root operator có middleware override riêng, chỉ khi claim là root và target tồn tại [S17].
Test đã có trường hợp tenant A gửi header B, root override và permission cache lạnh; fixtures của suite này vẫn dùng DB chung [S18].

**Isolation mặc định bật.** `BaseDbContext` áp dụng tenant isolation; extension chỉ bỏ qua entity được đánh dấu `IGlobalEntity` hoặc các loại không áp dụng được [S19].
Unique indexes được điều chỉnh theo tenant; giữ cơ chế này khi chuyển dedicated DB giúp giảm rủi ro ở tooling và các môi trường vẫn dùng shared DB.
Database riêng không thay thế việc gắn `TenantId` cho cache, message, object key và audit context.

**Migrate/seed tạo scope đúng thứ tự.** `TenantService` tạo scope, đặt tenant context trước khi resolve `IDbInitializer`, rồi mới gọi initializer từng module [S20].
DbMigrator là process riêng, migrate catalog trước rồi lặp qua tenant; đây là nền tảng tốt để tách bước nâng schema khỏi API startup [S14].

**Có nền tảng outbox/inbox và chia sẻ connection.** `ScopedDbConnectionProvider` giữ một connection object theo connection string trong DI scope [S21].
`BaseDbContext` dùng provider này khi routing dedicated, còn `EfCoreOutboxStore` có thể enlist vào transaction của caller [S09], [S22].
Đây là khả năng cần sử dụng đúng ở application flow, không phải bằng chứng mọi command đã atomic.

**Đã có test dedicated thực.** `PurgeJobsPerTenantTests` tạo một connection với tên database mới, provision, đăng nhập tenant admin và kiểm tra Files purge [S23].
Vì vậy không có cơ sở nói repo chỉ hỗ trợ database riêng trên README, hoặc toàn bộ integration tests đều chỉ chạy DB chung.

## 4. Các phát hiện cần xử lý

Mức ưu tiên dưới đây áp dụng theo mục tiêu SME Database-per-Tenant, không phải điểm CVSS hay tuyên bố lỗ hổng đã khai thác.
Module tùy chọn có thể tạm tắt; các nền tảng chung như routing, secrets và readiness phải được giải quyết trước khi nhận dữ liệu thật.

| Mã | Phát hiện | Ưu tiên | Mức xác minh |
|---|---|---|---|
| K01 | Event handler scope chỉ giữ tenant ID, làm mất connection riêng | Cao nếu bật event-driven modules | Trace source rõ; cần tái hiện ghi/đọc ở sai database |
| K02 | Migration status/health dùng catalog context; CLI list-pending thiếu per-module | Cao | Sai mục tiêu thấy trực tiếp trong source |
| K03 | Provisioning active sớm, step placeholder, password memory và race windows | Cao | Hành vi source rõ; crash/concurrency cần fault injection |
| K04 | Dedicated chưa bắt buộc; secret và credential lifecycle chưa tách | Cao | Contract, DTO và connection routing thấy trực tiếp |
| K05 | Impersonation xuyên tenant có giả định cùng database | Cao nếu bật impersonation | Query path rõ; cần test hai dedicated DB và cache lạnh |
| K06 | Transaction/inbox không tự bảo đảm exactly-once cho nghiệp vụ | Cao với số dư, tồn kho, hóa đơn | Cơ chế rõ; từng business failure window cần tái hiện |

### K01 — Event tenant scope làm mất thông tin database

**Quan sát từ mã:** `FinbuckleEventTenantScope.Begin` tạo `AppTenantInfo(tenantId, tenantId)` mới, không resolve connection string từ registry [S24].
`InMemoryEventBus` mở scope này trước khi resolve handlers và inbox; các DbContext mới được tạo dưới tenant có ID đúng nhưng connection trống [S25].
Theo `BaseDbContext.OnConfiguring`, connection trống giữ default connection, nên cùng tenant ID không đủ bảo đảm đúng DB vật lý [S09].

Trace cụ thể: Chat publish mention qua event bus; Notifications handler nhận `NotificationsDbContext` trong constructor rồi ghi notification [S26], [S27].
Guard của handler chỉ so sánh tenant ID, nên không phát hiện trường hợp ID đúng nhưng database sai.
`WebhookFanoutHandler` cũng nhận DbContext trước, sau đó tự đặt một `AppTenantInfo` chỉ có ID và đọc subscriptions [S28].
Đây là hai đường xử lý cần kiểm tra dù HTTP routing thông thường đã đúng.

**Phân biệt phần đã đúng:** outbox drain có `TenantStoreDrainTargetProvider` và `FinbuckleEventingDrainScope` mang connection string thật [S29], [S54].
Vấn đề là bước dispatch downstream có thể thay context đầy đủ bằng context chỉ có identity; không nên kết luận toàn bộ outbox không hỗ trợ DB riêng.

**Hệ quả cần tái hiện:** notification xuất hiện ở default DB hoặc tenant không thấy notification; webhook bỏ sót subscriptions nằm ở dedicated DB.
Đây chưa phải bằng chứng tenant khác đọc được dữ liệu; tenant row filter và physical routing là hai lớp khác nhau.
**Hướng sửa:** với consumer nghiệp vụ tenant, resolve descriptor đầy đủ trước khi tạo handler scope; với consumer control-plane như Billing, chọn scope trung tâm theo ownership. Outbox theo producer, inbox theo consumer; không chuyển mọi event có TenantId sang tenant DB. Xem quy tắc ở mục 8.2 tài liệu 02.
**Acceptance:** tạo dedicated tenant A, phát event từ HTTP lẫn background, kiểm A có side effect đúng một lần và default DB không có bản ghi nghiệp vụ của A.

### K02 — Migration status và readiness đang kiểm catalog

**Quan sát từ mã:** `GetTenantMigrationsQueryHandler` và `TenantMigrationsHealthCheck` lặp tenant, set context rồi resolve `TenantDbContext` [S30], [S31].
Nhưng `TenantDbContext` là context của registry, còn `AddHeroDbContext` cấp default connection; context này không routing theo tenant [S06], [S32].
Kết quả là kiểm catalog nhiều lần, không kiểm Identity/Catalog/Files migrations tại từng dedicated DB.

DbMigrator có thực hiện per-tenant migration; riêng nhánh `list-pending` chỉ in mô tả cho mỗi tenant, không liệt kê pending của các initializer [S33].
Đây là khác biệt giữa khả năng migrate và khả năng báo cáo trạng thái, cần phản ánh đúng trên dashboard vận hành.

**Hệ quả suy ra:** catalog có thể ở head trong khi dedicated DB down hoặc một module còn pending, nhưng báo cáo vẫn thể hiện không pending.
Không dùng health check này làm bằng chứng toàn fleet sẵn sàng trước khi sửa.
Test health hiện cố định connection cho `TenantDbContext`; test endpoint tạo tenant có connection null, nên chưa phân biệt được lỗi này [S34], [S55].

**Hướng sửa:** check catalog riêng; module cung cấp migration probe cho context của mình; aggregate theo database vật lý và version/module.
**Acceptance:** catalog ở head, DB A đủ schema, DB B thiếu một migration, DB C không kết nối được; báo cáo phải phân biệt cả ba.
Chính sách readiness toàn API khi một tenant lỗi cần thiết kế riêng để tránh một khách hàng làm gián đoạn toàn hệ thống; xem tài liệu 03.

### K03 — Provisioning có tiến trình lưu bền nhưng chưa đủ bền khi lỗi

**Active sớm:** constructor `AppTenantInfo` đặt `IsActive = true`; create không đợi provisioning Completed [S35], [S36].
Guard `EnsureCanActivateAsync` chỉ nằm ở activation path; test failure còn phải deactivate tenant trước khi kiểm reactivation [S37].
Vì vậy `IsActive` hiện chưa có nghĩa “đã sẵn sàng phục vụ”; không nên dùng riêng cờ này làm readiness tenant.

**Step placeholder:** `Database` chỉ đặt context/đánh dấu Completed, `CacheWarm` chỉ đánh dấu Completed; bước thực có I/O là Migrations và Seeding [S38].
Không có cấp phát role/grant/hạ tầng trong hai placeholder này.
EF migration vẫn có thể tạo database khi đủ quyền; test dedicated ở S23 cho thấy không nên kết luận repo hoàn toàn thiếu khả năng tạo DB.

**Password handoff:** create lưu admin password vào singleton `ConcurrentDictionary`; seed consume rồi xóa [S36], [S39].
Worker khác process hoặc restart sẽ không có entry; `IdentityDbInitializer` fallback sang `Seed:DefaultAdminPassword` nếu được cấu hình, không giới hạn cho root [S40].
Nếu không có fallback thì seed fail; nếu có, admin có thể được tạo với password cấu hình thay vì password operator đã cung cấp.
Đây là hệ quả của lifetime bộ nhớ; cần test crash và retry để xác nhận trải nghiệm end-to-end.

**Race và trạng thái mắc kẹt:** Start kiểm latest attempt rồi insert; save provisioning, enqueue job và save job ID là các bước tách nhau [S41].
Không thấy unique active-attempt constraint hoặc concurrency token trong mapping provisioning [S42].
Pending/Running bị từ chối retry; process chết giữa save và enqueue có thể cần reconciliation, không thể chỉ dựa vào endpoint retry.
Manual retry tạo attempt mới; skip completed step chỉ áp dụng khi chạy lại cùng attempt [S41].

**Hướng sửa:** trạng thái Provisioning/Ready/Failed rõ ràng; chỉ Ready mới phục vụ tenant; lease theo tenant và xử lý attempt stale.
Thay password buffer bằng invitation/set-password hoặc secret tạm bền vững có TTL; không dùng root seed password làm fallback cho tenant thường.
**Acceptance:** hai request retry đồng thời, worker A/B, kill giữa từng step, seed lỗi sau consume secret và phục hồi khi queue tạm mất kết nối.

### K04 — Dedicated còn là tùy chọn; quản lý secret chưa tách khỏi tenant DTO

**Quan sát từ mã:** validator chấp nhận connection null/rỗng; `TenantService` còn chuẩn hóa chuỗi bằng default connection thành rỗng [S43], [S44].
`BaseDbContext` fallback về database mặc định khi tenant connection không có [S09].
Đây là hành vi hợp lệ của mô hình hybrid, nhưng chưa đáp ứng chính sách “mọi tenant khách hàng bắt buộc database riêng”.

`AppTenantInfo.ConnectionString` là chuỗi kết nối trực tiếp; tenant list DTO nhận nguyên giá trị này [S35], [S45].
Endpoint list có permission gate, nên không mô tả đây là public credential leak; vấn đề là DB secret đi vào admin DTO/browser không cần thiết [S56].
Connection string validator chỉ parse cú pháp, không chứng minh reachability, ownership, database độc quyền hoặc quyền tối thiểu [S46].

Migrator có thể dùng elevated default connection, nhưng dedicated migration và runtime cùng đọc một tenant connection string [S14], [S20], [S09].
Không có field riêng cho runtime credential và migration credential trong `AppTenantInfo` [S35].
Do đó tách quyền cho database chính không tự giải quyết quyền của mọi database riêng.

**Hướng sửa:** descriptor có `DatabaseMode`, database identifier và secret reference; dedicated mode phải từ chối thiếu cấu hình thay vì fallback im lặng.
Connection string được xây dựng phía server từ registry/secrets; API trả alias và trạng thái, không trả credential.
Tách runtime user ít quyền khỏi provisioner/migrator và có đường rotation; ngân sách connection pool xem tài liệu 03.
**Acceptance:** tenant dedicated thiếu secret phải fail closed; runtime credential không migrate được; migrator đúng quyền vẫn nâng schema được.

### K05 — Impersonation xuyên database cần đường truy cập rõ ràng

**Quan sát từ mã:** start handler kiểm actor root/cùng tenant rồi gọi `BuildClaimsForUserAsync(targetUserId, targetTenantId)` [S47].
Method này dùng `_userManager.Users.IgnoreQueryFilters()` và `_dbContext` đã inject, có lọc lại tenant ID nhưng không tạo scope/database target [S48].
`IgnoreQueryFilters` bỏ điều kiện hàng trong database hiện tại; nó không chuyển kết nối sang database khác.
Grant service cũng lưu/đọc grants qua `IdentityDbContext` của scope hiện tại [S49].

**Hệ quả cần tái hiện:** root ở DB chính có thể không tìm được user nằm DB A nếu chỉ truyền `TargetTenantId` trong body.
Nếu header override làm query target chạy được, vẫn phải kiểm tra nơi lưu grant, end/revoke và kiểm JWT khi cache lạnh hoặc process khác.
Test impersonation đã có cross-tenant authorization, nhưng helper tạo tenant dùng connection null nên không chứng minh cross-database lifecycle [S50].

**Hướng sửa:** chốt nơi lưu grants và audit của operator; resolve full target descriptor trong scope riêng; actor context và target context phải tách rõ.
**Acceptance:** start/use/end/revoke giữa root và hai dedicated DB, có/không header target, Redis cache lạnh và restart API.
Nếu SME giai đoạn đầu không cần impersonation, tắt chức năng trước khi mở production sẽ giảm phạm vi cần xác minh.

### K06 — Outbox có khả năng atomic; từng nghiệp vụ vẫn cần transaction và idempotency

**Phần đã có:** outbox store enlist transaction tìm thấy trên cùng connection object; test kiểm commit/rollback và shared connection thực [S22], [S51].
Khi không có ambient transaction, `AddAsync` tự SaveChanges; không tự mở một transaction bao trùm mọi thao tác của caller [S22].
Ví dụ create tenant ghi catalog, gọi provisioning rồi mới thêm subscription event; không thấy một transaction chung ở handler [S36].
Vì vậy cần xác định từng failure window, không suy rộng tên “transactional outbox” thành toàn workflow đã atomic.

**Inbox:** bus kiểm processed, chạy handler rồi mới `MarkProcessedAsync`; business side effect và marker không được bọc bởi transaction chung tại bus [S25].
Crash sau side effect nhưng trước marker có thể khiến xử lý lại; hai consumer đồng thời cũng cần idempotency nghiệp vụ ngoài thao tác kiểm trước.
Inbox có composite key `(Id, HandlerName)` [S57], nhưng store bắt mọi `DbUpdateException` rồi coi như insert concurrent đã thắng [S52].
Nên phân biệt lỗi unique-conflict với lỗi ghi DB khác thay vì mặc định marker đã bền vững.

**Hệ quả cần kiểm chứng theo use case:** không ghi nhận số dư, điều chỉnh tồn kho hay phát hành hóa đơn hai lần khi event được giao lại.
At-least-once là mô hình có thể dùng tốt, miễn side effect có unique operation key và transaction đúng database.
Không giả định một local transaction bao trùm Billing trung tâm và dữ liệu nghiệp vụ trong DB tenant.

`IdentityDbContext` dedicated hiện dùng connection string trực tiếp, khác cơ chế scoped provider ở `BaseDbContext` [S08], [S09].
Đây là điểm cần thống nhất nếu tương lai có Identity transaction chung với outbox; khảo sát này chưa chứng minh một explicit Identity transaction đang hỏng.
**Acceptance:** rollback business/outbox trên dedicated DB; crash sau side effect; hai consumer nhận cùng event; lỗi ghi inbox không phải unique-conflict.

## 5. Đọc đúng mức độ kiểm thử hiện có

| Bộ test đọc được | Đã mô tả/kiểm tra trong source | Không nên suy rộng thành |
|---|---|---|
| `PurgeJobsPerTenantTests` | Root + shared tenant + dedicated DB mới; provision/login và Files purge [S23] | Mọi module/background path đều đã được test dedicated |
| `TenantHeaderOverrideTests` | Root override, claim thắng header tenant khác, permission cache lạnh [S18] | Toàn bộ root operations xuyên database đã đúng |
| `TenantProvisioningFailureTests` | Connection không truy cập được, Failed ở Migrations, reactivation guard [S37] | Tenant chưa Ready luôn inactive hoặc crash recovery đầy đủ |
| `RetryTenantProvisioningTests` | Attempt mới và retry tenant đã hoàn thành [S53] | Chuyển worker, mất password buffer hoặc retry concurrent an toàn |
| `TenantMigrationsHealthCheckTests` | Pending/error trên context fixture của catalog [S34] | Phát hiện drift hoặc outage từng dedicated module DB |
| `OutboxAtomicityTests` | Explicit transaction commit/rollback với root fixture [S51] | Mọi command và dedicated Identity flow atomic |
| `ImpersonationTests` | Authorization và grant lifecycle với helper dùng shared DB [S50] | Grant lifecycle xuyên database đã được xác nhận |

Các kết quả ở bảng là đánh giá nội dung test, không phải log thực thi mới.
Nên bổ sung test theo failure mode của K01–K06 thay vì chỉ tăng số lượng test CRUD thành công.
Một bộ acceptance tối thiểu cần hai dedicated DB, một catalog DB và kiểm tra trực tiếp cả database đích lẫn database không được phép ghi.

## 6. Hướng dùng repo cho SME

Giữ modular monolith, PostgreSQL, tenant filters và DbMigrator; chưa có lý do từ khảo sát này để tách microservices ngay.
Quyết định sớm phạm vi tenant: một doanh nghiệp; chi nhánh, kho và phòng ban nên là cấu trúc bên trong tenant nếu không có yêu cầu cách ly riêng.
Database riêng giúp xác định ranh giới dữ liệu; hiệu quả thực tế còn phụ thuộc provisioning, backup/restore, secrets, routing và test.

Ưu tiên sửa K02, K03, K04 và mọi phần K01/K05/K06 thuộc module sẽ bật.
Chỉ đưa thêm CRM, bán hàng, mua hàng, kế toán hoặc tồn kho sau khi các ranh giới dữ liệu nền đã ổn định.
Các module mẫu Catalog/Tickets hữu ích để học cấu trúc, nhưng không phải chứng nhận đủ nghiệp vụ ngành hoặc tính đúng đắn tài chính.

Tài liệu tiếp theo:

- [02 — Kiến trúc đề xuất](02-kien-truc-de-xuat.md): control plane, tenant plane, descriptor, scoped execution và ranh giới module.
- [03 — Vận hành và an toàn dữ liệu](03-van-hanh-va-an-toan-du-lieu.md): migrations, pool, backups, restore, secrets, metering và điều kiện production.
- [04 — Phạm vi SME và lộ trình](04-pham-vi-sme-va-lo-trinh.md): phạm vi sản phẩm, thứ tự triển khai và tiêu chí nghiệm thu.

## Đối chiếu nguồn

Mỗi ký hiệu nguồn trong nội dung mở đúng file và dòng tại commit khảo sát.
Các nguồn bao gồm implementation và test; liên kết test chỉ chứng minh nội dung kiểm thử được lưu trong repo.

[S01]: https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/src/Directory.Build.props#L4
[S02]: https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/src/Directory.Packages.props#L46
[S03]: https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/.agents/rules/architecture.md#L5
[S04]: https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/clients/admin/package.json#L14
[S05]: https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/src/Host/FSH.Starter.AppHost/AppHost.cs#L13
[S06]: https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/src/Modules/Multitenancy/Modules.Multitenancy/Data/TenantDbContext.cs#L9
[S07]: https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/src/Modules/Billing/Modules.Billing/Data/BillingDbContext.cs#L6
[S08]: https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/src/Modules/Identity/Modules.Identity/Data/IdentityDbContext.cs#L67
[S09]: https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/src/BuildingBlocks/Persistence/Context/BaseDbContext.cs#L49
[S10]: https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/src/Modules/Catalog/Modules.Catalog/Data/CatalogDbContext.cs#L12
[S11]: https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/src/BuildingBlocks/Eventing/Persistence/EventingDbContext.cs#L25
[S12]: https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/src/Host/FSH.Starter.Migrations.PostgreSQL/FSH.Starter.Migrations.PostgreSQL.csproj#L4
[S13]: https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/src/BuildingBlocks/Persistence/OptionsBuilderExtensions.cs#L45
[S14]: https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/src/Host/FSH.Starter.DbMigrator/Program.cs#L170
[S15]: https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/src/BuildingBlocks/Eventing/Outbox/EfCoreOutboxStore.cs#L76
[S16]: https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/src/Modules/Multitenancy/Modules.Multitenancy/MultitenancyModule.cs#L103
[S17]: https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/src/Modules/Multitenancy/Modules.Multitenancy/MultitenancyModule.cs#L140
[S18]: https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/src/Tests/Integration.Tests/Tests/Multitenancy/TenantHeaderOverrideTests.cs#L69
[S19]: https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/src/BuildingBlocks/Persistence/TenantIsolationExtensions.cs#L30
[S20]: https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/src/Modules/Multitenancy/Modules.Multitenancy/Services/TenantService.cs#L95
[S21]: https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/src/BuildingBlocks/Persistence/ScopedDbConnectionProvider.cs#L17
[S22]: https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/src/BuildingBlocks/Eventing/Outbox/EfCoreOutboxStore.cs#L198
[S23]: https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/src/Tests/Integration.Tests/Tests/Files/PurgeJobsPerTenantTests.cs#L117
[S24]: https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/src/Modules/Multitenancy/Modules.Multitenancy/Services/FinbuckleEventTenantScope.cs#L32
[S25]: https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/src/BuildingBlocks/Eventing/InMemory/InMemoryEventBus.cs#L55
[S26]: https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/src/Modules/Chat/Modules.Chat/Features/v1/Messages/SendMessage/SendMessageCommandHandler.cs#L101
[S27]: https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/src/Modules/Notifications/Modules.Notifications/IntegrationEventHandlers/MentionedInChannelIntegrationEventHandler.cs#L23
[S28]: https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/src/Modules/Webhooks/Modules.Webhooks/Services/WebhookFanoutHandler.cs#L33
[S29]: https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/src/Modules/Multitenancy/Modules.Multitenancy/Services/FinbuckleEventingDrainScope.cs#L29
[S30]: https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/src/Modules/Multitenancy/Modules.Multitenancy/Features/v1/GetTenantMigrations/GetTenantMigrationsQueryHandler.cs#L47
[S31]: https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/src/Modules/Multitenancy/Modules.Multitenancy/TenantMigrationsHealthCheck.cs#L42
[S32]: https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/src/BuildingBlocks/Persistence/PersistenceExtensions.cs#L54
[S33]: https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/src/Host/FSH.Starter.DbMigrator/Program.cs#L266
[S34]: https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/src/Tests/Integration.Tests/Tests/Health/TenantMigrationsHealthCheckTests.cs#L82
[S35]: https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/src/BuildingBlocks/Shared/Multitenancy/AppTenantInfo.cs#L26
[S36]: https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/src/Modules/Multitenancy/Modules.Multitenancy/Features/v1/CreateTenant/CreateTenantCommandHandler.cs#L37
[S37]: https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/src/Tests/Integration.Tests/Tests/Multitenancy/TenantProvisioningFailureTests.cs#L131
[S38]: https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/src/Modules/Multitenancy/Modules.Multitenancy/Provisioning/TenantProvisioningJob.cs#L39
[S39]: https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/src/Modules/Multitenancy/Modules.Multitenancy/Services/TenantInitialPasswordBuffer.cs#L6
[S40]: https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/src/Modules/Identity/Modules.Identity/Data/IdentityDbInitializer.cs#L246
[S41]: https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/src/Modules/Multitenancy/Modules.Multitenancy/Provisioning/TenantProvisioningService.cs#L36
[S42]: https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/src/Modules/Multitenancy/Modules.Multitenancy/Data/Configurations/TenantProvisioningConfiguration.cs#L10
[S43]: https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/src/Modules/Multitenancy/Modules.Multitenancy/Features/v1/CreateTenant/CreateTenantCommandValidator.cs#L22
[S44]: https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/src/Modules/Multitenancy/Modules.Multitenancy/Services/TenantService.cs#L72
[S45]: https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/src/Modules/Multitenancy/Modules.Multitenancy/Features/v1/GetTenants/GetTenantsSpecification.cs#L29
[S46]: https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/src/BuildingBlocks/Persistence/ConnectionStringValidator.cs#L19
[S47]: https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/src/Modules/Identity/Modules.Identity/Features/v1/Impersonation/StartImpersonation/StartImpersonationCommandHandler.cs#L68
[S48]: https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/src/Modules/Identity/Modules.Identity/Services/IdentityService.cs#L143
[S49]: https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/src/Modules/Identity/Modules.Identity/Services/ImpersonationGrantService.cs#L12
[S50]: https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/src/Tests/Integration.Tests/Tests/Impersonation/ImpersonationTests.cs#L894
[S51]: https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/src/Tests/Integration.Tests/Tests/Eventing/OutboxAtomicityTests.cs#L49
[S52]: https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/src/BuildingBlocks/Eventing/Inbox/EfCoreInboxStore.cs#L31
[S53]: https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/src/Tests/Integration.Tests/Tests/Multitenancy/RetryTenantProvisioningTests.cs#L41
[S54]: https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/src/Modules/Multitenancy/Modules.Multitenancy/Services/TenantStoreDrainTargetProvider.cs#L22
[S55]: https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/src/Tests/Integration.Tests/Tests/Multitenancy/TenantMigrationsTests.cs#L148
[S56]: https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/src/Modules/Multitenancy/Modules.Multitenancy/Features/v1/GetTenants/GetTenantsEndpoint.cs#L19
[S57]: https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/src/BuildingBlocks/Eventing/Inbox/InboxMessage.cs#L37
[S58]: https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/clients/dashboard/package.json#L14
