# Spesifikasi Teknis Frontend UI/UX JOKGER

Dokumen ini menjabarkan lapisan frontend sistem JOKGER: arsitektur aplikasi SolidJS, sistem desain, struktur layar, interaksi, aksesibilitas, dan perilaku responsif. Semua acuan data, RPC, dan peran mengikuti spesifikasi sistem sebelumnya.

---

## 1. Prinsip Desain

1. **Kecepatan kasir di atas segalanya.** Alur transaksi utama (pilih item, bayar, cetak) harus selesai dengan jumlah ketukan minimum. Target: pesanan tunai sederhana selesai dalam 6 ketukan atau kurang.
2. **Satu layar, satu tujuan.** Setiap halaman memiliki satu tugas utama. Tindakan sekunder ditempatkan di menu atau panel samping.
3. **Kesalahan dicegah sebelum terjadi.** Aksi destruktif (void, batal, retur, finalisasi opname) selalu memerlukan konfirmasi dengan alasan wajib.
4. **Status selalu terlihat.** Status shift, koneksi, printer, dan sinkronisasi ditampilkan terus-menerus di header.
5. **Angka selalu rapi.** Semua nilai rupiah menggunakan format `Rp 12.500`, rata kanan, dengan angka tabular agar kolom sejajar.
6. **Kontras dan keterbacaan di lingkungan cafe.** Layar tablet sering terkena cahaya terang, sehingga kontras minimum WCAG AA dan ukuran target sentuh minimum 44×44 px.

---

## 2. Arsitektur Frontend

### 2.1 Komposisi Aplikasi

```
apps/web/src/
├─ app/
│  ├─ App.tsx                 # Provider global, router root
│  ├─ routes.tsx              # Definisi rute dan guard
│  ├─ layouts/
│  │  ├─ AppShell.tsx         # Header, sidebar, area konten
│  │  └─ PosLayout.tsx        # Layout fullscreen untuk POS
│  └─ guards/
│     ├─ RequireAuth.tsx
│     └─ RequireRole.tsx
├─ features/                  # Modul per domain (lihat 2.3)
├─ shared/
│  ├─ ui/                     # Komponen dasar (lihat 4)
│  ├─ theme/                  # Token dan penerapan tema
│  ├─ lib/                    # Format, tanggal, supabase client
│  ├─ hooks/                  # useRealtime, useDebounce, useShortcut
│  └─ stores/                 # Store global (sesi, shift, toko, koneksi)
└─ main.tsx
```

### 2.2 Routing

| Path | Komponen | Peran | Layout |
|---|---|---|---|
| `/login` | LoginPage | publik | Kosong |
| `/pos` | PosPage | admin, super_admin | PosLayout |
| `/pos/open-bill/:id` | OpenBillPage | admin, super_admin | PosLayout |
| `/orders` | OrdersPage | admin, super_admin | AppShell |
| `/orders/:id` | OrderDetailPage | admin, super_admin | AppShell |
| `/history` | TransactionHistoryPage | admin, super_admin | AppShell |
| `/shift` | ShiftPage | admin, super_admin | AppShell |
| `/menu` | MenuPage | admin, super_admin | AppShell |
| `/inventory` | InventoryPage | admin, super_admin | AppShell |
| `/inventory/opname` | StockOpnamePage | admin, super_admin | AppShell |
| `/inventory/opname/:id` | StockOpnameDetailPage | admin, super_admin | AppShell |
| `/vouchers` | VouchersPage | admin, super_admin | AppShell |
| `/payments` | PaymentAccountsPage | admin, super_admin | AppShell |
| `/payments/verify` | PaymentVerificationPage | admin, super_admin | AppShell |
| `/reports` | ReportsPage | admin, super_admin | AppShell |
| `/settings` | SettingsPage | super_admin | AppShell |
| `/settings/branding` | BrandingPage | super_admin | AppShell |
| `/settings/staff` | StaffPage | super_admin | AppShell |
| `/settings/printer` | PrinterSettingsPage | super_admin | AppShell |
| `/audit` | AuditLogPage | super_admin | AppShell |

**Guard:** `RequireAuth` memeriksa sesi Supabase. `RequireRole` membaca `profiles.role` dari store sesi. Jika peran tidak sesuai, halaman menampilkan 403 dan tidak memuat data apa pun. Pemeriksaan ini hanya untuk UX; keamanan tetap dijamin RLS.

**Navigasi:** rute menggunakan `@solidjs/router` dengan `lazy` per halaman. Setiap halaman dipecah menjadi chunk tersendiri agar POS memuat cepat.

**Redirect:** setelah login, pengguna diarahkan ke `/pos` jika shift terbuka, atau ke `/shift` jika belum ada shift terbuka dan peran adalah admin atau super admin.

### 2.3 Struktur Fitur

Setiap folder fitur berisi:

```
features/pos/
├─ components/          # Komponen khusus fitur
├─ state/               # Store lokal fitur (keranjang, pilihan)
├─ api/                 # Pemanggilan RPC dan query (wrapper typed)
├─ schemas/             # Skema Zod untuk input form
├─ logic/               # Fungsi murni (tanpa efek samping)
└─ index.ts             # Ekspor publik fitur
```

