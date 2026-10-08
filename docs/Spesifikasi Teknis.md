# Spesifikasi Teknis JOKGER: Sistem Operasional Cafe Kopi

Dokumen ini mendefinisikan baseline produk siap deployment untuk satu outlet cafe. Semua nilai uang disimpan sebagai integer rupiah (`bigint`), tanpa tipe float.

---

## 1. Ruang Lingkup

**Termasuk:**
- Autentikasi dan otorisasi dua peran (admin, super admin)
- POS: menu, keranjang, pembayaran tunai dan transfer manual, riwayat transaksi, void, voucher, buka/tutup kasir
- Pesanan dengan status: semua, baru, diproses, menunggu diambil, dibatalkan, selesai
- Open bill: tambah item bertahap, bayar di akhir, struk keluar setelah bill ditutup
- Kode voucher kustom
- Payment manual: rekening bank dan e-wallet, input manual, verifikasi bukti transfer (screenshot)
- Inventory bahan baku, pergerakan stok (ledger), stok opname
- Cetak struk ke printer Bluetooth termal (ESC/POS)
- Pengaturan dan kustomisasi branding
- Laporan penjualan dasar, audit log, dan ekspor CSV

**Di luar cakupan baseline:**
- Payment gateway otomatis (Midtrans, Xendit, dan sejenisnya)
- Multi-outlet dan pemesanan online dari pelanggan
- Mode offline penuh (POS membutuhkan koneksi internet)
- Integrasi akuntansi pihak ketiga

---

## 2. Arsitektur Sistem

```
┌─────────────────────────────────────────────┐
│  Browser / PWA (SolidJS + Vite + TypeScript)│
│  - POS, Pesanan, Stok, Admin, Pengaturan    │
│  - Web Bluetooth (ESC/POS)                  │
└──────────────┬──────────────────────────────┘
               │ HTTPS
┌──────────────▼──────────────────────────────┐
│  Vercel                                     │
│  - Static hosting (SPA)                     │
│  - Serverless API route (operasi khusus     │
│    yang butuh service role key)             │
└──────────────┬──────────────────────────────┘
               │ supabase-js (anon key + JWT)
┌──────────────▼──────────────────────────────┐
│  Supabase                                   │
│  - PostgreSQL (tabel, RLS, RPC plpgsql)     │
│  - Auth (email + password)                  │
│  - Storage (bukti pembayaran, logo)         │
│  - Realtime (antrian pesanan)               │
└─────────────────────────────────────────────┘
```

**Prinsip utama:**
1. Logika yang harus atomik (pembuatan pesanan, pengurangan stok, voucher, penutupan open bill, verifikasi pembayaran) berada di fungsi PostgreSQL (RPC) dalam satu transaksi. Frontend tidak menulis langsung ke tabel tersebut.
2. Keamanan data dijamin oleh Row Level Security (RLS) di database, bukan hanya oleh UI.
3. Service role key tidak pernah dikirim ke browser. Hanya dipakai di serverless function Vercel.

---

## 3. Stack Teknologi

| Lapisan | Teknologi | Versi minimum |
|---|---|---|
| UI framework | SolidJS | 1.9+ |
| Router | @solidjs/router | 0.13+ |
| Build | Vite | 5+ |
| Bahasa | TypeScript (strict mode) | 5.4+ |
| Styling | Tailwind CSS | 3.4+ |
| State (global) | Solid stores + signals | bawaan |
| Data fetching | @supabase/supabase-js v2 | 2.4x+ |
| Validasi | Zod | 3.23+ |
| Form | Solid form utilities sendiri (berbasis Zod) | - |
| Ikon | lucide-solid | - |
| Tanggal | date-fns (zona waktu Asia/Jakarta) | 3+ |
| Hosting | Vercel | - |
| Backend | Supabase (PostgreSQL 15+) | - |
| Unit test | Vitest + @solidjs/testing-library | - |
| E2E test | Playwright | 1.45+ |
| DB test | pgTAP | - |
| Linting | ESLint + Prettier | - |
| Git hook | Husky + lint-staged | - |
| CI | GitHub Actions | - |

---

## 4. Struktur Repositori

```
jokger/
├─ apps/web/                     # Aplikasi SolidJS
│  ├─ src/
│  │  ├─ app/                    # Router, layout, guard
│  │  ├─ features/
│  │  │  ├─ auth/
│  │  │  ├─ pos/                 # Keranjang, pembayaran, void, voucher
│  │  │  ├─ shift/               # Buka/tutup kasir
│  │  │  ├─ orders/              # Daftar dan status pesanan
│  │  │  ├─ open-bill/
│  │  │  ├─ menu/                # Master menu dan modifier
│  │  │  ├─ inventory/           # Bahan, pergerakan, opname
│  │  │  ├─ vouchers/
│  │  │  ├─ payments/            # Rekening, verifikasi bukti
│  │  │  ├─ reports/
│  │  │  ├─ staff/               # Manajemen akun (super admin)
│  │  │  ├─ settings/            # Pengaturan dan branding
│  │  │  └─ printing/            # Bluetooth ESC/POS
│  │  ├─ shared/
│  │  │  ├─ ui/                  # Komponen dasar
│  │  │  ├─ lib/                 # supabase client, format rupiah, tanggal
│  │  │  ├─ types/               # Tipe hasil generate dari DB
│  │  │  └─ hooks/
│  │  └─ main.tsx
│  ├─ tests/
│  └─ vite.config.ts
├─ api/                          # Vercel serverless functions
│  └─ admin/create-staff.ts
├─ supabase/
│  ├─ migrations/                # SQL bertahap dan berurut
│  ├─ functions/                 # RPC plpgsql (atau dalam migrasi)
│  ├─ seed.sql
│  └─ tests/                     # pgTAP
├─ e2e/                          # Playwright
├─ .github/workflows/ci.yml
└─ package.json
```

