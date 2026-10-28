# Phạm vi SME và lộ trình triển khai

## 1. Mục đích, giả định và cách đọc

Tài liệu này đề xuất cách dùng FullStackHero làm nền tảng cho một sản phẩm SME có **database riêng cho từng tenant**. Khuyến nghị hiện tại là tiếp tục khảo sát và hoàn thiện nền tảng, sau đó xây một MVP nghiệp vụ có phạm vi hẹp. Việc chọn repo không đồng nghĩa với việc đã có một hệ thống bán hàng, quản lý kho hoặc kế toán sẵn sàng đưa vào vận hành.

**Giả định sản phẩm chưa được chốt:** nhóm khách hàng ban đầu là doanh nghiệp nhỏ trong ngành thương mại hoặc dịch vụ, cần quản lý khách hàng, danh mục hàng hóa/dịch vụ, bán hàng, kho đơn giản và công nợ phải thu. Các lựa chọn về ngành, số tenant, tải đồng thời, pháp nhân, quy trình duyệt và tích hợp phải được xác nhận trong phase 0. Nếu nhóm khách hàng thực tế là sản xuất, xây dựng, y tế, tài chính hoặc có nghiệp vụ chuyên ngành khác, cần đánh giá lại phạm vi và ước lượng.

**Mốc mã nguồn:** `a967537526f3b769c7d268da3ed2a6920af65ce9`. Các liên kết bằng chứng trong tài liệu trỏ tới commit cố định này. “Hiện trạng” là kết quả đọc mã và cấu hình; “đề xuất”, “nghiệm thu” và “ước lượng” là kế hoạch tương lai. Khảo sát này chưa chạy ứng dụng, test, migration, restore hoặc tải thực tế, nên không có tuyên bố test đã pass hay cam kết năng lực tải.