Komponen di luar fitur tidak boleh mengimpor dari `components/` fitur lain. Komunikasi antarfitur lewat store global atau props.

### 2.4 Manajemen State

| Jenis state | Solusi | Contoh |
|---|---|---|
| Sesi pengguna | Store global `session` | user, role, is_active |
| Shift aktif | Store global `shift` | id, status, opening_cash |
| Pengaturan toko | Store global `settings` | branding, pajak, struk |
| Koneksi | Signal global `online` | status jaringan dan Supabase |
| Keranjang POS | Store lokal `cart` | item, modifier, voucher |
| Data daftar | Resource Solid (`createResource`) dengan key | daftar pesanan per filter |
| Status realtime | Subscription di `onMount`, dibersihkan di `onCleanup` | antrian pesanan |
| Form | Signal per field, validasi Zod saat blur dan submit | form menu |

**Aturan state:**
- Data server tidak disalin ke store global kecuali bersifat global (sesi, shift, settings). Data lain dibaca melalui resource dengan key agar cache ikut invalidasi.
- Mutasi setelah RPC selalu memanggil `refetch` pada resource terkait. Tidak ada update optimistik untuk transaksi uang. Update optimistik hanya untuk toggle sederhana seperti `is_available`, dengan rollback jika gagal.
- Keranjang disimpan di `localStorage` dengan kunci `jokger.cart.v1` agar tidak hilang saat halaman di-refresh. Keranjang dihapus setelah pesanan berhasil dibuat.

### 2.5 Pola Data Fetching

- Semua pemanggilan RPC dibungkus fungsi typed di `api/`, yang mengembalikan `Result<T, AppError>`. Tidak ada `try/catch` tersebar di komponen.
- `AppError` memiliki `code` (misalnya `SHIFT_NOT_OPEN`, `VOUCHER_QUOTA_EXCEEDED`, `STOCK_INSUFFICIENT`, `BILL_CLOSED`) yang dipetakan ke pesan Bahasa Indonesia di file terpusat.
- Timeout request 15 detik. Setelah timeout, tombol aksi kembali aktif dengan pesan "Koneksi lambat, coba lagi".
- Realtime hanya untuk antrian pesanan dan status pembayaran. Halaman lain memakai refetch saat fokus jendela kembali.

---

## 3. Sistem Desain

### 3.1 Token Warna

Warna didefinisikan sebagai CSS custom properties di `:root`. Nilai default diturunkan dari `primary_color` dan `accent_color` di `store_settings`.

| Token | Fungsi | Default |
|---|---|---|
| `--color-brand` | Warna utama, tombol primer | `#6F4E37` |
| `--color-brand-hover` | Hover tombol primer | turunan 8% lebih gelap |
| `--color-brand-contrast` | Teks di atas brand | `#FFFFFF` atau `#1B1410` otomatis |
| `--color-accent` | Latar kartu dan highlight lembut | `#F5E6D3` |
| `--color-bg` | Latar halaman | `#FAF7F3` (terang) / `#17120E` (gelap) |
| `--color-surface` | Kartu dan panel | `#FFFFFF` / `#221B16` |
| `--color-surface-raised` | Modal dan dropdown | `#FFFFFF` / `#2B231D` |
| `--color-border` | Garis pemisah | `#E7DED4` / `#3A3029` |
| `--color-text` | Teks utama | `#1F1813` / `#F3ECE4` |
| `--color-text-muted` | Teks sekunder | `#6B5D52` / `#B5A89B` |
| `--color-success` | Verified, selesai | `#2E7D4F` |
| `--color-warning` | Pending, menipis | `#B26A00` |
| `--color-danger` | Void, batal, selisih | `#B42318` |
| `--color-info` | Diproses | `#1D5FA8` |

**Aturan kontras:** setiap kombinasi teks dan latar wajib memenuhi rasio minimum 4,5:1 untuk teks normal dan 3:1 untuk teks besar. Saat super admin menyimpan warna, validasi kontras dijalankan dan penyimpanan ditolak jika gagal.

**Status pesanan:** warna status tidak boleh menjadi satu-satunya pembeda. Setiap status memiliki ikon dan label teks.

| Status | Label | Warna | Ikon |
|---|---|---|---|
| new | Baru | info | circle-dot |
| processing | Diproses | warning | flame |
| ready | Menunggu diambil | brand | bell |
| completed | Selesai | success | check |
| cancelled | Dibatalkan | danger | x |

### 3.2 Tipografi

| Peran | Ukuran | Berat | Tinggi baris |
|---|---|---|---|
| Judul halaman | 24 px (mobile 20 px) | 700 | 1.25 |
| Judul panel | 18 px | 600 | 1.3 |
| Label tombol | 16 px (POS 18 px) | 600 | 1.2 |
| Isi | 15 px | 400 | 1.5 |
| Metadata, caption | 13 px | 500 | 1.4 |
| Total besar (POS) | 32 px | 700 | 1.1 |

Font default Inter. Angka rupiah dan nomor pesanan memakai `font-variant-numeric: tabular-nums`. Font lain (Poppins, Plus Jakarta Sans) dimuat hanya jika dipilih di branding, dengan `font-display: swap`.