---

## 5. Konfigurasi Lingkungan

| Variabel | Lokasi | Keterangan |
|---|---|---|
| `VITE_SUPABASE_URL` | Client | URL proyek Supabase |
| `VITE_SUPABASE_ANON_KEY` | Client | Anon key (aman dipublikasikan, dibatasi RLS) |
| `SUPABASE_URL` | Server (Vercel) | Sama dengan client |
| `SUPABASE_SERVICE_ROLE_KEY` | Server (Vercel) saja | Tidak boleh masuk bundle client |
| `APP_TIMEZONE` | Server | Nilai `Asia/Jakarta` |

Tiga lingkungan: `local` (Supabase CLI), `staging`, dan `production`. Setiap lingkungan memiliki proyek Supabase sendiri.

---

## 6. Model Data

### 6.1 Enum

```sql
create type role_type        as enum ('super_admin', 'admin');
create type order_type       as enum ('dine_in', 'takeaway');
create type order_status     as enum ('new', 'processing', 'ready', 'completed', 'cancelled');
create type bill_state       as enum ('open', 'closed');
create type payment_method   as enum ('cash', 'transfer', 'ewallet');
create type payment_status   as enum ('pending_verification', 'verified', 'rejected');
create type voucher_type     as enum ('percent', 'nominal');
create type movement_type    as enum ('purchase', 'sale', 'void_return', 'adjustment', 'opname', 'waste');
create type shift_status     as enum ('open', 'closed');
```

### 6.2 Identitas dan Pengaturan

```sql
create table profiles (
  id          uuid primary key references auth.users(id) on delete cascade,
  full_name   text not null,
  role        role_type not null default 'admin',
  is_active   boolean not null default true,
  created_at  timestamptz not null default now()
);

create table store_settings (
  id              int primary key default 1 check (id = 1),  -- satu baris saja
  store_name      text not null,
  address         text,
  phone           text,
  logo_path       text,
  primary_color   text not null default '#6F4E37',
  accent_color    text not null default '#F5E6D3',
  font_family     text not null default 'Inter',
  tax_percent     numeric(5,2) not null default 0,       -- PB1, 0 jika tidak dipungut
  service_percent numeric(5,2) not null default 0,
  rounding_rule   text not null default 'none' check (rounding_rule in ('none','up_100','nearest_100')),
  receipt_header  text,
  receipt_footer  text,
  paper_width_mm  int not null default 58 check (paper_width_mm in (58, 80)),
  updated_by      uuid references profiles(id),
  updated_at      timestamptz not null default now()
);

create table payment_accounts (
  id            uuid primary key default gen_random_uuid(),
  method        payment_method not null check (method in ('transfer','ewallet')),
  provider      text not null,            -- contoh: BCA, DANA, GoPay
  account_name  text not null,
  account_no    text not null,
  is_active     boolean not null default true,
  sort_order    int not null default 0
);

create table audit_logs (
  id          bigint generated always as identity primary key,
  actor_id    uuid references profiles(id),
  action      text not null,              -- contoh: 'order.void_item', 'voucher.create'
  entity      text not null,
  entity_id   text,
  payload     jsonb,
  created_at  timestamptz not null default now()
);
```

### 6.3 Menu

```sql
create table categories (
  id          uuid primary key default gen_random_uuid(),
  name        text not null unique,
  sort_order  int not null default 0,
  is_active   boolean not null default true
);

create table menu_items (
  id           uuid primary key default gen_random_uuid(),
  category_id  uuid not null references categories(id),
  name         text not null,
  description  text,
  price        bigint not null check (price >= 0),
  image_path   text,
  is_available boolean not null default true,   -- toggle habis hari ini
  is_active    boolean not null default true,   -- soft delete
  sort_order   int not null default 0,
  created_at   timestamptz not null default now(),
  updated_at   timestamptz not null default now()
);

create table modifier_groups (
  id            uuid primary key default gen_random_uuid(),
  name          text not null,              -- contoh: 'Level Gula'
  min_select    int not null default 0,
  max_select    int not null default 1
);

create table modifier_options (
  id          uuid primary key default gen_random_uuid(),
  group_id    uuid not null references modifier_groups(id) on delete cascade,
  name        text not null,
  extra_price bigint not null default 0,
  is_active   boolean not null default true
);

create table menu_item_modifier_groups (
  menu_item_id uuid references menu_items(id) on delete cascade,
  group_id     uuid references modifier_groups(id) on delete cascade,
  primary key (menu_item_id, group_id)
);
```

Harga menu hanya diubah lewat update langsung oleh admin. Harga item di pesanan selalu disalin (snapshot) ke `order_items`, sehingga perubahan harga tidak mengubah riwayat.

### 6.4 Inventory dan Resep