Repo khai báo .NET 10, SDK gốc 10.0.100 với `rollForward: latestFeature`; EF Core được pin 10.0.12, Finbuckle 10.1.0. Hai frontend dùng React 19 và Vite 7. Đây là phiên bản khai báo tại commit, không phải kết luận về phiên bản mới nhất hoặc tình trạng hỗ trợ hiện tại. Xem [global.json](https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/global.json#L1-L6), [package versions](https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/src/Directory.Packages.props#L46-L75) và [frontend dependencies](https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/clients/admin/package.json#L14-L54).

## 2. Ranh giới tenant, công ty và chi nhánh

**Đề xuất:** tenant là ranh giới hợp đồng SaaS, phân quyền cao nhất, định tuyến dữ liệu và phục hồi dữ liệu. Công ty/pháp nhân và chi nhánh là khái niệm nghiệp vụ bên trong tenant. Không dùng cùng một trường `TenantId` để đồng thời diễn tả cả ba khái niệm.

| Khái niệm | Ý nghĩa đề xuất | Lựa chọn MVP |
|---|---|---|
| Tenant | Một tổ chức khách hàng sử dụng SaaS; có định danh ổn định, subscription và database nghiệp vụ riêng | Một khách hàng SME tương ứng một tenant |
| Company | Pháp nhân hoặc đơn vị sở hữu chứng từ, mã số thuế và chính sách nghiệp vụ | Một company cho mỗi tenant; đặt khóa company rõ ràng nếu dự kiến mở rộng |
| Branch | Chi nhánh/địa điểm vận hành; có thể gắn người dùng, chứng từ và kho | Thiết kế `BranchId`; chỉ triển khai số chi nhánh tối thiểu mà pilot cần |
| Warehouse | Nơi ghi nhận tồn kho và chuyển động hàng hóa | Một kho ban đầu; chỉ hỗ trợ nhiều kho khi luồng điều chuyển và phân quyền được chốt |
| Platform operator | Nhân viên vận hành nhà cung cấp SaaS | Dùng admin app; thao tác hỗ trợ tenant có lý do, hạn sử dụng và audit |
| Tenant administrator | Người quản trị của khách hàng | Quản trị thành viên/quyền trong dashboard; không nhận quyền platform operator |

Một tenant có nhiều chi nhánh không tự động cần nhiều database. Tách tenant theo chi nhánh chỉ hợp lý khi mỗi chi nhánh thực sự là khách hàng hoặc ranh giới dữ liệu độc lập; nếu không, báo cáo hợp nhất, di chuyển người dùng và chứng từ sẽ trở nên khó hơn. Đây là quyết định mô hình sản phẩm cần chốt trước khi tạo dữ liệu thật.

Database-per-tenant cũng không đồng nghĩa với một máy chủ PostgreSQL riêng cho mỗi tenant. Có thể bắt đầu bằng các database riêng trên một cụm PostgreSQL dùng chung, với giới hạn tài nguyên và quyền truy cập được thiết kế rõ. Cụm dùng chung vẫn là miền lỗi dùng chung; chỉ nâng một tenant lên instance/cụm riêng khi yêu cầu hợp đồng, tải hoặc khả năng vận hành chứng minh cần thiết.

Billing SaaS là một phần của dữ liệu vận hành nền tảng: `BillingDbContext` mô tả rõ invoices/subscriptions ở database chính, dùng `TenantId` tường minh và lọc trong query services. Không chuyển các hóa đơn bán hàng của khách hàng vào bảng billing này. Xem [BillingDbContext.cs:6–25](https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/src/Modules/Billing/Modules.Billing/Data/BillingDbContext.cs#L6-L25).

## 3. Thành phần có thể tận dụng và phần nghiệp vụ cần xây

Host hiện đăng ký 10 module: Identity, Multitenancy, Auditing, Files, Webhooks, Billing, Catalog, Tickets, Chat và Notifications. Danh sách này là căn cứ xác định phạm vi có sẵn; không có module Sales, Purchasing, Inventory ledger, Receivables hoặc General Ledger trong danh sách module chạy của host. Xem [Program.cs:78–90](https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/src/Host/FSH.Starter.Api/Program.cs#L78-L90).

| Thành phần | Hiện trạng và giới hạn | Quyết định đề xuất cho MVP |
|---|---|---|
| Identity và phân quyền | Có module riêng; hai app có các màn hình quản lý người dùng/quyền. Dashboard chỉ kiểm tra đăng nhập ở route chung, còn menu có permission gate | Tận dụng; xác minh quyền bằng API và kiểm thử xuyên tenant. Bổ sung trải nghiệm rõ ràng cho đường dẫn trực tiếp không đủ quyền |
| Multitenancy | Có tenant catalog, provisioning và migrator; độ bao phủ database riêng cần nghiệm thu xuyên toàn bộ luồng | Giữ làm nền, nhưng phải qua phase 1 trước khi nhập dữ liệu nghiệp vụ thật |
| Auditing | Có module audit và logging theo user/tenant | Tận dụng cho truy vết; bổ sung quy tắc audit chứng từ, phân quyền đọc và retention |
| Files và Notifications | Có module Files/Notifications, route tệp và thông báo ở frontend | Tận dụng cho tệp đính kèm và thông báo nghiệp vụ sau khi xác minh isolation, vòng đời tệp và background jobs |
| Catalog | `Product` có SKU, tên, giá, stock dạng số nguyên, category/brand và ảnh; `AdjustStock` cập nhật một số dư | Tận dụng danh mục. Không xem trường `Stock` là sổ kho hoặc cơ chế giữ hàng đủ cho bán hàng đồng thời |
| Billing | Plan, subscription, usage invoice, invoice PDF và wallet/top-up; thanh toán invoice được đánh dấu thủ công | Chỉ dùng cho phí SaaS gói cố định trong MVP, tách khỏi công nợ bán hàng của SME |
| Tickets | Có state machine, người báo, người xử lý, comment, đóng/mở lại | Giữ nếu pilot cần tiếp nhận yêu cầu hỗ trợ; không mặc định dùng làm workflow duyệt nghiệp vụ |
| Chat, SSE và Webhooks | Là các phần được host đăng ký hoặc bật | Hoãn trải nghiệm chat và webhook tùy biến nếu không thuộc luồng giá trị chính; giữ phụ thuộc kỹ thuật thật sự cần thiết |
| CRM/bán hàng/kho/công nợ | Chưa có module nghiệp vụ tương ứng trong host; Catalog và Billing chỉ đáp ứng một phần dữ liệu nền | Xây các module riêng với contract và transaction boundary rõ ràng |

Bằng chứng chi tiết: [dashboard route guard](https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/clients/dashboard/src/auth/protected-route.tsx#L4-L31), [menu permission](https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/clients/dashboard/src/components/layout/nav-data.ts#L55-L125), [admin RouteGuard](https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/clients/admin/src/auth/route-guard.tsx#L26-L48), [Product fields](https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/src/Modules/Catalog/Modules.Catalog/Domain/Product.cs#L6-L27), [AdjustStock](https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/src/Modules/Catalog/Modules.Catalog/Domain/Product.cs#L139-L153), [Ticket lifecycle](https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/src/Modules/Tickets/Modules.Tickets/Domain/Ticket.cs#L9-L36) và [Billing endpoints](https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/src/Modules/Billing/Modules.Billing/BillingModule.cs#L85-L111).

### Phạm vi nghiệp vụ MVP đề xuất

Luồng giá trị cần hoàn thành là: **khách hàng → đơn bán → xác nhận bán/xuất kho → ghi nhận phải thu → thu tiền → đối chiếu**, kèm trả hàng/điều chỉnh có kiểm soát. Dịch vụ có thể bỏ bước xuất kho; hàng hóa phải ghi nhận chuyển động kho rõ ràng. Tên và ý nghĩa chứng từ cuối cùng cần được thống nhất với chủ doanh nghiệp và người phụ trách kế toán.

| Nhóm | Bao gồm ở MVP | Hoãn hoặc chỉ làm khi pilot chứng minh cần |
|---|---|---|
| Tổ chức và người dùng | Company, branch tối thiểu, vai trò chủ doanh nghiệp/bán hàng/kho/thu tiền, phạm vi truy cập | Hợp nhất nhiều pháp nhân, phân cấp tổ chức phức tạp |
| Danh mục | Khách hàng, sản phẩm/dịch vụ, đơn vị tính đơn giản, giá bán, import có kiểm tra và export | Bảng giá nhiều tầng, biến thể phức tạp, quy đổi nhiều đơn vị, CRM marketing |
| Bán hàng | Đơn bán, dòng hàng chụp lại đơn giá/thuế tại thời điểm lập, discount đơn giản, draft/post/cancel; quy tắc trả hàng tối thiểu | POS offline, khuyến mại đa điều kiện, marketplace, đa tiền tệ thực thu |
| Kho | Sổ chuyển động nhập/xuất/điều chỉnh; tồn đầu kỳ; chống ghi hai lần và tồn âm theo chính sách; đối chiếu số dư | Lô/serial/hạn dùng, WMS, sản xuất/BOM, tối ưu nhập hàng |
| Công nợ phải thu | Theo khách hàng và chứng từ; phiếu thu/phân bổ; thu một phần; hoàn/đảo giao dịch; aging cơ bản | Công nợ phải trả đầy đủ, ngân hàng tự động, tổng hợp kế toán, tài sản cố định, payroll |
| Báo cáo | Doanh số theo định nghĩa đã chốt, tồn kho, công nợ, lịch sử thay đổi; export có kiểm soát tenant/quyền | BI xuyên tenant thời gian thực, báo cáo tài chính hoặc báo cáo thuế đầy đủ |

Không thiết kế chứng từ đã ghi sổ như CRUD có thể sửa trực tiếp mọi trường. Đề xuất dùng trạng thái, version/concurrency token, idempotency và bút toán đảo/điều chỉnh để việc bán hàng, xuất kho và ghi phải thu có thể đối chiếu. Cách transaction hoặc outbox nối các module phải được lựa chọn sau khi xác minh nơi lưu dữ liệu thực tế; không mặc định một transaction EF bao phủ mọi module.

### VAT và hóa đơn điện tử

MVP chỉ nên chừa **adapter** kết nối nhà cung cấp hóa đơn điện tử và mô hình snapshot thông tin thuế trên chứng từ. Cần người có chuyên môn kế toán/thuế xác nhận cách đánh số, thời điểm phát hành, điều chỉnh/thay thế, thuế suất, quy tắc làm tròn và hồ sơ lưu trữ cho phân khúc khách hàng đã chọn. Đây là phạm vi thiết kế, không phải khẳng định repo hoặc MVP đã đáp ứng pháp luật Việt Nam. Nếu tích hợp là điều kiện bắt buộc của pilot, đưa nó thành hạng mục riêng với sandbox của nhà cung cấp, tiêu chí đối soát và nghiệm thu chuyên môn; cập nhật lại lịch và chi phí.

## 4. Các khoảng trống phải xử lý trước khi mở rộng nghiệp vụ

### 4.1. Thu phí SaaS: gói cố định trước, usage billing sau

Endpoint thanh toán hiện tự mô tả là đánh dấu invoice đã thanh toán thủ công, không có payment processor. Luồng này có thể phù hợp với MVP thu phí hợp đồng/chuyển khoản và đối soát thủ công, nếu bổ sung người thực hiện, chứng cứ, quyền duyệt và audit theo quy trình kinh doanh. Xem [MarkInvoicePaidEndpoint.cs:16–22](https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/src/Modules/Billing/Modules.Billing/Features/v1/Invoices/MarkInvoicePaid/MarkInvoicePaidEndpoint.cs#L16-L22).

**Khoảng trống đọc từ call chain:** job tháng chạy 00:05 UTC ngày đầu tháng và yêu cầu lập invoice cho tháng trước; `UsageReporter` lại gọi `GetCurrentAsync`, rồi gán dữ liệu đó vào period được yêu cầu. Redis quota tạo key theo tháng hiện tại và hết TTL đúng thời điểm bắt đầu tháng kế tiếp. Vì thế, lần snapshot chạy sau chuyển tháng có thể lấy counter tháng mới cho invoice tháng trước; comment về giữ counter cũ sau ranh giới tháng không khớp phép tính TTL. Đây là phát hiện tĩnh cần reproduce, chưa phải kết quả test. Xem [lịch job](https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/src/Modules/Billing/Modules.Billing/BillingModule.cs#L113-L121), [MonthlyInvoiceJob.cs:25–32](https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/src/Modules/Billing/Modules.Billing/Services/MonthlyInvoiceJob.cs#L25-L32), [UsageReporter.cs:43–60](https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/src/Modules/Billing/Modules.Billing/Services/UsageReporter.cs#L43-L60) và [RedisQuotaService.cs:190–216](https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/src/BuildingBlocks/Quota/RedisQuotaService.cs#L190-L216).

Quota mặc định còn đang tắt; nếu bật mà không cấu hình `QuotaOptions.Redis`, DI chọn bộ đếm in-memory theo process. Redis cho caching không tự chứng minh quota đã dùng Redis. Xem [appsettings.json:168–185](https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/src/Host/FSH.Starter.Api/appsettings.json#L168-L185) và [Quota Extensions.cs:25–54](https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/src/BuildingBlocks/Quota/Extensions.cs#L25-L54).

**Đề xuất MVP:** gói cố định theo thời hạn, với entitlement/giới hạn đã thỏa thuận và theo dõi mức sử dụng để vận hành. Chưa thu tiền theo API calls/storage overage. Nếu sau này dùng usage billing, cần dữ liệu đo có period tường minh, retention đủ cho backfill, snapshot bất biến, lịch sử thay đổi plan/rate, đối soát khi retry và test chuyển tháng/năm, job chạy muộn, mất Redis, thay plan giữa kỳ. Giá và giới hạn cụ thể chỉ chốt sau khảo sát giá trị khách hàng và chi phí phục vụ.

### 4.2. Việt hóa phải là hạng mục sản phẩm

Backend whitelist hiện chỉ có `en-US` và `pt-BR`; các nhãn menu, login và formatter ngày trên dashboard đang dùng tiếng Anh hoặc `en-US` trực tiếp. Khảo sát cây frontend ở commit này không thấy `src/i18n.ts` hoặc catalog locale như phần rules mô tả, nên không thể coi hướng dẫn i18n là bằng chứng tính năng đã hiện diện đầy đủ. Xem [SupportedCultures.cs:22–23](https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/src/BuildingBlocks/Core/Localization/SupportedCultures.cs#L22-L23), [menu](https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/clients/dashboard/src/components/layout/nav-data.ts#L55-L110), [login](https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/clients/dashboard/src/pages/login.tsx#L88-L126) và [formatters](https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/clients/dashboard/src/lib/list-helpers.ts#L3-L48).

Đề xuất xây catalog `vi-VN` cho hai app, API errors/validation, email và PDF thuộc MVP; format VND, số, ngày và múi giờ theo cấu hình đã chốt; kiểm tra tiếng Việt có dấu trong tìm kiếm/import/export. Lưu mã lỗi ổn định để UI xử lý, thay vì phụ thuộc câu thông báo tiếng Anh: dashboard hiện nhận diện tenant deactivated bằng `detail.toLowerCase().includes(...)`. Xem [api-client.ts:29–41](https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/clients/dashboard/src/lib/api-client.ts#L29-L41).

Đối với khách hàng SME, nên đổi nhãn kỹ thuật “Tenant” thành “Mã doanh nghiệp” hoặc giải quyết qua đường dẫn/lời mời; chọn phương án sau quan sát người dùng thử. Luồng operator và tenant administrator cần riêng biệt về quyền, tài khoản và sự rõ ràng trên giao diện. Việc menu ẩn không thay thế enforcement ở API.

### 4.3. Migration và readiness cần kiểm tra theo từng database/module

Migrator có lệnh áp dụng theo tenant và advisory lock. Tuy nhiên, `list-pending` chỉ liệt kê migration thật của tenant catalog; phần từng tenant chỉ in thông báo rồi `continue`. Vòng apply tuần tự có outer catch trả exit code 1 khi có lỗi. Đây là nền tảng hữu ích, nhưng chưa phải bằng chứng có cơ chế triển khai từng đợt, checkpoint, canary hoặc khả năng khôi phục một đợt nâng schema chưa hoàn tất. Xem [MigratorCommand.cs:7–10](https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/src/Host/FSH.Starter.DbMigrator/MigratorCommand.cs#L7-L10), [advisory lock](https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/src/Host/FSH.Starter.DbMigrator/PostgresMigratorLock.cs#L86-L114), [catalog preview](https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/src/Host/FSH.Starter.DbMigrator/Program.cs#L201-L210), [tenant loop](https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/src/Host/FSH.Starter.DbMigrator/Program.cs#L253-L287) và [failure exit](https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/src/Host/FSH.Starter.DbMigrator/Program.cs#L317-L327).

`TenantMigrationsHealthCheck` loop qua tenant nhưng resolve `TenantDbContext`; cần nghiệm thu rằng trạng thái module/database riêng được phản ánh đúng, không suy diễn từ tên check. Check cũng đưa tên, trạng thái và hạn tenant vào details, trong khi `/health/ready` là anonymous và trả details. Đề xuất tách probe tối thiểu dành cho load balancer khỏi màn hình chẩn đoán được bảo vệ. Xem [TenantMigrationsHealthCheck.cs:38–80](https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/src/Modules/Multitenancy/Modules.Multitenancy/TenantMigrationsHealthCheck.cs#L38-L80) và [HealthEndpoints.cs:15–59](https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/src/BuildingBlocks/Web/Health/HealthEndpoints.cs#L15-L59).

### 4.4. Backup, quan sát hệ thống và chất lượng phát hành

Docker Compose README hướng dẫn tar ba volume dữ liệu; đoạn backup không đưa quy trình tạo bản sao PostgreSQL nhất quán và diễn tập restore. Terraform production có cấu hình retention backup 30 ngày, Multi-AZ, deletion protection và versioning S3; các khai báo đó chưa chứng minh phục hồi từng tenant đã được diễn tập. Xem [backup Compose](https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/deploy/docker/README.md#L101-L112) và [production tfvars](https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/deploy/terraform/apps/starter/envs/prod/us-east-1/terraform.tfvars#L54-L89).

Cần xác định phạm vi phục hồi gồm database tenant, bản ghi tenant catalog, tệp đính kèm, khóa/secret và trạng thái integration jobs. Khôi phục dữ liệu cũ có thể phát lại sự kiện hoặc lệch đối soát với dữ liệu Billing ở database chính; runbook phải quy định cách tạm ngừng xử lý, đối chiếu và mở lại tenant. Đề xuất thử khôi phục tenant A sang database mới, kiểm tra dữ liệu/tệp, chuyển lại định tuyến sau xác nhận, đồng thời tenant B tiếp tục hoạt động. RPO/RTO phải đo trong diễn tập, không suy từ retention của backup.

Repo có OpenTelemetry, health endpoints và CloudWatch alarm definitions. Riêng Compose hiện chỉ map endpoint OTLP vào trường cấu hình; exporter trong production vẫn `Enabled=false`, còn auto-detect đọc biến môi trường chuẩn `OTEL_EXPORTER_OTLP_ENDPOINT`. Cần smoke test xác nhận traces/metrics đến collector trong topology chọn dùng. Xem [Compose:190](https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/deploy/docker/docker-compose.yml#L190), [production OTLP config](https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/src/Host/FSH.Starter.Api/appsettings.Production.json#L2-L16), [exporter condition](https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/src/BuildingBlocks/Web/Observability/OpenTelemetry/Extensions.cs#L54-L73) và [alarm definitions](https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/deploy/terraform/modules/cloudwatch_alarms/main.tf#L24-L94).

CI có build warnings-as-errors, unit/integration jobs, migrator container smoke và coverage gate cấu hình 80%. Frontend có lint/build, Playwright và container start/restart smoke. Đây là những kiểm soát có trong workflow, không phải số đo coverage hoặc kết quả chạy đã xác minh trong khảo sát. Playwright hiện mock API nên chưa bao phủ browser → API → tenant database thực. Xem [backend checks](https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/.github/workflows/backend.yml#L76-L104), [integration/migrator jobs](https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/.github/workflows/backend.yml#L123-L229), [coverage floor](https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/.github/workflows/backend.yml#L231-L240), [frontend checks](https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/.github/workflows/frontend.yml#L39-L156) và [Playwright scope](https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/clients/dashboard/playwright.config.ts#L3-L12).

## 5. Lộ trình theo điều kiện nghiệm thu

**Giả định nhân lực cho ước lượng:** 2 backend và 1 frontend làm toàn thời gian; QA và ops tham gia bán thời gian, nhưng có lịch cố định cho việc nghiệm thu nền tảng và pilot; có một người quyết định sản phẩm cùng đại diện khách hàng phản hồi trong vòng 1–2 ngày làm việc. Nhóm quen .NET/React/PostgreSQL, có sẵn môi trường phát triển và staging. Không bao gồm chuyển đổi dữ liệu legacy phức tạp, offline, payment gateway, kết nối ngân hàng, hoặc triển khai tuân thủ chuyên ngành.

Các khoảng thời gian sau là **ước lượng lập kế hoạch**, không phải cam kết tiến độ. Phase sau chỉ mở phần phụ thuộc khi gate phase trước được nghiệm thu. Có thể khảo sát UX trong lúc hoàn thiện nền tảng, nhưng không bỏ qua kiểm thử database riêng để chạy kịp demo nghiệp vụ.

| Phase | Thời lượng dự kiến với giả định trên | Kết quả cần giao | Điều kiện chuyển phase |
|---|---|---|---|
| 0 — Khảo sát và reproduce | **5–10 ngày kỹ thuật**, tương ứng khoảng 1–2 tuần nếu quyền truy cập sẵn sàng | Chốt 1–2 luồng nghiệp vụ chính; ADR tenant/company/branch; sơ đồ nơi lưu dữ liệu; danh sách rủi ro được reproduce; hồ sơ tải và RPO/RTO mong muốn | Mỗi vấn đề có bằng chứng, phạm vi, người xử lý; quyết định tiếp tục/thu hẹp/đổi nền tảng được ghi lại |
| 1 — Hoàn thiện database-per-tenant | Khoảng 3–5 tuần, cập nhật sau phase 0 | Provisioning, routing, auth, background jobs, migration, backup/restore, observability và deployment có runbook | Bộ acceptance trên ít nhất 2 database tenant riêng biệt, ngoài control DB, đạt; restore tenant được đo; không còn lỗi nghiêm trọng chưa xử lý trong phạm vi bảo vệ dữ liệu |
| 2 — MVP bán hàng, kho, công nợ | Khoảng 6–9 tuần | Luồng nghiệp vụ đã chốt, tiếng Việt, phân quyền, import/export, báo cáo và SaaS fixed tier | Browser-to-database scenarios thực; đối chiếu chứng từ/kho/công nợ; khách hàng xác nhận thao tác chính |
| 3 — Pilot vận hành | Khoảng 2–4 tuần | Pilot với nhóm khách hàng nhỏ, hướng dẫn, hỗ trợ, cảnh báo và diễn tập sự cố | Đạt tiêu chí vận hành và đối soát đã chốt; có chủ sở hữu vận hành, lịch backup/restore và quyết định mở rộng |

Tổng cửa sổ lập kế hoạch ban đầu khoảng **12–20 tuần**, chưa gồm hạng mục tùy chọn hoặc sự phụ thuộc bên ngoài. Nếu phase 0 phát hiện phải sửa nhiều BuildingBlocks, xử lý transaction xuyên database, hoặc tách lại background infrastructure, cần ước lượng lại phase 1 trước khi đưa ngày bàn giao. Repo còn yêu cầu phê duyệt khi sửa `src/BuildingBlocks`; đưa điều kiện này vào kế hoạch thay đổi, không sửa âm thầm. Xem [AGENTS.md — golden rules](https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/AGENTS.md#L84-L87).

### Backlog A — Khảo sát và điều kiện nền tảng

Tất cả các dòng dưới đây là công việc đề xuất, chưa đánh dấu hoàn thành.

| ID | Công việc / phụ trách chính | Phụ thuộc | Điều kiện nghiệm thu |
|---|---|---|---|
| A01 | Khảo sát khách hàng, chốt mô hình tổ chức / product + backend | Đại diện SME, ví dụ chứng từ thật đã loại bỏ thông tin nhạy cảm | Có luồng bán/đổi trả/thu tiền, ma trận vai trò, định nghĩa tồn và công nợ; quyết định một company/tenant được xác nhận |
| A02 | Reproduce các finding tại commit cố định / backend + QA | Staging có Docker/PostgreSQL/Redis/S3 compatible | Có lệnh chạy, dữ liệu mẫu, expected/actual và bằng chứng; phân biệt lỗi tái hiện được với giả thuyết cần kiểm tra |
| A03 | Kiểm kê nơi lưu dữ liệu và hợp đồng module / backend | A02 | Mỗi DbContext, cache key, blob prefix, job và outbox có chủ sở hữu, tenant context và đường phục hồi; xác định rõ dữ liệu control plane dùng chung |
| A04 | Nghiệm thu isolation database riêng / backend + QA | A03 | Hai tenant có hai database khác nhau và dữ liệu canary; read/write, raw SQL, bulk, file, job, search/export và realtime không chạm dữ liệu tenant khác; thiếu tenant context phải thất bại rõ |
| A05 | Provisioning và vòng đời tenant / backend + ops | A03–A04 | Tạo/retry sau process restart không tạo admin hoặc DB trùng; lỗi từng bước được theo dõi; chỉ activate khi đủ điều kiện; suspend/reactivate và offboarding có hành vi được chốt |
| A06 | Migration theo từng đợt / backend + ops | A03–A05 | Preview liệt kê đúng từng module/database; canary rồi batch; ghi trạng thái từng tenant; chạy lại idempotent; mô phỏng một tenant lỗi; có chiến lược tương thích schema khi rollout dở |
| A07 | Backup/restore và bí mật / ops + backend | A03 | Khôi phục A vào database mới, kiểm dữ liệu/tệp và liên kết control plane; B tiếp tục phục vụ; đo RPO/RTO; kiểm tra key/secret cần thiết; ghi runbook và người chịu trách nhiệm |
| A08 | Ranh giới operator/tenant và chẩn đoán / backend + frontend + QA | A04 | Ma trận API cho operator/tenant admin/member được kiểm; support access có audit; public health không trả danh sách tenant; đường dẫn thiếu quyền có UX rõ |
| A09 | Telemetry và phát hành / ops + QA | A05–A08 | Request nghiệp vụ thấy trace/log tương ứng; cảnh báo provisioning/migration/job thất bại được nhận; deploy rehearsal có smoke thực; rollback ứng dụng phù hợp schema đã triển khai |

### Backlog B — Nghiệp vụ SME

| ID | Công việc / phụ trách chính | Phụ thuộc | Điều kiện nghiệm thu |
|---|---|---|---|
| B01 | Company, branch và quyền nghiệp vụ / backend + frontend | A01, A04, A08 | Người dùng chỉ thao tác phạm vi được cấp; khóa tổ chức theo chứng từ; không dùng header tự chọn để vượt quyền |
| B02 | Danh mục và import/export / backend + frontend | B01 | SKU/customer key có quy tắc duy nhất theo phạm vi; import báo lỗi theo dòng; chạy lại không tạo bản sao ngoài ý muốn; export tôn trọng quyền và tenant |
| B03 | Đơn bán và trạng thái chứng từ / backend + frontend | B01–B02 | Giá/thuế/discount được snapshot; post/cancel có quyền; hai lần gửi không tạo hai chứng từ; cạnh tranh cập nhật không âm thầm ghi đè |
| B04 | Sổ kho và xuất/nhập/đảo / backend + QA | B03 và quyết định transaction/outbox ở A03 | Tổng ledger bằng tồn hiển thị; đồng thời xuất không vi phạm chính sách tồn; retry không nhân đôi chuyển động; trả hàng/đảo giữ lịch sử |
| B05 | Công nợ và thu tiền / backend + frontend + chuyên môn nghiệp vụ | B03–B04 | Công nợ = phát sinh đã ghi nhận trừ thu/phân bổ/điều chỉnh hợp lệ; thu một phần và thu lại sau retry đúng; đảo chứng từ có thể đối chiếu |
| B06 | Tiếng Việt và trải nghiệm / frontend + QA | A08, B01–B05 | Luồng chính, lỗi, email/PDF thuộc MVP có tiếng Việt; VND, ngày, múi giờ nhất quán; dữ liệu có dấu không hỏng; bàn phím và màn hình nhỏ dùng được |
| B07 | Gói SaaS cố định / backend + operator | A05, A08 | Giá/thời hạn/entitlement rõ ràng; gia hạn và đối soát thủ công có audit; không sinh phí overage từ counter chưa được nghiệm thu; invoice SaaS khác chứng từ bán hàng |
| B08 | Báo cáo và đối chiếu / backend + QA + khách hàng | B03–B05 | Một bộ dữ liệu mẫu có đáp án đã được xác nhận; doanh số, tồn và công nợ khớp; filter theo branch/period đúng; export có thể truy ngược chứng từ |
| B09 | Adapter VAT/hóa đơn điện tử nếu bắt buộc / backend + nhà cung cấp + chuyên môn thuế | A01, B03 và quyết định scope riêng | Sandbox contract tests; retry không phát hành trùng; lưu request/result và trạng thái đối soát; kịch bản điều chỉnh/thay thế được chuyên môn duyệt |

### Backlog C — Pilot và quyết định mở rộng

| ID | Công việc / phụ trách chính | Phụ thuộc | Điều kiện nghiệm thu |
|---|---|---|---|
| C01 | Browser-to-database suite / QA + backend + frontend | Gate phase 1, B01–B08 | Dùng browser thật và backend thật cho login, quyền, tenant A/B, bán/xuất/thu, retry và expire session; không thay toàn bộ API bằng mocks |
| C02 | Diễn tập tải và sự cố / ops + QA | A09, C01 | Tải theo hồ sơ phase 0; đo latency/error/connection pool và ảnh hưởng tenant lớn; kiểm mất Redis/storage/job restart; cảnh báo và runbook sử dụng được |
| C03 | Pilot có giới hạn / product + support + QA | C01–C02 | Đề xuất 3–5 tenant đầu tiên nếu khách hàng phù hợp; nhập số dư có đối chiếu; người dùng hoàn thành luồng chính; lỗi được phân loại và xử lý có chủ sở hữu |
| C04 | Quyết định mở rộng / product + kỹ thuật + ops | C03 và ít nhất một restore rehearsal sau dữ liệu pilot | Không còn sai lệch dữ liệu nghiêm trọng; RPO/RTO và mục tiêu dịch vụ được chấp nhận; có báo cáo chi phí/tenant, lịch bảo trì, capacity và hỗ trợ |

## 6. TCO và quyết định vận hành

Không nên chốt giá cloud hoặc giá bán chỉ từ cấu hình mẫu. TCO cần bao gồm hạ tầng, công phát triển/bảo trì, QA, vận hành/on-call, hỗ trợ khách hàng, lưu trữ/backup/log và chi phí tích hợp/chuyên môn nghiệp vụ. Chi phí theo tenant cần tính thêm thời gian onboarding, migration, restore, hỗ trợ và xử lý dữ liệu ngoài chi phí database.

| Nguồn chi phí/rủi ro | Ảnh hưởng của database-per-tenant | Cách kiểm soát đề xuất |
|---|---|---|
| Connections và tài nguyên PostgreSQL | Nhiều database có thể làm tăng số pool/connection và công giám sát; database riêng vẫn chia sẻ CPU/I/O nếu cùng cụm | Đo tải thật, giới hạn pool và concurrency; theo dõi theo tenant; có phương án di chuyển tenant lớn |
| Migration và patching | Mỗi đợt phát hành cần theo dõi schema trên toàn bộ tenant | Registry version, canary/batch, timeout và checkpoint; không chỉ chạy một lệnh rồi suy tất cả tenant cùng version |
| Backup và restore | Cách phục hồi tenant phải phối hợp DB, tệp và control plane | Tự động hóa kiểm kê/backup; diễn tập restore; định nghĩa retention và chi phí lưu trữ theo gói |
| Báo cáo xuyên tenant | Không có một query đơn giản trên mọi database cho thống kê nền tảng | Dùng aggregate/snapshot được cho phép; đánh giá độ trễ; tránh mở quyền query tùy ý tới database tenant |
| Hai frontend | Hai bộ route, quyền, formatter, i18n và tests cần bảo trì | Chia sẻ contract/primitives có chủ đích; giữ rõ luồng operator và khách hàng; đo lợi ích trước khi hợp nhất app |
| Quyền sở hữu source | Tùy biến sâu làm tăng chi phí nhận bản vá/upstream | ADR cho thay đổi framework, khóa baseline, lịch nâng dependency và regression tests theo rủi ro |
| Dịch vụ mở rộng | Chat, webhooks, payment gateway, e-invoice, đa kho tăng test matrix và runbooks | Chỉ mở khi một luồng giá trị của pilot yêu cầu và có người vận hành |

Mẫu Terraform production hiện có ba AZ, NAT theo AZ, ba API task tối thiểu, RDS Multi-AZ và hai Redis node. Đây là cấu hình tham khảo có nhiều thành phần; không nên dùng nguyên trạng làm baseline chi phí cho vài SME pilot nếu mục tiêu dịch vụ chưa yêu cầu. Ngược lại, Compose một host có miền lỗi tập trung và README nói rõ cần tự cung cấp reverse proxy/TLS. Chọn topology sau khi chốt SLA nội bộ, RPO/RTO và khả năng vận hành; ưu tiên managed PostgreSQL/object storage nếu nhóm chưa có năng lực trực vận hành các thành phần đó. Xem [production topology](https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/deploy/terraform/apps/starter/envs/prod/us-east-1/terraform.tfvars#L14-L37), [database/Redis/API sizing](https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/deploy/terraform/apps/starter/envs/prod/us-east-1/terraform.tfvars#L79-L127) và [Compose deployment boundary](https://github.com/chung6a8m/dotnet-starter-kit/blob/a967537526f3b769c7d268da3ed2a6920af65ce9/deploy/docker/README.md#L3-L16).

Trước khi báo giá, cần điền một mô hình chi phí theo tháng gồm: chi phí nền tảng cố định; biến phí theo tenant hoạt động, dung lượng và egress; giờ support/ops; thời gian release/restore; môi trường staging; dự phòng tăng tải và sự cố. Mức dự phòng và ngân sách chấp nhận do nhóm chốt sau phase 0, không suy ra từ nhãn “production-ready” của starter.

## 7. Quyết định cần chốt sau phase 0

- Nhóm SME và luồng nghiệp vụ đầu tiên; service-only hay thương mại có kho; một hay nhiều chi nhánh.
- Tenant là hợp đồng khách hàng hay pháp nhân; có yêu cầu di chuyển/xuất/xóa/khôi phục riêng từng company không.
- Mức tách biệt database mong muốn: cùng cụm, role riêng, instance riêng cho nhóm đặc biệt; quyền DDL/runtime và người được truy cập dữ liệu.
- Số tenant/người dùng đồng thời, kích thước dữ liệu/tệp, thời gian lưu, RPO/RTO và giờ hỗ trợ.
- Kho và công nợ cần đến đâu; thuế/hóa đơn điện tử có nằm trên đường bắt buộc trước pilot không.
- Gói SaaS cố định, cách đối soát thu tiền, entitlement và hành vi khi hết hạn; usage billing tiếp tục ở backlog riêng.
- Những finding nào đã reproduce, trách nhiệm sửa thuộc module hay BuildingBlocks, và ước lượng cập nhật cho phase 1.

Mốc quyết định đầu tiên là kết thúc phase 0 với bằng chứng và phạm vi được chốt. Mốc mở nghiệp vụ thật là hoàn thành gate database-per-tenant và phục hồi dữ liệu ở phase 1; lịch nghiệp vụ phải đi theo các điều kiện đó.