### 3.3 Spasi, Radius, dan Elevasi

- **Skala spasi:** 4, 8, 12, 16, 24, 32, 48 px. Tidak ada nilai di luar skala.
- **Radius:** kontrol 10 px, kartu 14 px, modal 18 px, badge 999 px.
- **Elevasi:** tiga level bayangan lembut. Pada mode gelap, elevasi diwakili oleh perbedaan warna permukaan, bukan bayangan.
- **Kepadatan:** mode normal (default) dan mode padat (untuk layar kecil dan daftar panjang). Dipilih di pengaturan pengguna.

### 3.4 Ikon

Menggunakan `lucide-solid` dengan ukuran 16, 20, dan 24 px. Setiap ikon yang berdiri sendiri wajib memiliki `aria-label`. Ikon dekoratif memakai `aria-hidden="true"`.

### 3.5 Tema Terang dan Gelap

- Mengikuti `prefers-color-scheme` pada perangkat secara default.
- Pengguna bisa memaksa terang atau gelap di menu profil. Pilihan disimpan per perangkat.
- POS mendukung mode gelap penuh untuk kasir yang bekerja di ruangan redup.
- Gambar logo tetap menggunakan versi yang kontras di kedua tema. Jika hanya ada satu versi, sistem memberi latar putih kecil di belakang logo.

---

## 4. Komponen Dasar (`shared/ui`)

| Komponen | Varian | Catatan |
|---|---|---|
| `Button` | primary, secondary, ghost, danger; ukuran sm, md, lg | Status loading menggantikan label dengan spinner dan menjaga lebar agar tidak bergeser. Tinggi minimum 44 px. |
| `IconButton` | sama dengan Button | Wajib `aria-label`. |
| `Input` | text, number, password, search | Label selalu terlihat (tidak hanya placeholder). Pesan error di bawah field dengan `aria-describedby`. |
| `CurrencyInput` | - | Hanya angka. Format ribuan saat fokus hilang. Keyboard numerik di mobile (`inputmode="numeric"`). |
| `Select` | - | Native select untuk daftar pendek; combobox dengan pencarian untuk daftar panjang (menu, pelanggan). |
| `Switch` | - | Label di samping, bukan di dalam. |
| `Checkbox`, `RadioGroup` | - | Area klik minimal 44 px. |
| `Textarea` | - | Penghitung karakter jika ada batas. |
| `DatePicker` | single, range | Preset: hari ini, kemarin, 7 hari, bulan ini. Zona Asia/Jakarta. |
| `Modal` | sm, md, lg, fullscreen | Fokus terjebak di dalam modal. Escape menutup kecuali modal konfirmasi destruktif. Kembali ke elemen pemicu saat ditutup. |
| `ConfirmDialog` | default, destructive | Tombol destruktif tidak menjadi fokus awal. Untuk aksi dengan alasan, menampilkan textarea wajib. |
| `Drawer` | kanan, bawah | Untuk detail item, modifier, dan pembayaran di mobile. |
| `Toast` | success, error, info, warning | Durasi 4 detik, dapat ditutup. Error tidak otomatis hilang. Diumumkan lewat `aria-live="polite"`, error memakai `assertive`. |
| `Table` | - | Header sticky, kolom dapat diurutkan, baris dapat dipilih. Pada layar sempit berubah menjadi daftar kartu. |
| `Badge` | status, neutral, count | Selalu disertai teks. |
| `Tabs` | - | Navigasi dengan panah kiri/kanan. |
| `Card` | - | Latar `--color-surface`. |
| `EmptyState` | - | Ilustrasi sederhana (ikon), judul, deskripsi, dan satu aksi. |
| `Skeleton` | - | Untuk daftar dan kartu. Tidak dipakai untuk total uang saat memuat, tampilkan "—". |
| `Spinner` | sm, md | Dengan teks tersembunyi untuk pembaca layar. |
| `Money` | - | Menampilkan nilai integer rupiah dengan format id-ID. |
| `StatusBadge` | order, payment, shift | Memetakan status ke label, warna, dan ikon. |
| `SearchInput` | - | Debounce 250 ms. Tombol hapus muncul saat ada isi. |
| `Pagination` | - | Infinite scroll dilarang di halaman riwayat dan laporan. Gunakan pagination dengan jumlah halaman yang jelas. |
| `Banner` | info, warning, danger | Untuk status koneksi dan printer. Tidak bisa ditutup jika kondisinya masih aktif. |
| `KeyboardHint` | - | Menampilkan shortcut di tooltip dan di layar POS. |

### 4.1 Format Angka dan Tanggal

- **Rupiah:** `new Intl.NumberFormat('id-ID', { style: 'currency', currency: 'IDR', maximumFractionDigits: 0 })`. Nilai negatif ditampilkan dengan tanda minus dan warna danger.
- **Angka biasa:** pemisah ribuan titik, desimal koma (`id-ID`).
- **Tanggal:** `d MMM yyyy` (contoh: 8 Okt 2026). Waktu `HH.mm` mengikuti kebiasaan lokal, dengan opsi `HH:mm`. Semua dalam Asia/Jakarta.
- **Nomor pesanan:** ditampilkan apa adanya, misalnya `JKG-20261008-0001`, dengan tombol salin.