```sql
create table inventory_items (
  id            uuid primary key default gen_random_uuid(),
  name          text not null,
  unit          text not null,           -- gram, ml, pcs
  current_qty   numeric(14,3) not null default 0,
  min_qty       numeric(14,3) not null default 0,   -- alert stok menipis
  unit_cost     bigint not null default 0,
  is_active     boolean not null default true
);

-- Resep: berapa bahan dipakai per 1 porsi menu
create table recipe_lines (
  menu_item_id      uuid references menu_items(id) on delete cascade,
  inventory_item_id uuid references inventory_items(id),
  qty_per_serving   numeric(14,3) not null check (qty_per_serving > 0),
  primary key (menu_item_id, inventory_item_id)
);

-- Ledger: sumber kebenaran pergerakan stok
create table stock_movements (
  id                bigint generated always as identity primary key,
  inventory_item_id uuid not null references inventory_items(id),
  movement_type     movement_type not null,
  qty_change        numeric(14,3) not null,      -- positif = masuk, negatif = keluar
  reference_id      text,                        -- id order, opname, pembelian
  note              text,
  actor_id          uuid references profiles(id),
  created_at        timestamptz not null default now()
);

create table stock_opnames (
  id          uuid primary key default gen_random_uuid(),
  status      text not null default 'draft' check (status in ('draft','finalized')),
  opened_by   uuid references profiles(id),
  opened_at   timestamptz not null default now(),
  finalized_at timestamptz
);

create table stock_opname_lines (
  opname_id         uuid references stock_opnames(id) on delete cascade,
  inventory_item_id uuid references inventory_items(id),
  system_qty        numeric(14,3) not null,      -- disalin saat opname dibuka
  counted_qty       numeric(14,3),
  primary key (opname_id, inventory_item_id)
);
```

**Aturan stok:** `current_qty` adalah hasil agregasi `stock_movements`, disimpan sebagai cache. Setiap perubahan stok wajib membuat baris ledger dan memperbarui `current_qty` dalam transaksi yang sama. Selisih opname menjadi movement bertipe `opname`.

### 6.5 Shift Kasir

```sql
create table shifts (
  id              uuid primary key default gen_random_uuid(),
  status          shift_status not null default 'open',
  opened_by       uuid not null references profiles(id),
  opened_at       timestamptz not null default now(),
  opening_cash    bigint not null check (opening_cash >= 0),
  closed_by       uuid references profiles(id),
  closed_at       timestamptz,
  expected_cash   bigint,        -- opening + tunai masuk - tunai keluar
  actual_cash     bigint,        -- dihitung manual saat tutup
  difference      bigint,        -- actual - expected
  note            text
);
-- Hanya satu shift terbuka pada satu waktu
create unique index one_open_shift on shifts ((status)) where status = 'open';
```

### 6.6 Voucher

```sql
create table vouchers (
  id              uuid primary key default gen_random_uuid(),
  code            text not null unique,        -- disimpan uppercase
  name            text not null,
  type            voucher_type not null,
  value           bigint not null check (value > 0),   -- persen (1-100) atau nominal rupiah
  min_subtotal    bigint not null default 0,
  max_discount    bigint,                      -- batas untuk tipe persen
  valid_from      timestamptz not null,
  valid_until     timestamptz not null,
  total_quota     int,                         -- null = tidak terbatas
  used_count      int not null default 0,
  per_order_limit int not null default 1,
  is_active       boolean not null default true,
  created_by      uuid references profiles(id),
  constraint valid_range check (valid_until > valid_from),
  constraint percent_range check (type <> 'percent' or value between 1 and 100)
);
```

### 6.7 Pesanan, Open Bill, dan Pembayaran

```sql
create table orders (
  id              uuid primary key default gen_random_uuid(),
  order_no        text not null unique,       -- format: JKG-YYYYMMDD-0001, reset harian
  shift_id        uuid not null references shifts(id),
  order_type      order_type not null,
  status          order_status not null default 'new',
  bill_state      bill_state,                 -- null = pesanan biasa, terisi = open bill
  table_label     text,                       -- nomor meja atau nama pelanggan (opsional)
  customer_name   text,
  subtotal        bigint not null default 0,
  discount_total  bigint not null default 0,
  service_amount  bigint not null default 0,
  tax_amount      bigint not null default 0,
  rounding_amount bigint not null default 0,
  grand_total     bigint not null default 0,
  voucher_id      uuid references vouchers(id),
  voucher_code    text,                       -- snapshot kode saat dipakai
  cancel_reason   text,
  created_by      uuid not null references profiles(id),
  created_at      timestamptz not null default now(),
  updated_at      timestamptz not null default now(),
  closed_at       timestamptz
);

create table order_items (
  id              uuid primary key default gen_random_uuid(),
  order_id        uuid not null references orders(id) on delete cascade,
  menu_item_id    uuid not null references menu_items(id),
  item_name       text not null,              -- snapshot
  unit_price      bigint not null,            -- snapshot harga dasar
  modifiers       jsonb not null default '[]',-- snapshot: [{name, extra_price}]
  qty             int not null check (qty > 0),
  line_total      bigint not null,
  note            text,
  is_voided       boolean not null default false,
  void_reason     text,
  voided_by       uuid references profiles(id),
  voided_at       timestamptz,
  created_at      timestamptz not null default now()
);

create table payments (
  id              uuid primary key default gen_random_uuid(),
  order_id        uuid not null references orders(id),
  method          payment_method not null,
  payment_account_id uuid references payment_accounts(id),
  amount          bigint not null check (amount > 0),
  status          payment_status not null default 'pending_verification',
  proof_path      text,                       -- path di bucket payment-proofs
  reference_no    text,                       -- nomor referensi yang diinput manual
  note            text,
  verified_by     uuid references profiles(id),
  verified_at     timestamptz,
  created_at      timestamptz not null default now()
);

create table voucher_redemptions (
  id          uuid primary key default gen_random_uuid(),
  voucher_id  uuid not null references vouchers(id),
  order_id    uuid not null references orders(id),
  discount    bigint not null,
  created_at  timestamptz not null default now()
);
```

**Aturan pembayaran manual:**
- Tunai langsung berstatus `verified` oleh kasir yang menerima uang.
- Transfer dan e-wallet berstatus `pending_verification` setelah bukti diunggah. Kasir atau admin memverifikasi dengan mencocokkan nominal, nomor referensi, dan screenshot.
- Pesanan dengan pembayaran `pending_verification` tetap bisa diproses ke dapur, tetapi tidak bisa ditandai `completed` sebelum semua pembayaran `verified`. Kebijakan ini bisa dinonaktifkan di pengaturan.
- Pembayaran `rejected` tidak dihitung dalam total terbayar. Kasir bisa membuat pembayaran baru.

