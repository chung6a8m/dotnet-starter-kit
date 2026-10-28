# Ý tưởng xây dựng ứng dụng SME theo mô hình Database-per-Tenant

- **Ngày đánh giá:** 28/10/2026.
- **Repository:** [chung6a8m/dotnet-starter-kit](https://github.com/chung6a8m/dotnet-starter-kit).
- **Mốc mã nguồn:** [a967537526f3b769c7d268da3ed2a6920af65ce9](https://github.com/chung6a8m/dotnet-starter-kit/commit/a967537526f3b769c7d268da3ed2a6920af65ce9), nhánh `main` tại thời điểm khảo sát.
- **Trạng thái:** phân tích và đề xuất để ra quyết định; chưa phải thiết kế đã được chủ sản phẩm phê duyệt.
- **Phạm vi thay đổi:** tài liệu trong `docs/sme/ideas`. Các giải pháp, bài kiểm thử và lộ trình dưới đây chưa được triển khai.

## 1. Kết luận

**Nên dùng repo này làm nền, với điều kiện hoàn thiện và kiểm chứng các luồng Database-per-Tenant trước khi đưa dữ liệu SME thật vào hệ thống.** Repo đã có modular monolith, phân ranh giới module, định tuyến database theo tenant, identity, provisioning, migration runner, eventing, audit, file storage và hai giao diện quản trị. Đây là lượng nền tảng đáng tái sử dụng.

Điều kiện quyết định không nằm ở việc thêm một trường connection string. Một sản phẩm Database-per-Tenant phải giữ đúng database trong cả HTTP request, background job, event consumer, hỗ trợ khách hàng, migration, backup và restore. Khảo sát mã nguồn cho thấy một số đường đi này còn chưa nhất quán; chi tiết, bằng chứng và giới hạn kết luận nằm trong [đánh giá repo](01-danh-gia-repo.md).

**Kiến trúc khuyến nghị:** giữ `.NET 10 + PostgreSQL + modular monolith`, chia hệ thống thành:

- **Control plane:** quản trị tenant, vị trí database, tiến trình provisioning, gói thuê bao/billing SaaS, danh tính vận hành nền tảng.
- **Data plane:** mỗi tenant có một database chứa identity của tenant, dữ liệu nghiệp vụ, metadata tệp, audit nghiệp vụ, outbox/inbox; các module dùng schema riêng trong database đó.
- **Hạ tầng dùng chung có phân vùng logic:** API/worker, cache, hàng đợi và object storage. Database riêng không mặc nhiên tạo compute, bộ nhớ, đường truyền hoặc quyền quản trị riêng.

Đây là kiến trúc đề xuất dựa trên hiện trạng. Repo hiện cho phép cả database chung và database riêng; `TenantDbContext` và `BillingDbContext` đã thuộc database chính. Cần bổ sung chính sách bắt buộc database riêng cho tenant khách hàng và xử lý rõ ngoại lệ control plane. [Bằng chứng](05-bang-chung-va-nguon-tham-khao.md)

## 2. Đọc bộ tài liệu theo nhu cầu

| Tài liệu | Nội dung chính | Dùng để quyết định |
|---|---|---|
| [01-danh-gia-repo.md](01-danh-gia-repo.md) | Kiến trúc hiện có, điểm mạnh, khoảng trống và các phát hiện từ source/test | Repo phù hợp đến đâu; phần nào cần sửa trước |
| [02-kien-truc-de-xuat.md](02-kien-truc-de-xuat.md) | Các phương án, control/data plane, tenant context, identity, module và transaction | Chọn kiến trúc đích và ranh giới dữ liệu |
| [03-van-hanh-va-an-toan-du-lieu.md](03-van-hanh-va-an-toan-du-lieu.md) | Provisioning, migration fleet, restore, chuyển database, secrets, capacity, acceptance tests | Điều kiện an toàn để chạy pilot/production |
| [04-pham-vi-sme-va-lo-trinh.md](04-pham-vi-sme-va-lo-trinh.md) | Phạm vi sản phẩm SME, module tái sử dụng, backlog, giai đoạn và ước lượng | Xây chức năng gì trước, ai làm, khi nào nghiệm thu |
| [05-bang-chung-va-nguon-tham-khao.md](05-bang-chung-va-nguon-tham-khao.md) | Mốc khảo sát, danh mục source và tài liệu chính thức | Kiểm chứng nhận định và cập nhật khi upstream thay đổi |

## 3. Các quyết định được đề xuất

| Chủ đề | Khuyến nghị | Lý do và điều kiện |
|---|---|---|
| Kiến trúc ứng dụng | Modular monolith; API và worker có thể tách process khi cần | Tận dụng cấu trúc hiện có, giảm gánh nặng vận hành; module vẫn có Contracts và ownership rõ |
| Database | PostgreSQL; một database cho mỗi tenant khách hàng | Đây là đường triển khai, migrations và nhiều tối ưu đã hiện hữu trong repo |
| Tách module | Schema theo module trong cùng tenant database | Giữ giao dịch nghiệp vụ nội bộ tenant khả thi, không nhân số database theo số module |
| Tenant và doanh nghiệp | Tenant là ranh giới thuê bao/dữ liệu; công ty, chi nhánh, kho là các phạm vi nghiệp vụ bên trong | Không dùng tenant để thay thế mọi khái niệm tổ chức |
| Identity MVP | Giữ identity theo tenant, operator ở root/control plane | Giảm thay đổi nền; nếu một tài khoản phải tham gia nhiều tenant thì phải thiết kế global identity trước |
| Billing | Tính phí thuê bao SaaS theo gói cố định ở MVP | Không dùng module Billing làm kế toán bán hàng; usage billing cần sửa/kiểm chứng dữ liệu theo kỳ |
| Hạ tầng ban đầu | Một cụm triển khai có giới hạn tải rõ; có khả năng thêm cụm và chuyển tenant | Không cần microservices/Kubernetes chỉ vì có multitenancy |
| Tùy biến | Cấu hình, feature flags, custom fields có kiểm soát | Giữ một dòng schema và code; hạn chế phân nhánh riêng cho từng khách hàng |
| Phương thức phát triển | Fork có chủ đích, tiếp nhận upstream chọn lọc | Sở hữu source đồng nghĩa chịu trách nhiệm nâng cấp, kiểm thử và vận hành |

Các hàng trên là **đề xuất**, không phải yêu cầu người dùng đã chốt. Đặc biệt, ngành SME, quy mô tenant, ngân sách vận hành, nhu cầu SSO và yêu cầu SQL Server vẫn cần xác nhận trước giai đoạn triển khai.

## 4. Những việc phải giải quyết trước pilot có dữ liệu thật

1. **Định tuyến end-to-end:** không để tenant request/event/job mất thông tin database và rơi về default database; kiểm cả inbox và side effect.
2. **Migration quan sát đúng mục tiêu:** kiểm từng module trong đúng database; catalog khỏe không được coi là tenant database khỏe.
3. **Provisioning đáng tin cậy:** tenant chỉ sẵn sàng sau khi hoàn tất; retry/restart/multiple workers không mất thông tin khởi tạo hoặc tạo trạng thái treo.
4. **Quyền truy cập và secrets:** runtime ít quyền, migrator/provisioner có quyền riêng; không trả connection string cho giao diện quản trị; phân biệt actor và tenant đang được hỗ trợ.
5. **Khôi phục có thực nghiệm:** restore riêng một tenant, kiểm dữ liệu, tệp, identity, outbox và ảnh hưởng tenant khác.
6. **Nghiệp vụ nhất quán:** tồn kho, công nợ, số chứng từ và quyền theo chi nhánh phải có quy tắc và kiểm thử độc lập; các module mẫu chưa thay thế những quy tắc này.

Các hạng mục có mức ưu tiên và tiêu chí nghiệm thu trong tài liệu 01, 03 và 04. Chỉ số hiệu năng, số tenant phục vụ được và SLA phải được đo trên cấu hình dự kiến.

## 5. Khi nào nên xem lại lựa chọn base?

Cần đánh giá lại chi phí dùng repo nếu có một trong các ràng buộc sau:

- Bắt buộc `net8.0` hoặc chỉ giữ toolchain Visual Studio 2022 cho sản phẩm mới, không có kế hoạch nâng cấp.
- Bắt buộc SQL Server và cần toàn bộ tính năng hiện tại hoạt động ngay mà không có ngân sách port/kiểm thử.
- Cần global SSO, một người dùng nhiều tenant, hoặc triển khai riêng/on-premise cho mọi khách hàng ngay từ đầu.
- Sản phẩm trọng tâm là kế toán đầy đủ, sản xuất/MRP, POS offline; phạm vi này lớn hơn nền tảng SaaS và các module mẫu.
- Đội ngũ chưa có khả năng quản trị nhiều database, xử lý migration thất bại và kiểm thử restore.

Những ràng buộc này không tự động khiến repo không dùng được; chúng thay đổi đáng kể chi phí và thứ tự triển khai. .NET 10 là LTS; yêu cầu SDK/Visual Studio cần theo ma trận chính thức, không suy từ thói quen dùng các repo .NET 8 trước đây. [Nguồn chính thức](05-bang-chung-va-nguon-tham-khao.md#nguon-ben-ngoai)

## 6. Mức độ chắc chắn và giới hạn

Khảo sát dựa trên đọc mã nguồn, cấu hình, CI và các test liên quan tại commit đã nêu; các nguồn bên ngoài được kiểm tra ngày 28/10/2026. Chưa build/chạy ứng dụng, chưa chạy bộ test của sản phẩm, chưa thực hiện load test hay thử backup/restore trong đợt viết tài liệu này.

Vì vậy:

- “Có triển khai” nghĩa là đã thấy code tương ứng.
- “Có test” nghĩa là đã đọc test tương ứng, không có nghĩa đã chạy và xác nhận pass.
- Phát hiện suy ra từ luồng code được ghi tách với rủi ro cần tái hiện.
- Mọi mốc thời gian, capacity và RPO/RTO đề xuất là đầu vào lập kế hoạch, chưa phải cam kết.

**Bước triển khai đầu tiên được đề xuất:** một giai đoạn 5–10 ngày làm việc để tái hiện các điểm rủi ro trên control database + hai tenant database riêng, chốt identity/database engine/phạm vi SME, rồi ước lượng lại giai đoạn hoàn thiện nền tảng. Đây là việc tiếp theo cần quyết định, chưa nằm trong commit tài liệu.