---

## 5. Layar Utama

### 5.1 Header dan Navigasi (AppShell)

- **Header:** logo dan nama toko, nama staff dan peran, indikator shift (terbuka atau tertutup, dengan jam buka), indikator koneksi, indikator printer, dan menu profil.
- **Sidebar (desktop ≥ 1024 px):** tetap terlihat, dapat diciutkan ke mode ikon. Grup menu: Operasional (POS, Pesanan, Riwayat, Shift), Katalog (Menu, Voucher), Stok (Inventory, Opname), Pembayaran (Rekening, Verifikasi), Laporan, Pengaturan (super admin), Audit (super admin).
- **Bottom navigation (mobile < 768 px):** lima item utama: POS, Pesanan, Shift, Stok, Lainnya. Menu Lainnya membuka drawer berisi sisa menu.
- **Menu yang tidak diizinkan tidak ditampilkan.** Admin tidak melihat grup Pengaturan dan Audit sama sekali.
- **Badge jumlah:** Pesanan menampilkan jumlah `new` dan `processing`. Verifikasi menampilkan jumlah `pending_verification`. Inventory menampilkan jumlah bahan menipis.

### 5.2 Login

- Field email dan password, tombol masuk, tautan lupa password.
- Pesan error umum ("Email atau password salah") agar tidak membocorkan keberadaan akun.
- Setelah 5 kegagalan, tombol dinonaktifkan 30 detik dengan hitung mundur.
- Tampilan menampilkan nama toko dan logo dari `store_settings`.

### 5.3 POS (PosPage)

Ini layar paling kritis dan dirancang untuk tablet landscape.

**Tata letak (lebar ≥ 1024 px):**

```
┌───────────────────────────────────────────────────────────┐
│ Header: logo · shift · koneksi · printer · staff          │
├───────────────────────────────┬───────────────────────────┤
│ Kategori (tab horizontal)     │  KERANJANG                │
│ Pencarian menu                │  ─────────────────        │
├───────────────────────────────┤  Item 1       2 × 18.000  │
│                               │    + Gula: Normal         │
│  Grid kartu menu              │  Item 2       1 × 22.000  │
│  (gambar, nama, harga,        │                           │
│   badge habis)                │  Subtotal      40.000     │
│                               │  Voucher       -5.000     │
│                               │  Layanan        3.500     │
│                               │  PB1            4.050     │
│                               │  Pembulatan         0     │
│                               │  ─────────────────        │
│                               │  TOTAL        42.550      │
│                               │                           │
│                               │  [ Bayar ]  [ Open Bill ] │
└───────────────────────────────┴───────────────────────────┘
```

**Panel menu:**
- Kategori sebagai tab horizontal yang bisa digeser. Kategori aktif ditandai dengan garis brand dan `aria-selected`.
- Grid kartu: 3 kolom pada tablet, 4 kolom pada layar 1280 px ke atas. Kartu menampilkan gambar (rasio 4:3, lazy load), nama, dan harga.
- Item yang `is_available = false` ditampilkan redup dengan badge "Habis" dan tidak bisa ditambahkan.
- Ketuk kartu menambah satu porsi langsung. Jika item memiliki modifier wajib atau opsional, drawer modifier terbuka terlebih dahulu.
- Tahan kartu (long press 500 ms) atau tombol kecil di kartu membuka drawer untuk catatan dan jumlah.
- Pencarian menu memfilter seluruh kategori. Hasil pencarian menggantikan grid sementara.

**Panel keranjang:**
- Setiap baris: nama, modifier, catatan, jumlah, dan total baris. Tombol `−` dan `+` berukuran 44 px.
- Geser baris ke kiri untuk hapus (dengan tombol konfirmasi cepat "Hapus" yang bisa diketuk). Pada desktop, tombol hapus selalu terlihat.
- Input voucher: field kode dengan tombol "Terapkan". Kode diubah ke uppercase saat diketik. Pesan hasil ditampilkan di bawah field dengan ikon dan warna. Voucher yang berhasil ditampilkan sebagai chip yang bisa dihapus.
- Ringkasan total memperbarui secara lokal menggunakan fungsi `calculateTotals` yang sama dengan logika server. Angka final tetap ditentukan server saat pesanan dibuat. Jika ada selisih, pembayaran tidak bisa diproses dan kasir diminta memuat ulang keranjang.
- Tombol "Kosongkan" dengan konfirmasi.

**Alur pembayaran (drawer dari bawah atau modal):**