---

## 7. Keamanan dan Otorisasi

### 7.1 Matriks Hak Akses

| Fitur | admin | super_admin |
|---|:-:|:-:|
| Login dan POS | ✓ | ✓ |
| Buka/tutup shift | ✓ | ✓ |
| Buat, proses, batalkan pesanan | ✓ | ✓ |
| Void item | ✓ | ✓ |
| Open bill | ✓ | ✓ |
| Verifikasi pembayaran | ✓ | ✓ |
| Kelola menu dan harga | ✓ | ✓ |
| Kelola stok, opname | ✓ | ✓ |
| Kelola voucher | ✓ | ✓ |
| Lihat laporan | ✓ | ✓ |
| Kelola rekening pembayaran | ✓ | ✓ |
| Kelola akun staff (buat, nonaktifkan, ganti peran) | ✗ | ✓ |
| Pengaturan toko dan branding | ✗ | ✓ |
| Pengaturan pajak, layanan, pembulatan | ✗ | ✓ |
| Pengaturan struk dan ukuran kertas | ✗ | ✓ |
| Lihat audit log | ✗ | ✓ |

Peran `admin` memiliki seluruh hak operasional. Peran `super_admin` memiliki semua hak `admin` ditambah kustomisasi dan manajemen akun.

### 7.2 Helper Fungsi dan RLS

```sql
create or replace function current_role_type()
returns role_type language sql stable security definer set search_path = public as $$
  select role from profiles where id = auth.uid() and is_active
$$;

create or replace function is_staff()
returns boolean language sql stable security definer set search_path = public as $$
  select current_role_type() is not null
$$;

create or replace function is_super_admin()
returns boolean language sql stable security definer set search_path = public as $$
  select current_role_type() = 'super_admin'
$$;
```

Contoh kebijakan:

```sql
alter table orders enable row level security;

create policy orders_select on orders for select using (is_staff());
-- Penulisan pesanan hanya lewat RPC, jadi tidak ada policy insert/update/delete langsung

alter table store_settings enable row level security;
create policy settings_select on store_settings for select using (true);   -- branding dibaca publik
create policy settings_update on store_settings for update using (is_super_admin());
```

Setiap tabel diaktifkan RLS. Tabel operasional tidak memiliki akses tulis langsung dari client. Semua penulisan melalui RPC `security definer` yang memeriksa `is_staff()` atau `is_super_admin()` di awal fungsi.

### 7.3 Keamanan Lain

- **Autentikasi:** email dan password dengan Supabase Auth. Kebijakan password minimal 10 karakter. Sesi kedaluwarsa setelah 8 jam tanpa aktivitas.
- **Nonaktivasi staff:** `is_active = false` langsung memblokir akses karena helper membaca kolom ini.
- **Pembuatan akun staff:** dilakukan lewat `/api/admin/create-staff` di Vercel, yang memakai service role key, memeriksa bahwa pemanggil adalah super admin, lalu membuat user dan profil.
- **Bukti pembayaran:** bucket `payment-proofs` bersifat privat. Akses melalui signed URL berumur 10 menit. Ukuran maksimal 5 MB, format JPG, PNG, atau WEBP. Validasi MIME dilakukan di sisi server.
- **Logo dan gambar menu:** bucket `public-assets` bersifat publik untuk dibaca dan hanya bisa ditulis oleh staff.
- **Rate limit:** RPC verifikasi voucher dan login dibatasi melalui pengaturan Supabase.
- **Header keamanan Vercel:** `Content-Security-Policy` ketat, `X-Frame-Options: DENY`, `Strict-Transport-Security`, dan `Referrer-Policy: strict-origin-when-cross-origin`.
- **Input:** semua input divalidasi dengan Zod di client dan ulang di RPC dengan pengecekan constraint database.

---

## 8. Fungsi RPC (Logika Bisnis Atomik)

Setiap RPC dijalankan dalam satu transaksi. Jika satu langkah gagal, semua perubahan dibatalkan.

| RPC | Input utama | Efek |
|---|---|---|
| `open_shift` | `opening_cash` | Membuat shift baru. Gagal jika ada shift terbuka. |
| `close_shift` | `actual_cash` | Menghitung `expected_cash` dari pembayaran tunai `verified` dalam shift, menyimpan selisih, dan menutup shift. Gagal jika ada open bill yang masih terbuka di shift tersebut. |
| `create_order` | `order_type`, `items[]`, `voucher_code?`, `table_label?`, `bill_mode` | Membuat pesanan, menyalin harga, menghitung total, menerapkan voucher, mengurangi stok sesuai resep, dan memberi nomor pesanan. |
| `add_items_to_open_bill` | `order_id`, `items[]` | Menambah item ke open bill dan mengurangi stok. Gagal jika bill sudah `closed`. |
| `void_order_item` | `item_id`, `reason` | Menandai item `is_voided`, mengembalikan stok, menghitung ulang total, dan mencatat audit. |
| `cancel_order` | `order_id`, `reason` | Hanya untuk status `new` atau `processing`. Mengembalikan stok, membatalkan voucher, dan mengubah status menjadi `cancelled`. |
| `change_order_status` | `order_id`, `to_status` | Memvalidasi transisi status (lihat 9.2). |
| `apply_voucher` | `order_id`, `code` | Memvalidasi voucher dan menghitung ulang diskon. |
| `submit_payment` | `order_id`, `method`, `amount`, `payment_account_id`, `reference_no`, `proof_path?` | Membuat pembayaran. Tunai langsung `verified`. |
| `verify_payment` | `payment_id`, `approve`, `note?` | Mengubah status pembayaran. Saat approve, memeriksa apakah total terbayar sudah mencukupi. |
| `close_open_bill` | `order_id`, `payments[]` | Menutup open bill, membuat pembayaran (mendukung split payment), memberi nomor struk final, dan mengubah `bill_state` menjadi `closed`. |
| `finalize_stock_opname` | `opname_id` | Membuat movement `opname` untuk setiap selisih lalu menandai opname `finalized`. |
| `record_stock_movement` | `item_id`, `type`, `qty`, `note` | Untuk pembelian dan waste. Tidak boleh membuat stok negatif, kecuali di-override oleh super admin. |

