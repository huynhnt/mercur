# Hướng dẫn Deploy dự án Mercur (MedusaJS v2 Monorepo) lên aaPanel

Tài liệu này hướng dẫn chi tiết cách triển khai (deploy) dự án **Mercur** từ kho lưu trữ GitHub lên máy chủ cài đặt **aaPanel** với hệ thống tên miền của bạn:
- Backend API: `https://s1.huylio.com`
- Admin Panel: `https://admin.huylio.com`
- Vendor Panel: `https://vendor.huylio.com`

*Lưu ý: Hướng dẫn này tối ưu việc sử dụng giao diện đồ họa **Node Project** và các tính năng của aaPanel để giảm thiểu việc gõ lệnh trên terminal.*

---

## 1. Yêu cầu hệ thống và Chuẩn bị trên aaPanel

### Bước 1.1: Cài đặt các phần mềm từ App Store của aaPanel
Truy cập vào giao diện quản trị aaPanel -> **App Store** và cài đặt các phần mềm sau:
1. **Nginx** (Phiên bản khuyến nghị: `Nginx 1.22` hoặc `1.24`) - Dùng làm Web Server.
2. **Redis** (Phiên bản khuyến nghị: `Redis 7.x` hoặc mới hơn) - Cần thiết cho Event Bus và Caching của MedusaJS v2.
3. **Node.js Version Manager** - Quản lý phiên bản Node.js.
   - Sau khi cài đặt, mở ứng dụng lên và cài đặt phiên bản Node.js **v20.x** (hoặc v22.x).
   - Chọn phiên bản Node.js vừa cài đặt làm **Registry/CLI version** (để sử dụng lệnh `node` và `npm` ngoài terminal).

---

### Bước 1.2: Cài đặt và cấu hình PostgreSQL
MedusaJS v2 sử dụng PostgreSQL làm cơ sở dữ liệu mặc định. Bạn cài đặt PostgreSQL trực tiếp thông qua Terminal (SSH):

1. **Kết nối SSH vào VPS/Server** của bạn.
2. **Cài đặt PostgreSQL** (ví dụ trên hệ điều hành Ubuntu/Debian):
   ```bash
   sudo apt update
   sudo apt install -y postgresql postgresql-contrib
   ```
3. **Kích hoạt và khởi động PostgreSQL**:
   ```bash
   sudo systemctl start postgresql
   sudo systemctl enable postgresql
   ```
4. **Tạo Database và User cho Mercur**:
   Truy cập PostgreSQL CLI:
   ```bash
   sudo -u postgres psql
   ```
   Chạy các câu lệnh SQL sau (hãy thay đổi mật khẩu bằng mật khẩu bảo mật của bạn):
   ```sql
   CREATE DATABASE mercur;
   CREATE USER mercur_user WITH PASSWORD 'MatKhauBaoMatCuaBan';
   GRANT ALL PRIVILEGES ON DATABASE mercur TO mercur_user;
   \q
   ```
   *Lưu ý:* URL kết nối Database của bạn sẽ là: `postgres://mercur_user:MatKhauBaoMatCuaBan@127.0.0.1:5432/mercur`.

---

## 2. Tải Code và Cấu hình Môi trường

### Bước 2.1: Clone dự án từ GitHub
1. Mở Terminal trên aaPanel hoặc kết nối SSH.
2. Di chuyển tới thư mục chứa các website:
   ```bash
   cd /www/wwwroot
   ```
3. Clone mã nguồn dự án từ GitHub của bạn:
   ```bash
   git clone <LINK_GITHUB_CUA_BAN> mercur
   cd mercur
   ```

### Bước 2.2: Cài đặt Bun (Package Manager)
Dự án được cấu hình sử dụng Bun để quản lý gói (như đã định nghĩa trong `bun.lock` và `package.json`). Cài đặt Bun giúp quá trình build nhanh hơn và tránh lỗi xung đột:
```bash
curl -fsSL https://bun.sh/install | bash
# Cập nhật biến môi trường cho phiên terminal hiện tại
source ~/.bashrc
```