1. Pilih metode: Tunai, Transfer, E-wallet, atau Split.
2. **Tunai:** input "Uang diterima" dengan tombol nominal cepat (50.000, 100.000, 150.000, uang pas). Kembalian tampil besar. Tombol "Selesai & Cetak" aktif hanya jika uang diterima ≥ total.
3. **Transfer atau E-wallet:** daftar rekening aktif dalam bentuk kartu. Setelah dipilih, tampil nomor rekening dengan tombol salin. Kasir mengisi nomor referensi dan opsional mengunggah screenshot. Ukuran dan format divalidasi di browser sebelum unggah. Tombol "Kirim untuk verifikasi" membuat pembayaran `pending_verification`.
4. **Split:** daftar baris pembayaran, sisa tagihan ditampilkan terus-menerus. Tombol tambah baris. Total baris harus sama dengan total sebelum tombol selesai aktif.
5. Setelah pesanan dibuat, layar sukses menampilkan nomor pesanan besar dan tombol "Pesanan Baru" (fokus otomatis), "Cetak Ulang", dan "Lihat Pesanan".

**Mode layar kecil (< 1024 px):** keranjang berubah menjadi bottom sheet dengan ringkasan (jumlah item dan total). Ketuk membuka keranjang penuh.

**Keyboard shortcut (desktop dan tablet dengan keyboard):**
- `/` fokus pencarian menu.
- Angka 1–9 memilih item pertama hingga kesembilan pada grid yang terlihat.
- `+` dan `-` pada baris keranjang yang dipilih.
- `F2` membuka pembayaran.
- `F3` membuka open bill.
- `Esc` menutup drawer.
- `F4` fokus input voucher.
- Shortcut ditampilkan di tooltip dan dapat dimatikan di pengaturan.

**Waktu respons:** aksi di keranjang (tambah, kurang, hapus) langsung terlihat tanpa menunggu server. Pembuatan pesanan menampilkan indikator loading di tombol dan memblokir klik ganda.

### 5.4 Open Bill (OpenBillPage)

- Header menampilkan nomor pesanan, meja atau nama pelanggan, waktu buka, dan total sementara.
- Panel kiri: menu yang sama seperti POS, dengan tombol "Tambah ke bill".
- Panel kanan: daftar item yang sudah masuk bill, dikelompokkan berdasarkan waktu penambahan. Item yang sudah dikirim ke dapur ditandai "Dikirim".
- Tombol "Tutup Bill" memunculkan drawer pembayaran dengan total final dan input voucher. Aksi ini tidak bisa dibatalkan kecuali dengan void per item, yang membutuhkan alasan.
- Bill yang sudah ditutup berubah menjadi tampilan baca saja dan tidak lagi menampilkan tombol tambah.
- Daftar open bill yang aktif tersedia di halaman Pesanan dengan filter "Open bill".

### 5.5 Pesanan (OrdersPage)

- **Tab status:** Semua, Baru, Diproses, Menunggu Diambil, Dibatalkan. Setiap tab menampilkan jumlah. Tab "Selesai" tersedia melalui filter agar tidak memenuhi bar.
- **Filter tambahan:** tipe (dine-in, takeaway), open bill, rentang waktu, kasir.
- **Tampilan:** kolom board (kanban) di desktop untuk status aktif (Baru, Diproses, Menunggu Diambil). Pada mobile, tab biasa dengan daftar kartu.
- **Kartu pesanan:** nomor pesanan, meja atau nama, jumlah item, waktu sejak dibuat (diperbarui tiap menit), dan badge status. Pesanan yang lebih dari 15 menit di status "Baru" diberi penanda waktu tunggu berwarna warning.
- **Aksi cepat pada kartu:** tombol "Proses", "Siap", "Selesai" sesuai transisi yang diizinkan. Tombol tidak tampil jika transisi tidak valid.
- **Realtime:** kartu baru muncul dengan animasi singkat dan suara notifikasi opsional. Notifikasi suara dimatikan secara default dan bisa diaktifkan di pengaturan perangkat.
- **Detail pesanan:** panel samping (desktop) atau halaman penuh (mobile) berisi item, pembayaran, status transaksi, riwayat status, dan aksi: Cetak Ulang, Void Item, Batalkan Pesanan. Aksi destruktif memakai ConfirmDialog dengan textarea alasan.
- **Pembayaran pending pada detail:** kartu bukti pembayaran dengan tombol "Verifikasi" yang membuka modal peninjauan.

### 5.6 Riwayat Transaksi (TransactionHistoryPage)

- Tabel dengan kolom: nomor pesanan, waktu, tipe, item (ringkasan), metode, total, status, kasir.
- Filter: rentang tanggal dengan preset, status, metode, kasir, tipe, dan pencarian nomor pesanan.
- Ringkasan di atas tabel: jumlah transaksi, total penjualan, rata-rata nilai transaksi untuk filter yang aktif.
- Klik baris membuka detail penuh: seluruh item termasuk void (dicoret dengan alasan), perhitungan total baris per baris, diskon, layanan, pajak, pembulatan, pembayaran dengan bukti, dan audit terkait.
- Tombol ekspor CSV untuk hasil filter saat ini.
- Pagination 25 baris per halaman, dengan pilihan 50 dan 100.

### 5.7 Shift (ShiftPage)