### 8.1 Aturan Nomor Pesanan

Nomor pesanan dibentuk dari `JKG-` + tanggal `YYYYMMDD` (zona Asia/Jakarta) + nomor urut harian empat digit. Nomor urut diambil dari tabel `daily_sequences(day date primary key, last_no int)` dengan `insert ... on conflict ... do update ... returning` untuk mencegah duplikasi saat transaksi bersamaan.

### 8.2 Perhitungan Total

Urutan perhitungan tetap dan dilakukan di RPC:

1. `subtotal` = jumlah `line_total` item yang tidak di-void. `line_total` = `(unit_price + sum(modifier extra_price)) * qty`.
2. `discount_total` dari voucher, dihitung terhadap `subtotal`:
   - Tipe persen: `min(subtotal * value / 100, max_discount)` bila `max_discount` diisi.
   - Tipe nominal: `min(value, subtotal)`.
3. `base = subtotal - discount_total`
4. `service_amount = round(base * service_percent / 100)`
5. `tax_amount = round((base + service_amount) * tax_percent / 100)`
6. `pre_round = base + service_amount + tax_amount`
7. `rounding_amount` sesuai `rounding_rule`.
8. `grand_total = pre_round + rounding_amount`

Semua pembulatan menggunakan `round()` pada integer rupiah.

---

## 9. Alur Bisnis

### 9.1 Pesanan Biasa

1. Kasir membuka shift (wajib sebelum transaksi apa pun).
2. Kasir memilih item, modifier, dan catatan. Keranjang ada di state lokal.
3. Kasir opsional memasukkan kode voucher. Validasi dilakukan di server.
4. Kasir menekan "Buat Pesanan". RPC `create_order` dipanggil dengan `bill_mode = 'none'`.
5. Pembayaran:
   - Tunai: kasir memasukkan jumlah uang. Sistem menghitung kembalian. Pembayaran `verified`.
   - Transfer atau e-wallet: kasir memilih rekening, memasukkan nomor referensi, dan opsional mengunggah screenshot. Pembayaran `pending_verification`.
6. Pesanan masuk ke antrian dengan status `new`.
7. Status berubah: `new` → `processing` → `ready` (menunggu diambil) → `completed`.
8. Struk dicetak otomatis jika printer tersambung, atau bisa dicetak ulang dari riwayat.

### 9.2 Transisi Status Pesanan

| Dari | Ke | Syarat |
|---|---|---|
| `new` | `processing` | Selalu diizinkan |
| `new` | `cancelled` | Wajib `cancel_reason` |
| `processing` | `ready` | Selalu diizinkan |
| `processing` | `cancelled` | Wajib `cancel_reason` |
| `ready` | `completed` | Semua pembayaran `verified` atau pesanan dibayar tunai |
| `ready` | `processing` | Diizinkan untuk koreksi, dicatat di audit |
| `completed` | - | Terminal. Koreksi hanya lewat fitur retur oleh super admin. |
| `cancelled` | - | Terminal |

Tab antrian dapur menampilkan `new` dan `processing`. Tab "Menunggu Diambil" menampilkan `ready`. Tab "Dibatalkan" dan "Semua" tersedia di halaman pesanan.

### 9.3 Open Bill

1. Kasir membuat pesanan dengan `bill_mode = 'open'`. Pesanan dibuat dengan `status = 'new'`, `bill_state = 'open'`, dan total belum final.
2. Item tambahan ditambahkan lewat `add_items_to_open_bill`. Setiap penambahan memicu pengurangan stok dan pembaruan total. Status pesanan bisa tetap `processing`.
3. Saat pelanggan membayar, kasir menekan "Tutup Bill". Sistem menampilkan total final dengan voucher opsional.
4. `close_open_bill` menerima array pembayaran, mendukung split payment (misalnya sebagian tunai dan sebagian transfer). Jumlah pembayaran harus sama dengan `grand_total`.
5. Setelah ditutup, `bill_state = 'closed'`, nomor struk final dibuat, dan struk dicetak. Pesanan tidak bisa ditambah lagi.
6. Open bill tidak boleh melewati penutupan shift. `close_shift` menolak jika ada open bill terbuka.

### 9.4 Void

- **Void item:** hanya untuk pesanan yang belum `completed`. Wajib alasan. Stok item dikembalikan. Total dihitung ulang. Tercatat di audit.
- **Pembatalan pesanan:** hanya untuk `new` atau `processing`. Jika sudah ada pembayaran `verified`, sistem membuat refund otomatis dalam bentuk catatan pembayaran negatif dan kasir harus mencatat pengembalian uang di luar sistem.
- **Retur setelah selesai:** hanya super admin. Menggunakan alasan wajib dan membuat audit khusus.

### 9.5 Buka dan Tutup Kasir

- **Buka kasir:** input saldo awal tunai. Shift baru dibuat dengan status `open`.
- **Tutup kasir:** sistem menampilkan ringkasan (jumlah pesanan, total per metode, tunai yang diharapkan). Kasir memasukkan tunai aktual. Selisih dihitung dan disimpan. Shift ditutup. Ringkasan shift bisa dicetak.
- Laporan harian dihitung dari seluruh shift pada tanggal tersebut.