### Bước 2.3: Cài đặt dependencies
Tại thư mục gốc `/www/wwwroot/mercur`, chạy lệnh:
```bash
bun install
```

### Bước 2.4: Tạo file môi trường `.env` cho Backend API
Tạo file `.env` cho thư mục Backend tại `apps/api/.env`:
```bash
nano apps/api/.env
```
Nhập cấu hình mẫu bên dưới (đã điền sẵn tên miền của bạn):

```ini
# Cấu hình Database PostgreSQL
DATABASE_URL=postgres://mercur_user:MatKhauBaoMatCuaBan@127.0.0.1:5432/mercur

# Cấu hình Redis
REDIS_URL=redis://127.0.0.1:6379

# Cấu hình CORS
# Tên miền cho Storefront (Thay đổi nếu bạn có storefront riêng)
STORE_CORS=https://huylio.com,https://www.huylio.com

# Tên miền cho Admin Panel
ADMIN_CORS=https://admin.huylio.com

# Tên miền cho Vendor Panel (Dashboard của Người bán)
VENDOR_CORS=https://vendor.huylio.com

# Auth CORS (Gộp danh sách các địa chỉ được phép xác thực)
AUTH_CORS=https://admin.huylio.com,https://vendor.huylio.com,https://huylio.com,https://www.huylio.com

# Khóa bảo mật JWT và Cookie (Nhập các chuỗi ngẫu nhiên dài và bảo mật)
JWT_SECRET=supersecretjwtkeythatisverylongandsecurehuylio
COOKIE_SECRET=supersecretcookiekeythatisverylongandsecurehuylio
```
*Nhấn `Ctrl + O` -> `Enter` để lưu và `Ctrl + X` để thoát editor nano.*

---

## 3. Build Dự Án và Run Migration

### Bước 3.1: Build các Packages & Frontend Panels
Từ thư mục gốc `/www/wwwroot/mercur`, trước khi build, hãy thiết lập biến môi trường chỉ hướng API về cho 2 website Admin & Vendor nhận diện:
```bash
export VITE_MERCUR_BACKEND_URL=https://s1.huylio.com
export VITE_MERCUR_VENDOR_URL=https://vendor.huylio.com
```

Sau đó chạy lệnh để build toàn bộ các package dùng chung và biên dịch 2 trang Admin/Vendor:
```bash
bun run build
```
Lệnh này sẽ biên dịch mã nguồn của 2 trang Admin/Vendor và xuất ra thư mục tĩnh tại:
- Admin Panel: `/www/wwwroot/mercur/apps/admin-test/dist`
- Vendor Panel: `/www/wwwroot/mercur/apps/vendor/dist`

### Bước 3.2: Chạy Database Migration
Để tạo các bảng cơ sở dữ liệu cho MedusaJS và Mercur, di chuyển vào thư mục ứng dụng API và chạy migration:
```bash
cd apps/api
bunx medusa db:migrate
```

### Bước 3.3: Seed Dữ liệu mẫu (Tùy chọn)
Nếu bạn muốn có sẵn các sản phẩm mẫu, danh mục, cấu hình ban đầu để kiểm thử hệ thống:
```bash
bun run seed
```

---

## 4. Triển khai Backend API bằng giao diện "Node Project" trên aaPanel

Giao diện **Node Project** trên aaPanel sẽ tự động tạo cấu hình khởi chạy PM2 và tự động tạo Reverse Proxy Nginx cho bạn.