- **Tanpa shift terbuka:** kartu besar "Buka Kasir" dengan input saldo awal tunai dan tombol primer.
- **Shift terbuka:** kartu ringkasan (waktu buka, saldo awal, jumlah pesanan, total per metode, tunai yang diharapkan secara real time). Tombol "Tutup Kasir".
- **Tutup kasir:** modal dua langkah. Langkah 1 menampilkan ringkasan. Langkah 2 meminta input tunai aktual. Selisih ditampilkan dengan warna dan teks ("Selisih kurang Rp 5.000", "Pas", "Selisih lebih Rp 2.000"). Ada peringatan jika open bill masih terbuka dan tombol tutup dinonaktifkan dengan penjelasan.
- **Riwayat shift:** daftar shift sebelumnya dengan selisih dan tombol cetak ringkasan.

### 5.8 Menu (MenuPage)

- Daftar kategori dan item dalam tampilan dua kolom: kategori di kiri, item di kanan.
- Urutan kategori dan item dapat diubah dengan seret (drag handle) dan juga dengan tombol panah untuk aksesibilitas keyboard.
- Form item: nama, deskripsi, kategori, harga, gambar (unggah, kompres di browser ke maks 800 px), status aktif, status tersedia, modifier, dan resep bahan.
- Resep: tabel bahan dengan jumlah per porsi dan satuan. Validasi jumlah > 0.
- Toggle "Habis" cepat langsung dari daftar.
- Hapus item menonaktifkan item (soft delete) dengan pesan bahwa riwayat tetap tersimpan.

### 5.9 Inventory (InventoryPage)

- Tabel bahan: nama, satuan, stok saat ini, stok minimum, nilai stok (stok × unit_cost), status (aman, menipis, habis).
- Baris menipis diberi ikon peringatan dan teks "Menipis".
- Aksi: tambah pembelian (`purchase`), catat waste, dan lihat riwayat pergerakan per bahan.
- Riwayat pergerakan: daftar ledger dengan jenis, jumlah, referensi (klik menuju pesanan atau opname), waktu, dan pengguna.
- Super admin dapat mengaktifkan stok negatif dengan konfirmasi tegas. Admin melihat tombol ini tidak tersedia dan mendapat pesan bahwa stok tidak cukup.

### 5.10 Stok Opname (StockOpnamePage dan Detail)

- Daftar opname dengan status draft dan finalized.
- Halaman detail: tabel dengan kolom bahan, stok sistem (terkunci, hanya baca), stok hitung (input), dan selisih (otomatis).
- Input stok hitung bisa menggunakan keyboard numerik. Tombol Enter berpindah ke baris berikutnya.
- Penyimpanan otomatis setiap perubahan dengan indikator "Tersimpan" dan "Menyimpan…".
- Filter "Hanya yang selisih" dan "Belum dihitung".
- Tombol "Finalisasi" membuka ringkasan semua selisih. Konfirmasi wajib mengetik ulang kata "FINALISASI" untuk mencegah kesalahan tidak sengaja.
- Setelah finalisasi, seluruh halaman menjadi baca saja dengan banner informasi.

### 5.11 Voucher (VouchersPage)

- Daftar voucher: kode, nama, tipe, nilai, periode, kuota terpakai, status, dan aksi.
- Form buat/ubah voucher dengan pratinjau: contoh belanja Rp 100.000 menunjukkan potongan yang dihitung dan total akhir.
- Kode otomatis dibuat dari nama dengan tombol "Buat acak". Kode diubah ke uppercase dan hanya boleh huruf, angka, dan tanda hubung. Validasi keunikan dilakukan secara asinkron dengan debounce.
- Date range picker untuk periode. Validasi: berakhir setelah mulai.
- Voucher yang sudah pernah dipakai tidak bisa dihapus. Tombol berubah menjadi "Nonaktifkan" dengan penjelasan.
- Laporan pemakaian per voucher: jumlah pemakaian dan total potongan.

### 5.12 Rekening Pembayaran (PaymentAccountsPage)

- Daftar rekening dan e-wallet dengan kartu: penyedia, nama pemilik, nomor (dimasking sebagian di daftar, lengkap di detail), status aktif, dan urutan.
- Seret untuk mengurutkan dengan panel tombol panah sebagai alternatif.
- Form dengan validasi nomor rekening (hanya angka).

### 5.13 Verifikasi Pembayaran (PaymentVerificationPage)

- Antrian dengan daftar pembayaran `pending_verification`, diurutkan dari yang paling lama.
- Layar dibagi dua: kiri screenshot bukti (dapat diperbesar, diputar, dan dibuka di tab baru), kanan detail pembayaran (nomor pesanan, nominal, rekening tujuan, nomor referensi, waktu).
- Tombol "Setujui" dan "Tolak". Tolak memerlukan alasan.
- Pintasan keyboard: `A` setujui, `R` tolak, panah atas dan bawah berpindah antrian.
- Setelah tindakan, item otomatis berpindah ke antrian berikutnya dengan toast konfirmasi.
- Bukti pembayaran dimuat dengan signed URL dan indikator loading. Jika bukti tidak tersedia, tampil pesan "Bukti belum dilampirkan" dan verifikasi tetap bisa dilakukan dengan catatan.

### 5.14 Laporan (ReportsPage)

