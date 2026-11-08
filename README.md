# THE PETAL ATELIER

Website thương mại điện tử hoa tươi bằng Next.js App Router + TypeScript + Supabase PostgreSQL/Auth/Storage. Giao diện tiếng Việt, tiền tệ VND, phong cách xanh lá/kem/hồng phấn.

## Trạng thái hiện tại

- Trang chủ, danh mục, trang sản phẩm, giỏ hàng lưu ở localStorage, checkout COD, tài khoản Supabase Auth, tra cứu đơn hàng, liên hệ, blog mẫu.
- 12 sản phẩm demo có nhãn dữ liệu minh họa.
- SQL migration có bảng dữ liệu cốt lõi, RLS, RPC tạo đơn COD transaction-safe và RPC tra cứu tối thiểu theo mã đơn + số điện thoại.
- API tạo đơn **không** tin giá từ trình duyệt; RPC lấy giá/tồn kho hiện tại trong database, khóa biến thể, giảm tồn kho và ghi snapshot trong cùng transaction.
- Đơn COD được tạo ở trạng thái `pending_confirmation`, thanh toán `unpaid`. Không có cổng thanh toán hay giả lập thanh toán thành công.
- Các trang admin hiện là khung khởi tạo và danh sách đọc bảo vệ role; CRUD đầy đủ, thay đổi trạng thái, quản lý coupon, biểu đồ doanh thu và upload ảnh cần được hoàn thiện trước vận hành thực tế.
- Frontend catalog ban đầu dùng dữ liệu demo tĩnh. Chỉ hiển thị catalog từ Supabase sau khi nạp sản phẩm thật đã xác minh.

## Yêu cầu

- Node.js 20+
- Một dự án Supabase
- npm

## Chạy cục bộ

1. Giải nén dự án, mở thư mục `the-petal-atelier`.
2. Cài dependencies:

   ```bash
   npm install
   ```
3. Sao chép `.env.example` thành `.env.local`, điền:
   - `NEXT_PUBLIC_SUPABASE_URL`
   - `NEXT_PUBLIC_SUPABASE_ANON_KEY`
   - `NEXT_PUBLIC_SITE_URL=http://localhost:3000`
4. Trong Supabase SQL Editor, chạy theo thứ tự:
   - `supabase/migrations/0001_initial_schema.sql`
   - `supabase/seed.sql`
5. Trong Supabase Authentication, bật email/password, cấu hình Site URL và redirect URL `http://localhost:3000/auth/callback`.
6. Chạy:

   ```bash
   npm run dev
   ```
7. Mở `http://localhost:3000`.

Nếu chưa có Supabase config, trang sản phẩm demo và giỏ hàng phía client vẫn có thể xem/thao tác; checkout sẽ báo rõ chưa cấu hình và không tạo đơn giả.

## Tạo tài khoản admin

Không cho phép người dùng tự đặt vai trò admin trong metadata. Sau khi đăng ký tài khoản quản trị, dùng Supabase SQL Editor với UID tài khoản đã xác minh:

```sql
update public.profiles set role = 'admin' where id = 'AUTH_USER_UUID';
```

Role được kiểm tra trong RLS qua `profiles`. Trang admin đọc `app_metadata.role` hiện tại là một lớp kiểm tra giao diện bổ sung; để đồng bộ, nên triển khai quy trình quản trị tin cậy cập nhật cả `profiles.role` và `auth.users.raw_app_meta_data.role` bằng Supabase Dashboard/Edge Function chỉ dành cho chủ dự án. Không cung cấp quyền service-role cho trình duyệt.

## Lưu trữ ảnh

Tạo bucket Storage tên `product-images`, đặt public nếu ảnh sản phẩm là công khai. Thêm policy chỉ cho `is_staff()` upload/update/delete; không mở quyền ghi công khai. Các URL Unsplash hiện tại chỉ là ảnh tham khảo, cần thay bằng ảnh cửa hàng có quyền sử dụng.

## Lưu ý quan trọng trước khi bán thật

1. Thay 12 sản phẩm tĩnh trong `src/lib/products.ts` bằng truy vấn server-side tới `products`, `product_images`, `product_variants`.
2. Thay ID demo dạng `p01` bằng UUID sản phẩm và variant ID trong database. RPC checkout chỉ chấp nhận ID có trong database và trạng thái active.
3. Seed sản phẩm thật qua admin/SQL có kiểm soát; đặt `status='active'`, tạo variant với `stock_quantity` và bản ghi `inventory`.
4. Đơn được đánh dấu `is_demo=true` để ngăn nhầm lẫn với đơn kinh doanh; trước khi vận hành, điều chỉnh RPC để đánh dấu theo cấu hình môi trường và thêm quy trình xử lý đơn thật.
5. Rà soát RPC `create_cod_order`: bổ sung giới hạn ngày giao, chống spam/rate limit, phí giao hàng theo khu vực, mã giảm giá, xử lý variant trùng lặp trong payload và chính sách hủy/hoàn tồn.
6. Đối với tracking, dùng rate limiting ở middleware/WAF và có thể thêm OTP. Không hiển thị địa chỉ hoặc thông tin liên hệ qua API tra cứu công khai.
7. Tạo trang admin CRUD, upload ảnh Storage, cập nhật trạng thái qua RPC chuyển trạng thái hợp lệ và ghi `order_status_history`.
8. Thêm kiểm thử đơn vị/tích hợp, kiểm thử RLS, kiểm thử cạnh tranh tồn kho, accessibility và performance trước production.
9. Cập nhật thông tin liên hệ, chính sách, giờ giao và các cam kết thực tế.
10. Chạy `npm run build` và xử lý hết cảnh báo/lỗi trước deploy.

## Deploy

- Đưa source lên Git.
- Import repository vào Vercel.
- Khai báo environment variables ở Vercel Project Settings.
- Cấu hình URL production trong Supabase Auth redirect allow list và `NEXT_PUBLIC_SITE_URL`.
- Chạy migration qua Supabase CLI/CI có review; không chạy seed demo lên production.
- Dùng domain chính thức, kiểm tra sitemap/robots, logs, backups, RLS và quy trình sao lưu.

## Bảo mật và giới hạn hiện tại

Đây là bản khởi tạo mã nguồn thực dụng, không phải xác nhận hệ thống đã được kiểm thử production. Tài khoản Supabase, khóa, email gửi, ảnh chính thức, địa chỉ cửa hàng và thông tin vận chuyển chưa được cung cấp nên chưa thể xác nhận kết nối dịch vụ thật. Không đặt `SUPABASE_SERVICE_ROLE_KEY` ở client hoặc biến `NEXT_PUBLIC_*`.