### 9.6 Stok Opname

1. Admin membuka opname. Sistem menyalin `current_qty` setiap bahan ke `system_qty`.
2. Admin mengisi `counted_qty` untuk setiap bahan. Data disimpan sementara dan bisa dilanjutkan nanti.
3. Sistem menampilkan daftar selisih.
4. Admin memfinalisasi. `finalize_stock_opname` membuat movement `opname` sebesar `counted - system` untuk setiap baris yang berbeda, lalu mengunci opname.
5. Opname yang sudah finalisasi tidak bisa diubah. Koreksi hanya dengan opname baru.

### 9.7 Voucher

Validasi dilakukan dalam `apply_voucher` dengan urutan berikut. Kegagalan pertama menghentikan proses dan mengembalikan pesan spesifik:

1. Kode ditemukan (dibandingkan dalam uppercase).
2. `is_active = true`.
3. Waktu sekarang berada di antara `valid_from` dan `valid_until`.
4. `used_count < total_quota` jika kuota diisi.
5. Jumlah pemakaian voucher ini pada pesanan tidak melebihi `per_order_limit`. Dalam praktik, nilai maksimal satu voucher per pesanan.
6. `subtotal >= min_subtotal`.

Voucher ditandai terpakai saat pesanan dibuat (bukan saat dibuat kodenya). Jika pesanan dibatalkan, `used_count` dikurangi dan entri `voucher_redemptions` dihapus.

Super admin dan admin bisa membuat, mengedit, menonaktifkan, dan melihat laporan pemakaian voucher. Voucher yang sudah pernah dipakai tidak bisa dihapus, hanya dinonaktifkan.

---

## 10. Pencetakan Struk Bluetooth

### 10.1 Pendekatan

Pencetakan menggunakan **Web Bluetooth API** dan perintah **ESC/POS** langsung ke printer termal. Tidak ada aplikasi perantara.

### 10.2 Batasan Platform yang Wajib Diketahui

- Web Bluetooth hanya didukung di Chrome dan Edge pada Android, Windows, macOS, dan ChromeOS.
- iOS (Safari dan semua browser di iOS) **tidak mendukung** Web Bluetooth. Untuk perangkat iOS, sistem menyediakan fallback cetak melalui `window.print()` dengan stylesheet struk, yang membutuhkan AirPrint atau printer lain yang disetujui pengguna.
- Halaman harus dibuka melalui HTTPS dan pengguna harus memberi izin per sesi.

### 10.3 Alur Teknis

1. Pengguna menekan "Hubungkan Printer" di pengaturan. Sistem memanggil `navigator.bluetooth.requestDevice` dengan filter service UUID printer umum (ESC/POS) atau `acceptAllDevices` sebagai fallback.
2. Karakteristik write dengan properti `writeWithoutResponse` atau `write` dicari dari GATT service.
3. Koneksi disimpan di memori untuk sesi berjalan. Pengaturan menyimpan nama perangkat sebagai referensi.
4. Pembuatan data cetak: fungsi `buildReceipt(order, settings)` menghasilkan array byte ESC/POS berdasarkan lebar kertas (384 dot untuk 58 mm, 576 dot untuk 80 mm).
5. Data dipecah menjadi potongan maksimal 100 byte (sesuai MTU umum BLE) dan dikirim berurutan dengan jeda kecil.
6. Logo dicetak dalam format raster. Gambar dikonversi ke monokrom di canvas sebelum dikirim.

### 10.4 Format Struk

Struk berisi, secara berurutan: logo, nama toko, alamat, nomor telepon, header kustom, garis pemisah, nomor pesanan, tanggal dan jam, meja atau nama pelanggan, daftar item dengan modifier dan jumlah, subtotal, diskon voucher (dengan kode), layanan, pajak, pembulatan, total, rincian pembayaran, kembalian, footer kustom, dan kode QR opsional mengarah ke nomor pesanan.

### 10.5 Penanganan Kegagalan

- Jika koneksi terputus saat cetak, sistem menampilkan notifikasi dan menyimpan data cetak dalam antrian lokal. Kasir bisa mencoba ulang.
- Cetak ulang selalu tersedia dari halaman detail pesanan dan menandai salinan sebagai "REPRINT".
- Pencetakan tidak pernah memblokir penyimpanan transaksi. Transaksi sudah tersimpan di database sebelum cetak dimulai.

---

## 11. Kustomisasi dan Branding

Hanya super admin yang bisa mengubah. Perubahan langsung terlihat di seluruh aplikasi melalui pembacaan `store_settings` saat aplikasi dimuat dan melalui realtime subscription.

- **Identitas:** nama toko, alamat, telepon, logo (PNG atau SVG, maks 1 MB, disimpan di storage).
- **Warna:** warna utama dan aksen. Sistem menghasilkan variasi lain (terang, gelap, teks kontras) secara otomatis dan memeriksa rasio kontras WCAG AA sebelum menyimpan.
- **Font:** pilihan terbatas (Inter, Poppins, Plus Jakarta Sans, dan sans-serif sistem) agar konsisten dan cepat dimuat.
- **Pajak dan layanan:** persentase PB1 dan biaya layanan, serta aturan pembulatan.
- **Struk:** header, footer, dan ukuran kertas (58 mm atau 80 mm).
- **Pembayaran:** daftar rekening dan e-wallet, status aktif, urutan tampil. Kebijakan "boleh proses pesanan sebelum pembayaran verified" bisa diatur.
- **Operasional:** jam buka (untuk tampilan), dan opsi wajib-verifikasi sebelum selesai.
- **Staff:** buat akun, nonaktifkan, ubah peran, reset password melalui email Supabase.