- Pilih rentang tanggal dengan preset.
- Tab: Ringkasan, Per Hari, Per Jam, Per Kategori, Per Item, Per Metode, Voucher.
- Ringkasan berupa kartu: total penjualan, jumlah transaksi, rata-rata nilai, total diskon, total pajak dan layanan, total void.
- Grafik batang dan garis dengan SVG sederhana. Setiap grafik memiliki tabel data di bawahnya sebagai alternatif aksesibel dan tombol unduh CSV.
- Tooltip grafik muncul saat fokus dan saat hover, serta bisa dibaca pembaca layar lewat tabel.

### 5.15 Pengaturan (SettingsPage, BrandingPage, StaffPage, PrinterSettingsPage)

- **Pengaturan umum:** identitas toko, jam buka, pajak, layanan, pembulatan, opsi wajib-verifikasi sebelum selesai.
- **Branding:** unggah logo dengan pratinjau di header, di struk, dan di kartu login. Pemilih warna dengan input hex dan pratinjau. Setiap warna menampilkan rasio kontras dan status lulus atau gagal. Pilihan font dengan pratinjau teks. Perubahan dapat dilihat di pratinjau sebelum disimpan, dengan tombol "Batal" untuk kembali.
- **Staff:** tabel akun dengan nama, peran, status aktif. Tambah staff memakai modal. Ubah peran dan nonaktifkan memakai ConfirmDialog. Super admin tidak bisa menonaktifkan akunnya sendiri.
- **Printer:** status koneksi, tombol "Hubungkan Printer", tombol "Cetak Uji", pilihan ukuran kertas, dan jumlah salinan default. Jika browser tidak mendukung Web Bluetooth, halaman menjelaskan batasan dan menawarkan cetak via browser.

### 5.16 Audit (AuditLogPage)

- Tabel dengan filter tindakan, aktor, rentang tanggal, dan entitas.
- Setiap baris dapat dibuka untuk melihat payload dalam format yang mudah dibaca (bukan JSON mentah).
- Hanya dibaca. Tidak ada aksi edit atau hapus.

---

## 6. Interaksi dan Umpan Balik

### 6.1 Pola Umpan Balik

| Kejadian | Umpan balik |
|---|---|
| Aksi berhasil | Toast success singkat, atau perubahan langsung pada UI |
| Aksi gagal | Pesan error spesifik di dekat aksi dan toast error |
| Menunggu server | Indikator loading pada tombol yang diklik, tombol nonaktif |
| Data sedang dimuat | Skeleton pada daftar, "—" pada angka |
| Koneksi terputus | Banner di atas konten, tombol server dinonaktifkan |
| Printer terputus | Badge kuning di header dan toast dengan aksi "Coba lagi" |

### 6.2 Konfirmasi Aksi

| Aksi | Konfirmasi | Alasan wajib |
|---|---|---|
| Hapus item keranjang | Tidak (bisa dibatalkan selama 5 detik lewat toast) | Tidak |
| Kosongkan keranjang | Ya | Tidak |
| Void item | Ya | Ya |
| Batalkan pesanan | Ya | Ya |
| Tutup kasir | Ya (dua langkah) | Tidak |
| Finalisasi opname | Ya, ketik ulang "FINALISASI" | Tidak |
| Tolak pembayaran | Ya | Ya |
| Retur pesanan selesai | Ya, ketik ulang nomor pesanan | Ya |
| Nonaktifkan staff atau voucher | Ya | Tidak |

### 6.3 Pencegahan Kesalahan

- Tombol submit dinonaktifkan selama request berjalan. Klik ganda diabaikan.
- Form dengan perubahan belum disimpan memicu peringatan saat meninggalkan halaman.
- Nominal uang tunai yang jauh di atas total (lebih dari 10 kali lipat) memunculkan konfirmasi "Pastikan nominal benar".
- Harga yang berubah di keranjang ditandai dengan badge dan memerlukan persetujuan kasir sebelum pembayaran.

### 6.4 Animasi dan Transisi

- Durasi: 150 ms untuk mikro-interaksi, 220 ms untuk modal dan drawer.
- Easing: `ease-out` untuk masuk, `ease-in` untuk keluar.
- Hormati `prefers-reduced-motion`: animasi dinonaktifkan dan transisi berubah menjadi perubahan instan.
- Tidak ada animasi yang berjalan lebih dari 300 ms pada alur transaksi.

---

## 7. Aksesibilitas

- **Standar:** WCAG 2.1 AA.
- **Navigasi keyboard:** semua fungsi dapat diakses tanpa mouse. Urutan fokus mengikuti urutan visual. Indikator fokus terlihat (outline 2 px dengan offset 2 px, warna brand).
- **Skip link:** tautan "Lewati ke konten utama" di awal setiap halaman.
- **Landmark:** `header`, `nav`, `main`, `aside` digunakan dengan tepat. Setiap halaman memiliki satu `h1`.
- **Form:** setiap field memiliki label terkait. Error diumumkan dan difokuskan ke field pertama yang salah saat submit.
- **Tabel:** menggunakan `th scope`, dan caption untuk tabel data.
- **Target sentuh:** minimal 44×44 px, jarak antar target minimal 8 px.
- **Warna:** informasi tidak hanya bergantung pada warna. Status selalu disertai teks dan ikon.
- **Zoom:** tata letak tetap berfungsi pada zoom 200%. Teks dapat diperbesar hingga 200% tanpa kehilangan fungsi.
- **Pembaca layar:** toast dan perubahan status realtime diumumkan melalui live region. Pesanan baru di antrian diumumkan sekali, bukan setiap pembaruan.
- **Pengujian:** axe-core di Playwright untuk setiap halaman. Pengujian manual dengan TalkBack dan VoiceOver pada alur POS dan pesanan.

