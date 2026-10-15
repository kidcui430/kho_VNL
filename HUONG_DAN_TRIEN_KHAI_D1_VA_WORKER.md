# HƯỚNG DẪN TRIỂN KHAI CLOUDFLARE D1 (SQLITE) & WORKER (PHA 1)

Tài liệu này cung cấp toàn bộ mã nguồn Database Schema và Backend Worker cho **Pha 1** (Nâng cấp Nhà Cung Cấp, thêm Tab Mã Hàng SX cho khối VP, logic Xuất Line SX đa ngành và quản lý kho chuyên nghiệp).

---

## MỤC LỤC
1. [Hướng Dẫn Cập Nhật Database Cloudflare D1](#1-hướng-dẫn-cập-nhật-database-cloudflare-d1)
   - [Trường hợp A: Database đã chạy từ trước (Nâng cấp an toàn không mất dữ liệu cũ)](#trường-hợp-a-database-đã-chạy-từ-trước)
   - [Trường hợp B: Tạo mới Database từ đầu](#trường-hợp-b-tạo-mới-database-từ-đầu)
2. [Hướng Dẫn Cập Nhật Cloudflare Worker](#2-hướng-dẫn-cập-nhật-cloudflare-worker)
3. [Kiểm Tra Các Tính Năng Mới Trên Giao Diện](#3-kiểm-tra-các-tính-năng-mới-trên-giao-diện)

---

## 1. HƯỚNG DẪN CẬP NHẬT DATABASE CLOUDFLARE D1

Vào **Cloudflare Dashboard** -> **Storage & Databases** -> **D1 SQL Database** -> Chọn Database của bạn -> Chọn tab **Console**.

### TÙY CHỌN 1: DROP SẠCH VÀ TẠO MỚI HOÀN TOÀN TỪ ĐẦU (Khuyến nghị cho D1 mới)
Copy toàn bộ đoạn mã SQL bên dưới, dán vào ô **Console** và nhấn **Execute**:

```sql
PRAGMA foreign_keys = OFF;

-- 1. DROP SẠCH BẢNG CŨ VÀ VIEW CŨ
DROP VIEW IF EXISTS v_ton_nvl;
DROP TABLE IF EXISTS Giao_Dich_NVL;
DROP TABLE IF EXISTS Danh_Muc_NVL;
DROP TABLE IF EXISTS San_Pham_SX;
DROP TABLE IF EXISTS Nha_Cung_Cap;
DROP TABLE IF EXISTS Nguoi_Dung;

PRAGMA foreign_keys = ON;

-- 2. TẠO MỚI BẢNG NGƯỜI DÙNG
CREATE TABLE Nguoi_Dung (
  ma_nv TEXT PRIMARY KEY,
  ten_nv TEXT NOT NULL,
  password TEXT NOT NULL,
  bo_phan TEXT NOT NULL CHECK (bo_phan IN ('VP','XUONG','SX')),
  role TEXT NOT NULL DEFAULT 'USER' CHECK (role IN ('ADMIN','USER')),
  active INTEGER NOT NULL DEFAULT 1,
  created_at TEXT NOT NULL DEFAULT (datetime('now', '+7 hours'))
);

-- 3. TẠO MỚI BẢNG NHÀ CUNG CẤP (NCC)
CREATE TABLE Nha_Cung_Cap (
  ma_ncc TEXT PRIMARY KEY,
  ten_ncc TEXT NOT NULL,
  dia_chi TEXT DEFAULT '',
  so_dt TEXT DEFAULT '',
  nguoi_lien_he TEXT DEFAULT '',
  email TEXT DEFAULT '',
  ghi_chu TEXT DEFAULT '',
  active INTEGER NOT NULL DEFAULT 1,
  created_at TEXT NOT NULL DEFAULT (datetime('now', '+7 hours')),
  updated_at TEXT NOT NULL DEFAULT (datetime('now', '+7 hours'))
);

-- 4. TẠO MỚI BẢNG NGUYÊN VẬT LIỆU (NVL)
CREATE TABLE Danh_Muc_NVL (
  ma_hang TEXT PRIMARY KEY,
  phan_loai TEXT NOT NULL,
  ten_hang TEXT NOT NULL,
  quy_cach TEXT DEFAULT '',
  don_vi_tinh TEXT NOT NULL,
  ma_ncc TEXT DEFAULT '',
  trang_thai TEXT NOT NULL DEFAULT 'ACTIVE' CHECK (trang_thai IN ('ACTIVE','BLOCKED','INACTIVE')),
  ghi_chu TEXT DEFAULT '',
  created_at TEXT NOT NULL DEFAULT (datetime('now', '+7 hours')),
  updated_at TEXT NOT NULL DEFAULT (datetime('now', '+7 hours'))
);

CREATE UNIQUE INDEX idx_nvl_unique_name ON Danh_Muc_NVL(phan_loai, ten_hang, quy_cach);
CREATE INDEX idx_nvl_phan_loai ON Danh_Muc_NVL(phan_loai);

-- 5. TẠO MỚI BẢNG MÃ HÀNG SẢN XUẤT (SẢN PHẨM / BÁN THÀNH PHẨM XUẤT LINE)
CREATE TABLE San_Pham_SX (
  ma_sp TEXT PRIMARY KEY,
  ten_sp TEXT NOT NULL,
  quy_cach TEXT DEFAULT '',
  don_vi_tinh TEXT DEFAULT 'Cái',
  ghi_chu TEXT DEFAULT '',
  active INTEGER NOT NULL DEFAULT 1,
  created_at TEXT NOT NULL DEFAULT (datetime('now', '+7 hours')),
  updated_at TEXT NOT NULL DEFAULT (datetime('now', '+7 hours'))
);

-- 6. TẠO MỚI BẢNG GIAO DỊCH KHO (SỔ CÁI LEDGER)
CREATE TABLE Giao_Dich_NVL (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  ma_hang TEXT NOT NULL,
  loai_giao_dich TEXT NOT NULL CHECK (loai_giao_dich IN (
    'OPENING',
    'NHAP_KHO',
    'XUAT_XUONG',
    'XUAT_SAN_XUAT',
    'YEU_CAU_XUONG',
    'TRA_KHO',
    'HOAN_TRA_XUONG',
    'HUY_KHO',
    'HUY_XUONG'
  )),
  so_luong REAL NOT NULL CHECK (so_luong > 0),
  sl_phu REAL DEFAULT 0,
  dvt_phu TEXT DEFAULT '',
  quy_cach_gd TEXT DEFAULT '',
  khoi_luong_kg REAL DEFAULT 0,
  delta_kho REAL NOT NULL DEFAULT 0,
  delta_xuong REAL NOT NULL DEFAULT 0,
  trang_thai TEXT NOT NULL DEFAULT 'approved' CHECK (trang_thai IN ('pending','approved','rejected')),
  nguoi_giao_dich TEXT NOT NULL,
  nguoi_duyet TEXT DEFAULT '',
  ma_ncc TEXT DEFAULT '',
  ma_sp_sx TEXT DEFAULT '',
  so_chung_tu TEXT DEFAULT '',
  don_gia REAL NOT NULL DEFAULT 0,
  thanh_tien REAL NOT NULL DEFAULT 0,
  ghi_chu TEXT DEFAULT '',
  thoi_gian_gd TEXT NOT NULL DEFAULT (datetime('now', '+7 hours')),
  created_at TEXT NOT NULL DEFAULT (datetime('now', '+7 hours')),
  approved_at TEXT,
  FOREIGN KEY (ma_hang) REFERENCES Danh_Muc_NVL(ma_hang),
  FOREIGN KEY (nguoi_giao_dich) REFERENCES Nguoi_Dung(ma_nv)
);

CREATE INDEX idx_gd_ngay ON Giao_Dich_NVL(thoi_gian_gd);
CREATE INDEX idx_gd_ma_hang ON Giao_Dich_NVL(ma_hang);
CREATE INDEX idx_gd_status ON Giao_Dich_NVL(trang_thai);
CREATE INDEX idx_gd_type_date ON Giao_Dich_NVL(loai_giao_dich, thoi_gian_gd);

-- 7. VIEW TỒN HIỆN TẠI
CREATE VIEW v_ton_nvl AS
SELECT
  d.ma_hang,
  d.phan_loai,
  d.ten_hang,
  d.quy_cach,
  d.don_vi_tinh,
  d.ma_ncc,
  d.trang_thai,
  d.ghi_chu,
  COALESCE(SUM(CASE WHEN g.trang_thai = 'approved' THEN g.delta_kho ELSE 0 END), 0) AS ton_kho,
  COALESCE(SUM(CASE WHEN g.trang_thai = 'approved' THEN g.delta_xuong ELSE 0 END), 0) AS ton_xuong
FROM Danh_Muc_NVL d
LEFT JOIN Giao_Dich_NVL g ON g.ma_hang = d.ma_hang
GROUP BY
  d.ma_hang, d.phan_loai, d.ten_hang, d.quy_cach,
  d.don_vi_tinh, d.ma_ncc, d.trang_thai, d.ghi_chu;

-- 8. DỮ LIỆU MẪU BAN ĐẦU
INSERT INTO Nguoi_Dung (ma_nv, ten_nv, password, bo_phan, role) VALUES
('ADMIN', 'Quản trị viên', '123', 'VP', 'ADMIN'),
('VP01', 'Nhân viên kho', '123', 'VP', 'USER'),
('XUONG01', 'Tổ sản xuất 01', '123', 'XUONG', 'USER');

INSERT INTO Nha_Cung_Cap (ma_ncc, ten_ncc, dia_chi, so_dt, nguoi_lien_he, email) VALUES
('NCC001', 'Nhà cung cấp Thép Việt Nhật', 'TP.HCM', '0901234567', 'Nguyễn Văn A', 'vietnhat@supplier.com'),
('NCC002', 'Công ty Hóa Chất Minh Trí', 'Bình Dương', '0908765432', 'Trần Thị B', 'minhtri@supplier.com');

INSERT INTO Danh_Muc_NVL (ma_hang, phan_loai, ten_hang, quy_cach, don_vi_tinh, ma_ncc, ghi_chu) VALUES
('NVL001', 'Hóa Chất', 'Dung dịch tẩy rửa công nghiệp', 'Can 20L', 'Lít', 'NCC002', 'Bảo quản nơi thoáng mát'),
('NVL002', 'Bao Bì', 'Thùng carton 5 lớp', '500x400x300 mm', 'Cái', 'NCC001', ''),
('NVL003', 'Kim Loại', 'Thép tròn đặc C45', 'Ø20 mm x 6m', 'Kg', 'NCC001', 'ĐVT kiểm đếm: Cây');

INSERT INTO San_Pham_SX (ma_sp, ten_sp, quy_cach, don_vi_tinh, ghi_chu) VALUES
('SP001', 'Trục dẫn hướng máy CNC', 'Ø20 x 500mm, tôi cao tần', 'Cái', 'Mã bản vẽ: DWG-SP01'),
('SP002', 'Bạc đạn trượt tuyến tính', 'Size 30mm', 'Bộ', 'Gia công chính xác');

-- Tồn đầu kỳ mẫu
INSERT INTO Giao_Dich_NVL
(ma_hang, loai_giao_dich, so_luong, delta_kho, delta_xuong, trang_thai, nguoi_giao_dich, ghi_chu, thoi_gian_gd)
VALUES
('NVL001', 'OPENING', 500, 500, 0, 'approved', 'ADMIN', 'Tồn đầu kỳ ban đầu', datetime('now','-10 days','+7 hours')),
('NVL002', 'OPENING', 1200, 1000, 200, 'approved', 'ADMIN', 'Tồn đầu kỳ ban đầu', datetime('now','-10 days','+7 hours')),
('NVL003', 'OPENING', 300, 250, 50, 'approved', 'ADMIN', 'Tồn đầu kỳ ban đầu', datetime('now','-10 days','+7 hours'));
```

---

### TÙY CHỌN 2: CHỈ CẬP NHẬT CỘT MỚI (NẾU MUỐN GIỮ DỮ LIỆU CŨ)

```sql
-- 1. Bổ sung cột cập nhật thời gian cho Nhà Cung Cấp
ALTER TABLE Nha_Cung_Cap ADD COLUMN updated_at TEXT DEFAULT (datetime('now', '+7 hours'));

-- 2. Bổ sung các cột thông tin kỹ thuật cho Mã Hàng Sản Xuất
ALTER TABLE San_Pham_SX ADD COLUMN quy_cach TEXT DEFAULT '';
ALTER TABLE San_Pham_SX ADD COLUMN don_vi_tinh TEXT DEFAULT 'Cái';
ALTER TABLE San_Pham_SX ADD COLUMN ghi_chu TEXT DEFAULT '';
ALTER TABLE San_Pham_SX ADD COLUMN created_at TEXT DEFAULT (datetime('now', '+7 hours'));
ALTER TABLE San_Pham_SX ADD COLUMN updated_at TEXT DEFAULT (datetime('now', '+7 hours'));

-- 3. Bổ sung các cột quy cách đóng gói đa ngành cho Giao Dịch Kho
ALTER TABLE Giao_Dich_NVL ADD COLUMN sl_phu REAL DEFAULT 0;
ALTER TABLE Giao_Dich_NVL ADD COLUMN dvt_phu TEXT DEFAULT '';
ALTER TABLE Giao_Dich_NVL ADD COLUMN quy_cach_gd TEXT DEFAULT '';
ALTER TABLE Giao_Dich_NVL ADD COLUMN khoi_luong_kg REAL DEFAULT 0;
```

---

### Trường hợp B: Tạo mới Database từ đầu
*(Nếu bạn tạo một DB test mới hoặc khởi tạo lần đầu)*:

Copy toàn bộ nội dung file [schema.txt](file:///d:/HTML-CSS/Study/Kho_VLN/schema.txt) bên dưới vào Console:

```sql
PRAGMA foreign_keys = ON;

-- 1. BẢNG NGƯỜI DÙNG
CREATE TABLE IF NOT EXISTS Nguoi_Dung (
  ma_nv TEXT PRIMARY KEY,
  ten_nv TEXT NOT NULL,
  password TEXT NOT NULL,
  bo_phan TEXT NOT NULL CHECK (bo_phan IN ('VP','XUONG','SX')),
  role TEXT NOT NULL DEFAULT 'USER' CHECK (role IN ('ADMIN','USER')),
  active INTEGER NOT NULL DEFAULT 1,
  created_at TEXT NOT NULL DEFAULT (datetime('now', '+7 hours'))
);

-- 2. BẢNG NHÀ CUNG CẤP (NCC)
CREATE TABLE IF NOT EXISTS Nha_Cung_Cap (
  ma_ncc TEXT PRIMARY KEY,
  ten_ncc TEXT NOT NULL,
  dia_chi TEXT DEFAULT '',
  so_dt TEXT DEFAULT '',
  nguoi_lien_he TEXT DEFAULT '',
  email TEXT DEFAULT '',
  ghi_chu TEXT DEFAULT '',
  active INTEGER NOT NULL DEFAULT 1,
  created_at TEXT NOT NULL DEFAULT (datetime('now', '+7 hours')),
  updated_at TEXT NOT NULL DEFAULT (datetime('now', '+7 hours'))
);

-- 3. BẢNG DANH MỤC NGUYÊN VẬT LIỆU (NVL)
CREATE TABLE IF NOT EXISTS Danh_Muc_NVL (
  ma_hang TEXT PRIMARY KEY,
  phan_loai TEXT NOT NULL,
  ten_hang TEXT NOT NULL,
  quy_cach TEXT DEFAULT '',
  don_vi_tinh TEXT NOT NULL,
  ma_ncc TEXT DEFAULT '',
  trang_thai TEXT NOT NULL DEFAULT 'ACTIVE' CHECK (trang_thai IN ('ACTIVE','BLOCKED','INACTIVE')),
  ghi_chu TEXT DEFAULT '',
  created_at TEXT NOT NULL DEFAULT (datetime('now', '+7 hours')),
  updated_at TEXT NOT NULL DEFAULT (datetime('now', '+7 hours'))
);

CREATE UNIQUE INDEX IF NOT EXISTS idx_nvl_unique_name
ON Danh_Muc_NVL(phan_loai, ten_hang, quy_cach);

CREATE INDEX IF NOT EXISTS idx_nvl_phan_loai
ON Danh_Muc_NVL(phan_loai);

-- 4. BẢNG MÃ HÀNG SẢN XUẤT (SẢN PHẨM / BÁN THÀNH PHẨM XUẤT LINE)
CREATE TABLE IF NOT EXISTS San_Pham_SX (
  ma_sp TEXT PRIMARY KEY,
  ten_sp TEXT NOT NULL,
  quy_cach TEXT DEFAULT '',
  don_vi_tinh TEXT DEFAULT 'Cái',
  ghi_chu TEXT DEFAULT '',
  active INTEGER NOT NULL DEFAULT 1,
  created_at TEXT NOT NULL DEFAULT (datetime('now', '+7 hours')),
  updated_at TEXT NOT NULL DEFAULT (datetime('now', '+7 hours'))
);

-- 5. BẢNG GIAO DỊCH KHO (LEDGER SỔ CÁI)
CREATE TABLE IF NOT EXISTS Giao_Dich_NVL (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  ma_hang TEXT NOT NULL,
  loai_giao_dich TEXT NOT NULL CHECK (loai_giao_dich IN (
    'OPENING',
    'NHAP_KHO',
    'XUAT_XUONG',
    'XUAT_SAN_XUAT',
    'YEU_CAU_XUONG',
    'TRA_KHO',
    'HOAN_TRA_XUONG',
    'HUY_KHO',
    'HUY_XUONG'
  )),
  so_luong REAL NOT NULL CHECK (so_luong > 0),
  sl_phu REAL DEFAULT 0,
  dvt_phu TEXT DEFAULT '',
  quy_cach_gd TEXT DEFAULT '',
  khoi_luong_kg REAL DEFAULT 0,
  delta_kho REAL NOT NULL DEFAULT 0,
  delta_xuong REAL NOT NULL DEFAULT 0,
  trang_thai TEXT NOT NULL DEFAULT 'approved' CHECK (trang_thai IN ('pending','approved','rejected')),
  nguoi_giao_dich TEXT NOT NULL,
  nguoi_duyet TEXT DEFAULT '',
  ma_ncc TEXT DEFAULT '',
  ma_sp_sx TEXT DEFAULT '',
  so_chung_tu TEXT DEFAULT '',
  don_gia REAL NOT NULL DEFAULT 0,
  thanh_tien REAL NOT NULL DEFAULT 0,
  ghi_chu TEXT DEFAULT '',
  thoi_gian_gd TEXT NOT NULL DEFAULT (datetime('now', '+7 hours')),
  created_at TEXT NOT NULL DEFAULT (datetime('now', '+7 hours')),
  approved_at TEXT,
  FOREIGN KEY (ma_hang) REFERENCES Danh_Muc_NVL(ma_hang),
  FOREIGN KEY (nguoi_giao_dich) REFERENCES Nguoi_Dung(ma_nv)
);

CREATE INDEX IF NOT EXISTS idx_gd_ngay
ON Giao_Dich_NVL(thoi_gian_gd);

CREATE INDEX IF NOT EXISTS idx_gd_ma_hang
ON Giao_Dich_NVL(ma_hang);

CREATE INDEX IF NOT EXISTS idx_gd_status
ON Giao_Dich_NVL(trang_thai);

CREATE INDEX IF NOT EXISTS idx_gd_type_date
ON Giao_Dich_NVL(loai_giao_dich, thoi_gian_gd);

-- 6. VIEW TỒN HIỆN TẠI (TÍNH TỪ TOÀN BỘ GIAO DỊCH APPROVED)
CREATE VIEW IF NOT EXISTS v_ton_nvl AS
SELECT
  d.ma_hang,
  d.phan_loai,
  d.ten_hang,
  d.quy_cach,
  d.don_vi_tinh,
  d.ma_ncc,
  d.trang_thai,
  d.ghi_chu,
  COALESCE(SUM(CASE WHEN g.trang_thai = 'approved' THEN g.delta_kho ELSE 0 END), 0) AS ton_kho,
  COALESCE(SUM(CASE WHEN g.trang_thai = 'approved' THEN g.delta_xuong ELSE 0 END), 0) AS ton_xuong
FROM Danh_Muc_NVL d
LEFT JOIN Giao_Dich_NVL g ON g.ma_hang = d.ma_hang
GROUP BY
  d.ma_hang, d.phan_loai, d.ten_hang, d.quy_cach,
  d.don_vi_tinh, d.ma_ncc, d.trang_thai, d.ghi_chu;

-- DỮ LIỆU KHỞI TẠO MẪU
INSERT OR IGNORE INTO Nguoi_Dung (ma_nv, ten_nv, password, bo_phan, role) VALUES
('ADMIN', 'Quản trị viên', '123', 'VP', 'ADMIN'),
('VP01', 'Nhân viên kho', '123', 'VP', 'USER'),
('XUONG01', 'Tổ sản xuất 01', '123', 'XUONG', 'USER');

INSERT OR IGNORE INTO Nha_Cung_Cap (ma_ncc, ten_ncc, dia_chi, so_dt, nguoi_lien_he, email) VALUES
('NCC001', 'Nhà cung cấp Thép Việt Nhật', 'TP.HCM', '0901234567', 'Nguyễn Văn A', 'vietnhat@supplier.com'),
('NCC002', 'Công ty Hóa Chất Minh Trí', 'Bình Dương', '0908765432', 'Trần Thị B', 'minhtri@supplier.com');

INSERT OR IGNORE INTO Danh_Muc_NVL
(ma_hang, phan_loai, ten_hang, quy_cach, don_vi_tinh, ma_ncc, ghi_chu)
VALUES
('NVL001', 'Hóa Chất', 'Dung dịch tẩy rửa công nghiệp', 'Can 20L', 'Lít', 'NCC002', 'Bảo quản nơi khô ráo'),
('NVL002', 'Bao Bì', 'Thùng carton 5 lớp', '500x400x300 mm', 'Cái', 'NCC001', ''),
('NVL003', 'Kim Loại', 'Thép tròn đặc C45', 'Ø20 mm x 6m', 'Kg', 'NCC001', 'ĐVT kiểm đếm: Cây');

INSERT OR IGNORE INTO San_Pham_SX (ma_sp, ten_sp, quy_cach, don_vi_tinh, ghi_chu) VALUES
('SP001', 'Trục dẫn hướng máy CNC', 'Ø20 x 500mm, tôi cao tần', 'Cái', 'Mã bản vẽ: DWG-SP01'),
('SP002', 'Bạc đạn trượt tuyến tính', 'Size 30mm', 'Bộ', 'Gia công chính xác');
```

---

## 2. HƯỚNG DẪN CẬP NHẬT CLOUDFLARE WORKER

1. Vào **Cloudflare Dashboard** -> **Workers & Pages** -> Chọn Worker API của bạn (ví dụ `api-nvl`).
2. Nhấn vào nút **Edit code** (Quick Edit).
3. Xóa toàn bộ nội dung cũ trong tab `index.js` (hoặc `worker.js`) và dán toàn bộ đoạn code từ file [worker.txt](file:///d:/HTML-CSS/Study/Kho_VLN/worker.txt) vào:

```javascript
// ============================================================
// CLOUDFLARE WORKER BACKEND - QUẢN LÝ KHO & SẢN XUẤT (D1 SQLITE)
// ============================================================

const corsHeaders = {
  "Access-Control-Allow-Origin": "*",
  "Access-Control-Allow-Methods": "GET, POST, PUT, OPTIONS",
  "Access-Control-Allow-Headers": "Content-Type",
};

const jsonHeaders = { ...corsHeaders, "Content-Type": "application/json; charset=UTF-8" };

function json(data, status = 200) {
  return new Response(JSON.stringify(data), { status, headers: jsonHeaders });
}

function bad(message, status = 400) {
  return json({ error: message }, status);
}

async function body(request) {
  try {
    return await request.json();
  } catch {
    return {};
  }
}

function cleanDate(value) {
  return /^\d{4}-\d{2}-\d{2}$/.test(value || "") ? value : null;
}

function movementFor(type, qty) {
  const q = Number(qty);
  const map = {
    NHAP_KHO: { dk: q, dx: 0 },
    XUAT_XUONG: { dk: -q, dx: q },
    XUAT_SAN_XUAT: { dk: -q, dx: q },
    YEU_CAU_XUONG: { dk: -q, dx: q },
    TRA_KHO: { dk: q, dx: -q },
    HOAN_TRA_XUONG: { dk: q, dx: -q },
    HUY_KHO: { dk: -q, dx: 0 },
    HUY_XUONG: { dk: 0, dx: -q },
  };
  return map[type] || null;
}

async function getUser(env, maNv) {
  return env.DB.prepare(
    "SELECT ma_nv, ten_nv, bo_phan, role, active FROM Nguoi_Dung WHERE ma_nv = ?"
  ).bind(maNv).first();
}

async function getStock(env, maHang) {
  return env.DB.prepare(
    "SELECT ma_hang, ton_kho, ton_xuong, trang_thai FROM v_ton_nvl WHERE ma_hang = ?"
  ).bind(maHang).first();
}

function canUseType(user, type) {
  if (!user) return false;
  if (user.bo_phan === "XUONG" || user.bo_phan === "SX") {
    return ["NHAP_KHO", "XUAT_SAN_XUAT", "XUAT_XUONG", "YEU_CAU_XUONG", "HOAN_TRA_XUONG", "HUY_XUONG"].includes(type);
  }
  return ["NHAP_KHO", "XUAT_SAN_XUAT", "XUAT_XUONG", "HUY_KHO", "TRA_KHO"].includes(type);
}

function validateStockAfter(stock, movement) {
  const warehouse = Number(stock?.ton_kho || 0) + movement.dk;
  const workshop = Number(stock?.ton_xuong || 0) + movement.dx;
  if (warehouse < -1e-9) return "Tồn kho không đủ cho giao dịch này.";
  if (workshop < -1e-9) return "Tồn xưởng không đủ cho giao dịch này.";
  return null;
}

function labelSql() {
  return `CASE g.loai_giao_dich
    WHEN 'NHAP_KHO' THEN 'Nhập kho'
    WHEN 'XUAT_XUONG' THEN 'Xuất cho xưởng'
    WHEN 'XUAT_SAN_XUAT' THEN 'Xuất ra Line Sản Xuất'
    WHEN 'YEU_CAU_XUONG' THEN 'Xưởng yêu cầu NVL'
    WHEN 'TRA_KHO' THEN 'Trả kho'
    WHEN 'HOAN_TRA_XUONG' THEN 'Xưởng hoàn trả'
    WHEN 'HUY_KHO' THEN 'Hủy tại kho'
    WHEN 'HUY_XUONG' THEN 'Hủy tại xưởng'
    WHEN 'OPENING' THEN 'Tồn đầu kỳ'
    ELSE g.loai_giao_dich END`;
}

export default {
  async fetch(request, env) {
    if (request.method === "OPTIONS") return new Response(null, { headers: corsHeaders });
    const url = new URL(request.url);
    const path = url.pathname;

    try {
      // 1. AUTH
      if (path === "/api/login" && request.method === "POST") {
        const p = await body(request);
        const user = await env.DB.prepare(`
          SELECT ma_nv, ten_nv, bo_phan, role
          FROM Nguoi_Dung
          WHERE ma_nv = ? AND password = ? AND active = 1
        `).bind(String(p.ma_nv || "").toUpperCase().trim(), String(p.password || "")).first();
        if (!user) return bad("Sai tài khoản hoặc mật khẩu.", 401);
        return json(user);
      }

      if (path === "/api/change-password" && request.method === "POST") {
        const p = await body(request);
        if (!p.ma_nv || !p.old_password || !p.new_password) return bad("Thiếu dữ liệu đổi mật khẩu.");
        const found = await env.DB.prepare(
          "SELECT ma_nv FROM Nguoi_Dung WHERE ma_nv=? AND password=? AND active=1"
        ).bind(p.ma_nv, p.old_password).first();
        if (!found) return bad("Mật khẩu hiện tại không đúng.", 401);
        await env.DB.prepare("UPDATE Nguoi_Dung SET password=? WHERE ma_nv=?")
          .bind(p.new_password, p.ma_nv).run();
        return json({ success: true });
      }

      // 2. BOOTSTRAP MASTER DATA
      if (path === "/api/bootstrap" && request.method === "GET") {
        const [items, nccs, products] = await Promise.all([
          env.DB.prepare(`SELECT * FROM v_ton_nvl WHERE trang_thai != 'INACTIVE' ORDER BY phan_loai, ten_hang`).all(),
          env.DB.prepare(`SELECT * FROM Nha_Cung_Cap WHERE active=1 ORDER BY ten_ncc`).all(),
          env.DB.prepare(`SELECT ma_sp, ten_sp, quy_cach, don_vi_tinh, ghi_chu, active FROM San_Pham_SX WHERE active=1 ORDER BY ten_sp`).all(),
        ]);
        return json({
          items: items.results || [],
          nccs: nccs.results || [],
          products: products.results || [],
        });
      }

      // 3. NVL ITEMS
      if (path === "/api/items" && request.method === "POST") {
        const p = await body(request);
        if (!p.phan_loai || !p.ten_hang || !p.don_vi_tinh) return bad("Phân loại, tên NVL và đơn vị tính là bắt buộc.");

        const row = await env.DB.prepare(`
          SELECT COALESCE(MAX(CAST(SUBSTR(ma_hang, 4) AS INTEGER)), 0) AS max_no
          FROM Danh_Muc_NVL WHERE ma_hang LIKE 'NVL%'
        `).first();
        const maHang = `NVL${String(Number(row?.max_no || 0) + 1).padStart(3, "0")}`;

        await env.DB.prepare(`
          INSERT INTO Danh_Muc_NVL
          (ma_hang, phan_loai, ten_hang, quy_cach, don_vi_tinh, ma_ncc, ghi_chu)
          VALUES (?, ?, ?, ?, ?, ?, ?)
        `).bind(
          maHang,
          p.phan_loai.trim(), p.ten_hang.trim(), p.quy_cach || "",
          p.don_vi_tinh, p.ma_ncc || "", p.ghi_chu || ""
        ).run();

        return json({ success: true, ma_hang: maHang });
      }

      if (path === "/api/items/edit" && request.method === "POST") {
        const p = await body(request);
        if (!p.ma_hang || !p.ten_hang || !p.don_vi_tinh) return bad("Mã NVL, tên NVL và ĐVT không được để trống.");

        await env.DB.prepare(`
          UPDATE Danh_Muc_NVL
          SET phan_loai = ?, ten_hang = ?, quy_cach = ?, don_vi_tinh = ?, ma_ncc = ?, ghi_chu = ?, updated_at = datetime('now','+7 hours')
          WHERE ma_hang = ?
        `).bind(
          p.phan_loai || "", p.ten_hang.trim(), p.quy_cach || "",
          p.don_vi_tinh, p.ma_ncc || "", p.ghi_chu || "", p.ma_hang
        ).run();

        return json({ success: true });
      }

      if (path === "/api/items/delete" && request.method === "POST") {
        const p = await body(request);
        if (!p.ma_hang) return bad("Thiếu mã NVL cần xóa.");

        const tx = await env.DB.prepare(
          "SELECT COUNT(*) AS total FROM Giao_Dich_NVL WHERE ma_hang = ?"
        ).bind(p.ma_hang).first();

        if (Number(tx?.total || 0) > 0) {
          await env.DB.prepare(`
            UPDATE Danh_Muc_NVL
            SET trang_thai = 'INACTIVE', updated_at = datetime('now','+7 hours')
            WHERE ma_hang = ?
          `).bind(p.ma_hang).run();
          return json({ success: true, mode: "deactivated", message: "NVL đã có lịch sử giao dịch nên đã được chuyển sang trạng thái Ẩn." });
        } else {
          await env.DB.prepare("DELETE FROM Danh_Muc_NVL WHERE ma_hang = ?").bind(p.ma_hang).run();
          return json({ success: true, mode: "deleted", message: "Đã xóa hoàn toàn NVL." });
        }
      }

      if (path === "/api/items/status" && request.method === "POST") {
        const p = await body(request);
        if (!p.ma_hang || !["ACTIVE", "BLOCKED", "INACTIVE"].includes(p.trang_thai)) return bad("Dữ liệu trạng thái không hợp lệ.");
        await env.DB.prepare(`
          UPDATE Danh_Muc_NVL
          SET trang_thai=?, ghi_chu=?, updated_at=datetime('now','+7 hours')
          WHERE ma_hang=?
        `).bind(p.trang_thai, p.ghi_chu || "", p.ma_hang).run();
        return json({ success: true });
      }

      // 4. NHÀ CUNG CẤP (NCC)
      if (path === "/api/suppliers" && request.method === "POST") {
        const p = await body(request);
        if (!p.ten_ncc) return bad("Tên nhà cung cấp là bắt buộc.");
        const row = await env.DB.prepare(`
          SELECT COALESCE(MAX(CAST(SUBSTR(ma_ncc, 4) AS INTEGER)), 0) AS max_no
          FROM Nha_Cung_Cap WHERE ma_ncc LIKE 'NCC%'
        `).first();
        const maNcc = `NCC${String(Number(row?.max_no || 0) + 1).padStart(3, "0")}`;
        await env.DB.prepare(`
          INSERT INTO Nha_Cung_Cap
          (ma_ncc, ten_ncc, dia_chi, so_dt, nguoi_lien_he, email, ghi_chu)
          VALUES (?, ?, ?, ?, ?, ?, ?)
        `).bind(maNcc, p.ten_ncc.trim(), p.dia_chi || "", p.so_dt || "", p.nguoi_lien_he || "", p.email || "", p.ghi_chu || "").run();
        return json({ success: true, ma_ncc: maNcc });
      }

      if (path === "/api/suppliers/edit" && request.method === "POST") {
        const p = await body(request);
        if (!p.ma_ncc || !p.ten_ncc) return bad("Mã và tên nhà cung cấp là bắt buộc.");
        await env.DB.prepare(`
          UPDATE Nha_Cung_Cap
          SET ten_ncc = ?, dia_chi = ?, so_dt = ?, nguoi_lien_he = ?, email = ?, ghi_chu = ?, updated_at = datetime('now','+7 hours')
          WHERE ma_ncc = ?
        `).bind(
          p.ten_ncc.trim(), p.dia_chi || "", p.so_dt || "",
          p.nguoi_lien_he || "", p.email || "", p.ghi_chu || "", p.ma_ncc
        ).run();
        return json({ success: true });
      }

      if (path === "/api/suppliers/delete" && request.method === "POST") {
        const p = await body(request);
        if (!p.ma_ncc) return bad("Thiếu mã NCC cần xóa.");

        const [txCount, nvlCount] = await Promise.all([
          env.DB.prepare("SELECT COUNT(*) AS total FROM Giao_Dich_NVL WHERE ma_ncc = ?").bind(p.ma_ncc).first(),
          env.DB.prepare("SELECT COUNT(*) AS total FROM Danh_Muc_NVL WHERE ma_ncc = ?").bind(p.ma_ncc).first(),
        ]);

        const hasHistory = (Number(txCount?.total || 0) > 0) || (Number(nvlCount?.total || 0) > 0);

        if (hasHistory) {
          await env.DB.prepare("UPDATE Nha_Cung_Cap SET active = 0, updated_at = datetime('now','+7 hours') WHERE ma_ncc = ?").bind(p.ma_ncc).run();
          return json({ success: true, mode: "deactivated", message: "NCC đã có liên kết dữ liệu nên hệ thống đã chuyển sang trạng thái Ẩn." });
        } else {
          await env.DB.prepare("DELETE FROM Nha_Cung_Cap WHERE ma_ncc = ?").bind(p.ma_ncc).run();
          return json({ success: true, mode: "deleted", message: "Đã xóa hoàn toàn nhà cung cấp." });
        }
      }

      // 5. MÃ HÀNG SẢN XUẤT (San_Pham_SX)
      if (path === "/api/products" && request.method === "GET") {
        const result = await env.DB.prepare(
          "SELECT ma_sp, ten_sp, quy_cach, don_vi_tinh, ghi_chu, active FROM San_Pham_SX WHERE active=1 ORDER BY ma_sp"
        ).all();
        return json({ products: result.results || [] });
      }

      if (path === "/api/products" && request.method === "POST") {
        const p = await body(request);
        if (!p.ten_sp) return bad("Tên sản phẩm sản xuất là bắt buộc.");

        let maSp = (p.ma_sp || "").trim().toUpperCase();
        if (!maSp) {
          const row = await env.DB.prepare(`
            SELECT COALESCE(MAX(CAST(SUBSTR(ma_sp, 3) AS INTEGER)), 0) AS max_no
            FROM San_Pham_SX WHERE ma_sp LIKE 'SP%'
          `).first();
          maSp = `SP${String(Number(row?.max_no || 0) + 1).padStart(3, "0")}`;
        }

        const exist = await env.DB.prepare("SELECT ma_sp FROM San_Pham_SX WHERE ma_sp = ?").bind(maSp).first();
        if (exist) return bad(`Mã sản phẩm '${maSp}' đã tồn tại.`);

        await env.DB.prepare(`
          INSERT INTO San_Pham_SX
          (ma_sp, ten_sp, quy_cach, don_vi_tinh, ghi_chu, active)
          VALUES (?, ?, ?, ?, ?, 1)
        `).bind(
          maSp, p.ten_sp.trim(), p.quy_cach || "",
          p.don_vi_tinh || "Cái", p.ghi_chu || ""
        ).run();

        return json({ success: true, ma_sp: maSp });
      }

      if (path === "/api/products/edit" && request.method === "POST") {
        const p = await body(request);
        if (!p.ma_sp || !p.ten_sp) return bad("Mã và tên sản phẩm là bắt buộc.");

        await env.DB.prepare(`
          UPDATE San_Pham_SX
          SET ten_sp = ?, quy_cach = ?, don_vi_tinh = ?, ghi_chu = ?, updated_at = datetime('now','+7 hours')
          WHERE ma_sp = ?
        `).bind(
          p.ten_sp.trim(), p.quy_cach || "",
          p.don_vi_tinh || "Cái", p.ghi_chu || "", p.ma_sp
        ).run();

        return json({ success: true });
      }

      if (path === "/api/products/delete" && request.method === "POST") {
        const p = await body(request);
        if (!p.ma_sp) return bad("Thiếu mã sản phẩm cần xóa.");

        const tx = await env.DB.prepare(
          "SELECT COUNT(*) AS total FROM Giao_Dich_NVL WHERE ma_sp_sx = ?"
        ).bind(p.ma_sp).first();

        if (Number(tx?.total || 0) > 0) {
          await env.DB.prepare("UPDATE San_Pham_SX SET active = 0, updated_at = datetime('now','+7 hours') WHERE ma_sp = ?").bind(p.ma_sp).run();
          return json({ success: true, mode: "deactivated", message: "Mã hàng đã có lịch sử xuất line nên hệ thống đã chuyển sang trạng thái Ẩn." });
        } else {
          await env.DB.prepare("DELETE FROM San_Pham_SX WHERE ma_sp = ?").bind(p.ma_sp).run();
          return json({ success: true, mode: "deleted", message: "Đã xóa hoàn toàn mã hàng sản xuất." });
        }
      }

      // 6. GIAO DỊCH KHO
      if (path === "/api/transactions" && request.method === "POST") {
        const p = await body(request);
        const user = await getUser(env, p.user_id);
        if (!user || !user.active) return bad("Người dùng không hợp lệ.", 401);
        if (!Array.isArray(p.transactions) || p.transactions.length === 0) return bad("Phiếu chưa có NVL.");

        const txDate = cleanDate(p.tx_date);
        if (!txDate) return bad("Ngày giao dịch không hợp lệ.");

        const isWorkshop = user.bo_phan === "XUONG" || user.bo_phan === "SX";
        const pending = isWorkshop;
        const projected = new Map();
        const preparedRows = [];

        for (const item of p.transactions) {
          const qty = Number(item.so_luong);
          if (!item.ma_hang || !Number.isFinite(qty) || qty <= 0) return bad("NVL hoặc số lượng không hợp lệ.");
          if (!canUseType(user, item.loai_giao_dich)) return bad(`Loại giao dịch ${item.loai_giao_dich} không hợp lệ với bộ phận hiện tại.`);

          const stock = await getStock(env, item.ma_hang);
          if (!stock) return bad(`Không tìm thấy mã NVL ${item.ma_hang}.`);
          if (stock.trang_thai === "BLOCKED" && ["XUAT_XUONG", "XUAT_SAN_XUAT", "YEU_CAU_XUONG"].includes(item.loai_giao_dich)) {
            return bad(`Mã ${item.ma_hang} đang bị chặn xuất.`);
          }

          const mv = movementFor(item.loai_giao_dich, qty);
          if (!mv) return bad("Loại giao dịch không được hỗ trợ.");

          if (pending) {
            const err = validateStockAfter(stock, mv);
            if (err) return bad(`${stock.ma_hang}: ${err}`);
          } else {
            const current = projected.get(item.ma_hang) || {
              ton_kho: Number(stock.ton_kho || 0),
              ton_xuong: Number(stock.ton_xuong || 0),
            };
            const err = validateStockAfter(current, mv);
            if (err) return bad(`${stock.ma_hang}: ${err}`);
            current.ton_kho += mv.dk;
            current.ton_xuong += mv.dx;
            projected.set(item.ma_hang, current);
          }

          preparedRows.push({ item, qty, mv });
        }

        const stmts = preparedRows.map(({ item, qty, mv }) => env.DB.prepare(`
          INSERT INTO Giao_Dich_NVL
          (ma_hang, loai_giao_dich, so_luong, sl_phu, dvt_phu, quy_cach_gd, khoi_luong_kg,
           delta_kho, delta_xuong, trang_thai, nguoi_giao_dich, ma_ncc, ma_sp_sx, so_chung_tu,
           don_gia, thanh_tien, ghi_chu, thoi_gian_gd)
          VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ? || ' ' || strftime('%H:%M:%S','now','+7 hours'))
        `).bind(
          item.ma_hang, item.loai_giao_dich, qty,
          Number(item.sl_phu || 0), item.dvt_phu || "", item.quy_cach_gd || "", Number(item.khoi_luong_kg || 0),
          mv.dk, mv.dx,
          pending ? "pending" : "approved", user.ma_nv,
          item.ma_ncc || "", item.ma_sp_sx || "", item.so_chung_tu || "",
          Number(item.don_gia || 0), Number(item.don_gia || 0) * qty,
          item.ghi_chu || "", txDate
        ));

        await env.DB.batch(stmts);
        return json({ success: true, status: pending ? "pending" : "approved", count: stmts.length });
      }

      // 7. DUYỆT PHIẾU PENDING
      if (path === "/api/approve" && request.method === "POST") {
        const p = await body(request);
        const approver = await getUser(env, p.user_id);
        if (!approver || (approver.bo_phan !== "VP" && approver.role !== "ADMIN")) return bad("Bạn không có quyền duyệt.", 403);

        const tx = await env.DB.prepare(`
          SELECT * FROM Giao_Dich_NVL WHERE id=? AND trang_thai='pending'
        `).bind(Number(p.id)).first();
        if (!tx) return bad("Phiếu pending không tồn tại hoặc đã được xử lý.");

        if (p.action === "reject") {
          await env.DB.prepare(`
            UPDATE Giao_Dich_NVL
            SET trang_thai='rejected', nguoi_duyet=?, approved_at=datetime('now','+7 hours'),
                ghi_chu=CASE WHEN ?='' THEN ghi_chu ELSE ghi_chu || ' | Từ chối: ' || ? END
            WHERE id=? AND trang_thai='pending'
          `).bind(approver.ma_nv, p.reason || "", p.reason || "", tx.id).run();
          return json({ success: true });
        }

        if (p.action !== "approve") return bad("Action không hợp lệ.");
        const stock = await getStock(env, tx.ma_hang);
        const mv = { dk: Number(tx.delta_kho), dx: Number(tx.delta_xuong) };
        const err = validateStockAfter(stock, mv);
        if (err) return bad(`Không thể duyệt: ${err}`);
        if (stock.trang_thai === "BLOCKED" && ["YEU_CAU_XUONG", "XUAT_SAN_XUAT", "XUAT_XUONG"].includes(tx.loai_giao_dich)) {
          return bad("NVL đang bị chặn xuất.");
        }

        await env.DB.prepare(`
          UPDATE Giao_Dich_NVL
          SET trang_thai='approved', nguoi_duyet=?, approved_at=datetime('now','+7 hours')
          WHERE id=? AND trang_thai='pending'
        `).bind(approver.ma_nv, tx.id).run();
        return json({ success: true });
      }

      // 8. LỊCH SỬ GIAO DỊCH
      if (path === "/api/history" && request.method === "GET") {
        const from = cleanDate(url.searchParams.get("from"));
        const to = cleanDate(url.searchParams.get("to"));
        const type = url.searchParams.get("type") || "";
        const q = (url.searchParams.get("q") || "").trim();

        const wh = ["1=1"];
        const binds = [];
        if (from) { wh.push("date(g.thoi_gian_gd) >= ?"); binds.push(from); }
        if (to) { wh.push("date(g.thoi_gian_gd) <= ?"); binds.push(to); }
        if (type) { wh.push("g.loai_giao_dich = ?"); binds.push(type); }
        if (q) {
          wh.push("(g.ma_hang LIKE ? OR d.ten_hang LIKE ? OR d.phan_loai LIKE ? OR s.ten_sp LIKE ? OR g.ma_sp_sx LIKE ?)");
          const like = `%${q}%`; binds.push(like, like, like, like, like);
        }

        const result = await env.DB.prepare(`
          SELECT g.*, d.ten_hang, d.phan_loai, d.don_vi_tinh,
                 u.ten_nv, s.ten_sp, ${labelSql()} AS loai_hien_thi
          FROM Giao_Dich_NVL g
          JOIN Danh_Muc_NVL d ON d.ma_hang=g.ma_hang
          LEFT JOIN Nguoi_Dung u ON u.ma_nv=g.nguoi_giao_dich
          LEFT JOIN San_Pham_SX s ON s.ma_sp=g.ma_sp_sx
          WHERE ${wh.join(" AND ")}
          ORDER BY g.thoi_gian_gd DESC, g.id DESC
          LIMIT 1000
        `).bind(...binds).all();
        return json({ rows: result.results || [] });
      }

      // 9. BÁO CÁO MATRIX
      if (path === "/api/report/daily" && request.method === "GET") {
        const from = cleanDate(url.searchParams.get("from"));
        const to = cleanDate(url.searchParams.get("to"));
        const type = url.searchParams.get("type") || "";
        const q = (url.searchParams.get("q") || "").trim();

        if (!from || !to) return bad("Cần from và to theo YYYY-MM-DD.");
        if (from > to) return bad("Từ ngày không được lớn hơn đến ngày.");

        const wh = ["g.trang_thai='approved'", "g.loai_giao_dich!='OPENING'", "date(g.thoi_gian_gd) BETWEEN ? AND ?"];
        const binds = [from, to];

        if (type) {
          wh.push("g.loai_giao_dich = ?");
          binds.push(type);
        }
        if (q) {
          wh.push("(g.ma_hang LIKE ? OR d.ten_hang LIKE ? OR d.phan_loai LIKE ?)");
          const like = `%${q}%`;
          binds.push(like, like, like);
        }

        const rows = await env.DB.prepare(`
          SELECT
            d.ma_hang,
            d.ten_hang,
            d.phan_loai,
            d.don_vi_tinh,
            COALESCE(SUM(CASE WHEN g.loai_giao_dich='NHAP_KHO' THEN g.so_luong ELSE 0 END), 0) AS nhap,
            COALESCE(SUM(CASE WHEN g.loai_giao_dich IN ('XUAT_XUONG','XUAT_SAN_XUAT','YEU_CAU_XUONG') THEN g.so_luong ELSE 0 END), 0) AS xuat_sx,
            COALESCE(SUM(CASE WHEN g.loai_giao_dich IN ('TRA_KHO','HOAN_TRA_XUONG') THEN g.so_luong ELSE 0 END), 0) AS hoan_tra,
            COALESCE(SUM(CASE WHEN g.loai_giao_dich='NHAP_KHO' THEN g.khoi_luong_kg ELSE 0 END), 0) AS tong_kg_nhap,
            v.ton_kho,
            v.ton_xuong
          FROM Danh_Muc_NVL d
          JOIN Giao_Dich_NVL g ON g.ma_hang = d.ma_hang
          LEFT JOIN v_ton_nvl v ON v.ma_hang = d.ma_hang
          WHERE ${wh.join(" AND ")}
          GROUP BY d.ma_hang, d.ten_hang, d.phan_loai, d.don_vi_tinh
          ORDER BY d.phan_loai, d.ten_hang
        `).bind(...binds).all();

        const total = await env.DB.prepare(`
          SELECT
            COALESCE(SUM(CASE WHEN g.loai_giao_dich='NHAP_KHO' THEN g.so_luong ELSE 0 END), 0) AS nhap,
            COALESCE(SUM(CASE WHEN g.loai_giao_dich IN ('XUAT_XUONG','XUAT_SAN_XUAT','YEU_CAU_XUONG') THEN g.so_luong ELSE 0 END), 0) AS xuat_sx,
            COALESCE(SUM(CASE WHEN g.loai_giao_dich IN ('TRA_KHO','HOAN_TRA_XUONG') THEN g.so_luong ELSE 0 END), 0) AS hoan_tra,
            COALESCE(SUM(CASE WHEN g.loai_giao_dich='NHAP_KHO' THEN g.khoi_luong_kg ELSE 0 END), 0) AS tong_kg_nhap
          FROM Giao_Dich_NVL g
          JOIN Danh_Muc_NVL d ON d.ma_hang = g.ma_hang
          WHERE ${wh.join(" AND ")}
        `).bind(...binds).first();

        return json({ rows: rows.results || [], total: total || { nhap: 0, xuat_sx: 0, hoan_tra: 0, tong_kg_nhap: 0 } });
      }

      if (path === "/api/report/detail" && request.method === "GET") {
        const maHang = url.searchParams.get("ma_hang");
        const from = cleanDate(url.searchParams.get("from"));
        const to = cleanDate(url.searchParams.get("to"));

        const wh = ["g.trang_thai='approved'", "g.loai_giao_dich!='OPENING'"];
        const binds = [];

        if (maHang) { wh.push("g.ma_hang = ?"); binds.push(maHang); }
        if (from) { wh.push("date(g.thoi_gian_gd) >= ?"); binds.push(from); }
        if (to) { wh.push("date(g.thoi_gian_gd) <= ?"); binds.push(to); }

        const result = await env.DB.prepare(`
          SELECT g.id, g.thoi_gian_gd, g.ma_hang, d.ten_hang, d.phan_loai,
                 d.don_vi_tinh, g.so_luong, g.sl_phu, g.dvt_phu, g.quy_cach_gd, g.khoi_luong_kg,
                 g.loai_giao_dich, ${labelSql()} AS loai_hien_thi,
                 g.nguoi_giao_dich, u.ten_nv, g.ma_sp_sx, s.ten_sp,
                 g.so_chung_tu, g.ghi_chu
          FROM Giao_Dich_NVL g
          JOIN Danh_Muc_NVL d ON d.ma_hang=g.ma_hang
          LEFT JOIN Nguoi_Dung u ON u.ma_nv=g.nguoi_giao_dich
          LEFT JOIN San_Pham_SX s ON s.ma_sp=g.ma_sp_sx
          WHERE ${wh.join(" AND ")}
          ORDER BY g.thoi_gian_gd DESC, g.id DESC
          LIMIT 200
        `).bind(...binds).all();
        return json({ rows: result.results || [] });
      }

      if (path === "/api/health") return json({ ok: true, service: "Kho & San Xuat D1 Worker", version: "2.0" });
      return bad("API not found", 404);
    } catch (e) {
      return bad(e?.message || "Lỗi máy chủ.", 500);
    }
  },
};
```

4. Nhấn **Save and Deploy**. Worker sẽ có hiệu lực ngay lập tức trên toàn cầu.

---

## 3. KIỂM TRA CÁC TÍNH NĂNG MỚI TRÊN GIAO DIỆN

1. **Quản lý Nhà Cung Cấp**:
   - Đăng nhập tài khoản VP (`ADMIN` / `123`).
   - Vào tab **Nhà cung cấp**: Có ô tìm kiếm nhanh, nút **Sửa** (mở popup đổi tên, SĐT, địa chỉ, người liên hệ, email...) và nút **Xóa**.
   - Nếu xóa NCC đã có lịch sử nhập hàng -> Hệ thống tự động chuyển sang trạng thái Ẩn (`active = 0`) để không làm hỏng báo cáo và giao dịch cũ.

2. **Quản lý Mã Hàng Sản Xuất**:
   - Vào tab mới **Mã hàng SX**:
   - Bấm thêm mã mới (ví dụ: Tên `Trục dẫn hướng Ø20`, Quy cách `Dài 500mm`, ĐVT `Cái`).
   - Có thể Sửa, Xóa và Tìm kiếm mã sản phẩm.

3. **Xuất ra Line Sản Xuất**:
   - Vào tab **Giao dịch** -> **Tạo phiếu mới**.
   - Chọn loại giao dịch: **Xuất ra Line Sản Xuất**.
   - Ô **Sản phẩm xuất** sẽ tự động hiển thị danh sách các mã hàng sản xuất đang hoạt động (kèm quy cách kỹ thuật).
   - Khi lưu phiếu, lịch sử và báo cáo sẽ ghi nhận chính xác vật tư này được xuất để gia công cho sản phẩm nào.