Tema diterapkan lewat CSS custom properties di `:root`, sehingga perubahan warna tidak memerlukan build ulang.

---

## 12. Modul Pendukung

- **Riwayat transaksi:** tabel dengan filter tanggal, status, metode pembayaran, kasir, dan pencarian nomor pesanan. Klik membuka detail lengkap termasuk item, void, pembayaran, bukti, dan audit.
- **Laporan:** penjualan per hari, per jam, per kategori, per item, dan per metode pembayaran. Ringkasan untuk rentang tanggal. Ekspor CSV. Grafik dibuat dengan komponen SVG sederhana tanpa library berat.
- **Audit log:** daftar aktivitas penting (void, retur, perubahan harga, perubahan voucher, perubahan pengaturan, verifikasi pembayaran). Hanya dapat dilihat super admin.
- **Notifikasi stok menipis:** badge di menu inventory saat `current_qty <= min_qty`.
- **Pencarian global:** pencarian menu, pesanan, dan pelanggan dari satu input.
- **Realtime:** antrian pesanan diperbarui otomatis melalui Supabase Realtime pada tabel `orders` dan `payments`. Subscription dibatasi dengan filter `status`.
- **PWA:** manifest dan service worker untuk caching aset statis. Data transaksi tidak di-cache.
- **Keyboard shortcut** di POS untuk input cepat (angka untuk qty, F-key untuk aksi utama).

---

## 13. Standar Kualitas Kode

### 13.1 Aturan Kode

- TypeScript `strict: true`, `noUncheckedIndexedAccess: true`, tidak ada `any` yang tidak dijustifikasi.
- Tipe database dihasilkan otomatis dengan `supabase gen types` dan dicek di CI. Perbedaan tipe menggagalkan build.
- Satu komponen per file. Logika bisnis dipisahkan ke fungsi murni di `lib/` agar mudah diuji.
- Penamaan: komponen PascalCase, fungsi dan variabel camelCase, konstanta SCREAMING_SNAKE_CASE, tabel dan kolom snake_case.
- Tidak ada `console.log` di production. Error dicatat melalui satu modul logger.
- Semua string UI menggunakan satu file terpusat untuk memudahkan kustomisasi dan penerjemahan.

### 13.2 Format dan Linting

- Prettier untuk format. ESLint dengan `eslint-plugin-solid` dan `@typescript-eslint`.
- Husky dan lint-staged menjalankan format, lint, dan type-check pada file yang berubah sebelum commit.
- Commit mengikuti Conventional Commits.

### 13.3 Pengujian

| Jenis | Alat | Cakupan minimum | Yang diuji |
|---|---|---|---|
| Unit | Vitest | 85% baris pada `lib/` | Perhitungan total, voucher, format rupiah, builder ESC/POS, validasi transisi status |
| Komponen | @solidjs/testing-library | Semua komponen form dan tabel utama | Interaksi, validasi, state loading dan error |
| Database | pgTAP | Semua RPC | Nilai kembali, error case, RLS per peran, atomisitas (rollback saat gagal) |
| E2E | Playwright | Alur kritis | Login, buka shift, pesanan tunai, pesanan transfer dengan verifikasi, open bill dengan split payment, void, tutup shift, opname, buat voucher, cetak ke mock printer |
| Aksesibilitas | axe-core melalui Playwright | Semua halaman | Tidak ada pelanggaran serius atau kritis |
| Performa | Lighthouse CI | Halaman POS | Skor performa minimum 85 di mobile |

**Data uji:** seed terpisah untuk lingkungan test. Setiap test database berjalan dalam transaksi yang di-rollback.

**Mock printer:** implementasi `PrinterAdapter` dengan dua versi, Web Bluetooth asli dan mock yang menyimpan byte ke memori untuk verifikasi isi struk.

### 13.4 Kriteria Merge

Sebuah perubahan hanya bisa di-merge jika:
- Semua test unit, komponen, database, dan E2E lulus.
- Type-check dan lint bersih.
- Tidak ada penurunan cakupan di bawah ambang.
- Perubahan skema disertai migrasi baru. Migrasi lama tidak diubah.
- Setidaknya satu review.

---

## 14. CI/CD dan Deployment

### 14.1 Pipeline GitHub Actions

Pipeline dijalankan pada setiap pull request dan push ke `main`:

1. Install dependensi dengan lockfile terkunci.
2. Lint, format check, dan type-check.
3. Unit dan komponen test dengan laporan cakupan.
4. Start Supabase lokal, jalankan semua migrasi dari nol, jalankan pgTAP.
5. Build aplikasi.
6. Jalankan Playwright melawan build lokal dengan database lokal.
7. Lighthouse CI.
8. Pemindaian dependensi (`npm audit` dengan ambang high) dan secret scanning.

### 14.2 Alur Rilis

- Pull request → preview deployment Vercel dengan proyek Supabase staging.
- Merge ke `main` → deploy ke staging secara otomatis.
- Tag `v*.*.*` → deploy ke production setelah persetujuan manual di environment GitHub.
- Migrasi database dijalankan sebelum deploy aplikasi, dengan `supabase db push`. Migrasi harus backward-compatible untuk satu versi aplikasi sebelumnya (pola expand lalu contract).
- Rollback aplikasi: redeploy versi sebelumnya dari Vercel. Rollback database: migrasi perbaikan baru, bukan revert manual.

### 14.3 Konfigurasi Vercel

- Framework preset: Vite. Output: `dist`.
- Serverless functions di `/api` dengan runtime Node.js 20.
- Header keamanan didefinisikan di `vercel.json`.
- Environment variable diisi berbeda untuk Preview, Staging, dan Production.

### 14.4 Backup dan Pemulihan