---

## 8. Responsivitas

| Breakpoint | Lebar | Perilaku |
|---|---|---|
| Mobile | < 768 px | Bottom navigation, tabel jadi kartu, keranjang jadi bottom sheet, drawer penuh layar |
| Tablet | 768–1023 px | Bottom navigation dengan label, POS dua kolom dengan keranjang lebih sempit, sidebar ciut |
| Desktop | ≥ 1024 px | Sidebar penuh, POS tiga area (menu, keranjang, aksi) |
| Layar besar | ≥ 1440 px | Grid menu lebih lebar, panel samping detail tetap terbuka |

**Orientasi:** POS dirancang untuk tablet landscape. Dalam orientasi potret, POS menampilkan peringatan kecil yang menyarankan landscape, tetapi tetap berfungsi.

**Pendekatan:** mobile-first CSS. Utility Tailwind dengan breakpoint `md` dan `lg`.

---

## 9. Performa Frontend

| Metrik | Target |
|---|---|
| First Contentful Paint halaman POS | Di bawah 1,2 detik pada 4G menengah |
| Largest Contentful Paint POS | Di bawah 2,0 detik |
| Interaction to Next Paint | Di bawah 200 ms untuk tambah item dan ketuk tombol |
| Total JavaScript awal (gzip) | Di bawah 150 KB |
| Chunk per halaman | Di bawah 60 KB gzip |
| Cumulative Layout Shift | Di bawah 0,05 |

**Teknik:**
- Route-level code splitting dengan `lazy`.
- Gambar menu: `loading="lazy"`, `decoding="async"`, ukuran eksplisit untuk mencegah pergeseran tata letak. Format WebP dengan fallback.
- Daftar panjang (riwayat, audit) menggunakan virtualisasi baris.
- Ikon diimpor per nama (tree-shaking), tidak mengimpor seluruh paket.
- Tidak ada library grafik besar. Grafik dibuat dengan SVG.
- Font dimuat dengan `font-display: swap` dan hanya subset yang dipakai (latin dan latin-ext).
- Service worker hanya meng-cache aset statis. Respons API dan data transaksi tidak pernah di-cache.

---

## 10. Pengujian Frontend

| Jenis | Alat | Cakupan |
|---|---|---|
| Komponen | Vitest + @solidjs/testing-library | Semua komponen `shared/ui` dengan varian dan state (loading, error, disabled) |
| Fitur | Vitest | Store keranjang, kalkulasi total di sisi klien, validasi Zod, pemetaan AppError |
| Visual regression | Playwright screenshot | Halaman POS, Pesanan, Shift, dan Verifikasi dalam tema terang dan gelap, dua ukuran layar |
| E2E | Playwright | Alur kritis seperti di spesifikasi sistem, ditambah: keyboard-only POS, tema gelap, layar 768 px, koneksi putus saat pembayaran |
| Aksesibilitas | axe-core | Setiap halaman, nol pelanggaran serius atau kritis |
| Performa | Lighthouse CI | POS dan Pesanan pada profil mobile |

**Mock printer dan mock jaringan:** Playwright dapat mensimulasikan offline (`context.setOffline`) dan kegagalan RPC untuk menguji banner dan pesan error.

---

## 11. Kriteria Penerimaan Frontend

1. Pesanan tunai sederhana dapat diselesaikan dalam 6 ketukan atau kurang, diukur dari pilihan item pertama hingga tombol "Selesai & Cetak".
2. Semua tombol aksi memiliki target sentuh minimal 44×44 px, diverifikasi lewat pengujian otomatis.
3. Semua halaman lulus axe-core tanpa pelanggaran serius atau kritis pada tema terang dan gelap.
4. Admin tidak melihat menu, rute, atau aksi yang hanya untuk super admin, dan mendapat halaman 403 jika mengetik URL langsung.
5. Perubahan branding (warna, logo, font) terlihat di seluruh aplikasi setelah disimpan tanpa muat ulang halaman.
6. Validasi kontras warna menolak penyimpanan jika rasio di bawah 4,5:1 untuk teks normal.
7. Setiap aksi destruktif memiliki konfirmasi, dan aksi void serta pembatalan tidak dapat dilanjutkan tanpa alasan.
8. Keranjang bertahan saat halaman di-refresh dan dihapus setelah pesanan berhasil.
9. Pesan error dari server ditampilkan dalam Bahasa Indonesia sesuai pemetaan kode error.
10. POS dapat digunakan sepenuhnya dengan keyboard dan dengan layar 768 px.
11. Halaman POS mencapai skor performa Lighthouse minimal 85 pada profil mobile.
12. Tidak ada animasi yang mengganggu saat `prefers-reduced-motion` aktif.