1. Truy cập vào aaPanel -> **Website** -> Chọn tab **Node project**.
2. Nhấn nút **Add Node Project** và điền các thông tin cấu hình như sau:
   - **Project Path**: Click biểu tượng thư mục và chọn đường dẫn đến thư mục chứa app API:
     `/www/wwwroot/mercur/apps/api`
   - **Node.js Version**: Chọn phiên bản Node.js bạn đã cài đặt (Node v20.x).
   - **Run Command**: Chọn lệnh **start** từ menu thả xuống (aaPanel tự động quét file `package.json` để lấy ra lệnh này).
   - **Project Port**: Điền `9000` (đây là cổng mặc định của MedusaJS API).
   - **Project Name**: Đặt tên gợi nhớ, ví dụ `mercur-api`.
   - **Binding Domain**: Nhập tên miền API của bạn: `s1.huylio.com`.
   - **User**: Chọn `www` hoặc `root` (khuyến nghị chọn `www`).
3. Nhấn **Submit** để khởi tạo.
4. **Cài đặt SSL cho API**:
   - Sau khi tạo xong, dự án sẽ tự động xuất hiện ở tab **Node project** và tên miền `s1.huylio.com` cũng tự động được thêm vào danh sách quản lý tên miền.
   - Click vào tên miền dự án `s1.huylio.com` -> Chọn tab **SSL** -> Chọn **Let's Encrypt** -> Tích chọn tên miền -> Nhấn **Apply** để bật HTTPS.

---

## 5. Triển khai Admin Panel & Vendor Panel (Giao diện Tĩnh)

> **Lưu ý quan trọng**: Hai trang Admin và Vendor là ứng dụng React Vite Single Page Application (SPA). Chúng đã được build thành các file tĩnh (`dist`) ở **Bước 3.1**. 
> Bạn **KHÔNG** nên dùng mục "Node project" cho 2 trang này vì làm vậy sẽ chạy thêm các tiến trình Node.js không cần thiết, làm tốn tài nguyên RAM/CPU của server. Cách tối ưu nhất là tạo Website tĩnh thông thường và để Nginx phục vụ trực tiếp.

### Bước 5.1: Cấu hình Website Admin Panel (admin.huylio.com)
1. Trên aaPanel, vào mục **Website** -> tab **HTML project** (hoặc mục Add Site thông thường).
2. Nhấp **Add Site** và nhập:
   - **Domain**: `admin.huylio.com`
   - **Document Root** (Thư mục gốc): `/www/wwwroot/mercur/apps/admin-test/dist`
   - **Database**: Chọn `No`
3. Nhấn **Submit**.
4. Cài đặt SSL: Click vào cấu hình website `admin.huylio.com` -> tab **SSL** -> **Let's Encrypt** -> **Apply**.
5. Cấu hình định tuyến **URL Rewrite** (Nginx Fallback):
   - Để tránh lỗi `404 Not Found` khi tải lại trang (F5) trong ứng dụng React Router, click vào cấu hình website -> chọn mục **URL Rewrite** và dán đoạn mã sau:
     ```nginx
     location / {
         try_files $uri $uri/ /index.html;
     }
     ```
   - Nhấn **Save**.

### Bước 5.2: Cấu hình Website Vendor Panel (vendor.huylio.com)
Thực hiện tương tự như Admin Panel:
1. Vào aaPanel -> **Website** -> **Add Site**:
   - **Domain**: `vendor.huylio.com`
   - **Document Root**: `/www/wwwroot/mercur/apps/vendor/dist`
2. Nhấn **Submit**.
3. Cài đặt SSL Let's Encrypt tương tự.
4. Cấu hình định tuyến **URL Rewrite** giống hệt Admin Panel:
   ```nginx
   location / {
       try_files $uri $uri/ /index.html;
   }
   ```
5. Nhấn **Save**.

---

## 6. Kiểm tra hoạt động
Sau khi cấu hình xong:
- Truy cập `https://s1.huylio.com/health` -> Nếu hiển thị trạng thái OK là Backend API đã chạy tốt.
- Truy cập `https://admin.huylio.com` để quản trị hệ thống.
- Truy cập `https://vendor.huylio.com` để quản trị dành cho Vendor.