- Supabase point-in-time recovery aktif pada production.
- Backup logis harian ke bucket terpisah.
- Target pemulihan: RPO maksimal 5 menit, RTO maksimal 2 jam.
- Prosedur pemulihan diuji minimal satu kali sebelum go-live.

---

## 15. Observabilitas

- **Error tracking:** Sentry untuk frontend dan serverless function. Data pribadi dan token disaring sebelum dikirim.
- **Log database:** log RPC gagal disimpan di tabel `error_logs` dengan konteks (fungsi, actor, timestamp) untuk investigasi.
- **Metrik:** Vercel Analytics untuk performa halaman. Supabase dashboard untuk query lambat dan penggunaan koneksi.
- **Alert:** notifikasi ke channel tim saat error rate frontend melebihi 2% dalam 15 menit, atau saat RPC gagal lebih dari 10 kali berturut-turut.

---

## 16. Persyaratan Non-Fungsional

| Aspek | Target |
|---|---|
| Waktu respons RPC transaksi | p95 di bawah 400 ms |
| Waktu muat POS pertama | Di bawah 2 detik pada 4G menengah |
| Ketersediaan | 99,5% bulanan (bergantung pada Supabase dan Vercel) |
| Konkurensi | Mendukung 5 kasir dan 5 admin aktif bersamaan per outlet |
| Ukuran data | Dirancang untuk 500 ribu baris pesanan per tahun tanpa penurunan berarti |
| Retensi data | Transaksi disimpan minimal 10 tahun. Log audit tidak dihapus. |
| Bahasa | Bahasa Indonesia. Format tanggal dan angka mengikuti lokal id-ID. |
| Perangkat | Desktop, tablet, dan ponsel. POS dioptimalkan untuk tablet landscape dan layar 10 inci ke atas. |
| Browser | Chrome, Edge, Safari, Firefox versi terbaru dua. Fitur Bluetooth dikhususkan untuk Chromium. |
| Aksesibilitas | WCAG 2.1 AA untuk kontras dan navigasi keyboard. |

**Konkurensi stok:** pengurangan stok dilakukan dengan `select ... for update` pada baris `inventory_items` di dalam RPC, sehingga dua pesanan bersamaan tidak mengurangi stok melebihi ketersediaan (kecuali bahan dengan `current_qty` negatif yang diizinkan oleh super admin).

---

## 17. Penanganan Error dan Kondisi Batas

- **Koneksi internet terputus:** POS menampilkan banner peringatan dan menonaktifkan tombol yang membutuhkan server. Keranjang tetap di memori lokal dan bisa dikirim ulang.
- **Dua kasir menutup bill yang sama:** RPC menggunakan `for update` pada baris `orders`. Yang kedua menerima pesan "Bill sudah ditutup".
- **Voucher habis kuota saat checkout:** validasi ulang di RPC. Pesanan gagal dengan pesan jelas dan keranjang tidak hilang.
- **Stok tidak cukup:** dicek saat membuat pesanan. Jika tidak cukup, item ditandai dan kasir bisa menghapusnya atau mengubah resep.
- **Printer tidak tersedia:** pesanan tetap tersimpan. Cetak masuk antrian.
- **Screenshot pembayaran gagal diunggah:** pembayaran tetap dibuat tanpa bukti, dengan status `pending_verification` dan penanda "bukti belum dilampirkan".
- **Pengguna nonaktif sedang login:** sesi ditolak pada request berikutnya karena helper RLS membaca `is_active`.
- **Pergantian tanggal saat shift terbuka:** shift tetap milik tanggal sebelumnya. Nomor pesanan memakai tanggal waktu pembuatan pesanan.
- **Perubahan harga saat ada keranjang:** harga dikunci saat `create_order`. Keranjang menampilkan peringatan jika harga berubah sebelum checkout.

---

## 18. Kriteria Penerimaan (Acceptance)

Produk dinyatakan siap deployment jika seluruh poin berikut terpenuhi:

1. Admin dan super admin dapat login. Admin tidak dapat mengakses halaman pengaturan toko, staff, atau audit log, baik dari UI maupun dengan memanggil API langsung.
2. Kasir tidak dapat membuat pesanan sebelum membuka shift.
3. Pesanan tunai selesai dari pilihan item hingga struk dalam satu alur.
4. Pesanan transfer dapat dibuat dengan bukti, diverifikasi, lalu diselesaikan. Pesanan dengan pembayaran belum verified tidak bisa selesai jika kebijakan aktif.
5. Open bill dapat dibuka, ditambah item beberapa kali, ditutup dengan split payment, dan struk hanya keluar setelah penutupan.
6. Voucher persen dan nominal dapat dibuat dan diterapkan dengan batas kuota, tanggal, dan minimum belanja yang benar.
7. Void item dan pembatalan pesanan mengembalikan stok dan total yang benar.
8. Stok berkurang sesuai resep saat pesanan dibuat dan kembali saat dibatalkan atau di-void.
9. Stok opname menghasilkan movement selisih yang tepat dan tidak bisa diubah setelah finalisasi.
10. Tutup kasir menampilkan selisih tunai yang benar.
11. Struk tercetak dengan benar di printer termal 58 mm dan 80 mm pada Chrome Android dan Windows.
12. Perubahan branding, logo, warna, dan pengaturan pajak langsung terlihat di seluruh aplikasi.
13. Semua RPC lulus pgTAP, termasuk uji RLS untuk setiap peran.
14. Semua alur kritis lulus Playwright di CI.
15. Tidak ada secret di bundle client. Verifikasi dilakukan oleh pipeline.
16. Migrasi dapat dijalankan dari database kosong tanpa error.
17. Dokumentasi kode berupa komentar pada RPC dan tipe publik tersedia, sesuai permintaan untuk tidak membuat dokumen terpisah.
