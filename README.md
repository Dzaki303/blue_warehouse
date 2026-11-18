# WMS Warehouse

Aplikasi web Warehouse Management System berdasarkan flowchart dan sequence diagram Kapita.

Stack: Python (Flask, OOP), REST API (JSON), MongoDB Atlas, session server-side, frontend Webix, deploy ke Render.

## Fitur

| Modul | Alur |
|---|---|
| Login dan Session | Login username/password, lalu masuk dashboard sesuai role. Gagal: pesan error. Session disimpan di MongoDB (koleksi `sessions`, kedaluwarsa otomatis), logout mencabut session. |
| Master Data | Barang, Kategori, dan Warehouse dikelola Admin. Semua perubahan tercatat di Audit Log. |
| Data Supplier | Supplier adalah pihak luar yang sudah punya daftar produk dan harga. Admin menginput data supplier di menu Data Supplier; produk dan harganya dikelola supplier sendiri lewat Portal Supplier (atau Admin lewat Import JSON). Tombol **Lihat Produk dan Harga** hanya menampilkan daftar produk dan harga. Manager hanya membacanya untuk PO. |
| Request Barang | Staff membuat request (status Menunggu), Manager menyetujui/menolak, Staff input barang keluar, stok dipotong atomic (transaksi MongoDB), status Selesai. Stok kurang: status Backorder. |
| Purchase Order | Manager membuat PO dan langsung berstatus Ordered (tanpa approval). Alurnya: pilih supplier dulu, lalu pilih barang yang sudah terdaftar dari supplier itu; harga dari supplier, subtotal dan total berubah sesuai qty. Setelah ada barang, supplier terkunci. Staff cek barang: Sesuai menambah stok; Rusak / Kelebihan / Salah Barang / Lainnya dicatat sebagai klaim ke supplier. Status: Ordered, Partially Received, Received, Selesai (Kurang), Dibatalkan. |
| Klaim Supplier | Klaim punya status tindak lanjut (Terbuka, Diganti, Refund, Ditutup) yang diperbarui Manager, dan tampil di laporan. |
| Stock Opname | Manager buat sesi, Staff input hasil hitung, selisih di atas ambang (default 10%) memicu hitung ulang, lalu adjustment diajukan ke Manager, disetujui, stok disesuaikan (transaksi ADJ). |
| Kartu Stok | Semua transaksi IN / OUT / ADJ beserta saldo. |
| Monitoring dan Laporan | Admin/Manager: preview dan unduh Excel dan PDF (stok, stok menipis, kartu stok, request, PO, klaim supplier), serta Audit Log. |
| Pengguna | Admin kelola akun dan role. |

Aturan bisnis: approver tidak boleh sama dengan pembuat request maupun penghitung opname; stok tidak pernah negatif. PO tidak memakai approval; PO yang salah dapat dibatalkan (selama belum ada barang diterima), dan PO yang kurang kiriman dapat ditutup oleh Manager dengan alasan.

## Menu dan hak akses per role

| Menu | Admin | Manager | Staff |
|---|:-:|:-:|:-:|
| Dashboard | Ya | - | - |
| Barang dan Stok (+ Kartu Stok) | Kelola | - | Lihat |
| Kategori, Warehouse | Kelola | - | - |
| Data Supplier (data supplier) | Kelola | - | - |
| Lihat Produk dan Harga (baca daftar produk & harga supplier) | Lihat | Lihat | - |
| Tambah Barang dari Supplier (daftarkan produk supplier) | Ya | - | - |
| Request Barang | - | Setujui / tolak | Buat, input barang keluar |
| Purchase Order dan Barang Masuk | - | Buat PO, batalkan, tutup, kelola klaim | Cek dan terima barang |
| Stock Opname | - | Buat sesi, setujui adjustment | Input hasil hitung |
| Monitoring dan Laporan (+ Audit Log) | Ya | Ya | - |
| Pengguna | Kelola | - | - |

Manager membaca daftar supplier hanya untuk membuat PO.

## 1) Siapkan MongoDB Atlas

1. Daftar/login di https://cloud.mongodb.com lalu buat Cluster (Free M0 sudah cukup, mendukung transaksi).
2. Database Access: Add New Database User (catat username dan password).
3. Network Access: Add IP Address. Untuk lokal tambahkan IP Anda; untuk Render pilih "Allow access from anywhere" (0.0.0.0/0).
4. Database, Connect, Drivers: salin connection string, contoh:
   `mongodb+srv://USER:PASSWORD@cluster0.xxxxx.mongodb.net/?retryWrites=true&w=majority`
   (jika password mengandung karakter khusus seperti `@` `#` `/`, ubah ke bentuk URL-encoded).

Database dan koleksi dibuat otomatis oleh aplikasi saat pertama kali jalan.

## 2) Menjalankan di komputer lokal

Butuh Python 3.10+.

```bash
cd wms-warehouse
python -m venv venv
# Windows:   venv\Scripts\activate
# Linux/Mac: source venv/bin/activate
pip install -r requirements.txt

cp .env.example .env        # Windows: copy .env.example .env
# edit .env: isi MONGO_URI (dari Atlas) dan SECRET_KEY (string acak panjang)

python app.py
```

Buka http://localhost:5000

### Akun bawaan (dibuat otomatis saat database masih kosong)

| Role | Username | Password |
|---|---|---|
| Admin | `admin` | `admin123` |
| Manager | `manager` | `manager123` |
| Staff Warehouse | `staff` | `staff123` |
| Supplier (hanya untuk portal `/portal`) | `supplier` | `supplier123` |

Segera ganti password lewat menu Pengguna (login sebagai admin). Setelah itu set `SEED_DEFAULTS=false` di `.env` bila tidak ingin data contoh dibuat lagi.

### Mencoba alur lengkap (sekitar 5 menit)
1. Login staff, menu Request Barang, klik Request Baru, pilih barang, kirim.
2. Login manager, menu Request Barang, pilih baris, klik Setujui.
3. Login staff, menu Request Barang, pilih baris, klik Input Barang Keluar dan Serahkan. Cek stok berkurang di Kartu Stok.
4. Login admin, menu Data Supplier: tambah supplier baru (data contoh sudah ada). Tombol Lihat Produk dan Harga hanya menampilkan produk dan harga supplier; produk/harga diisi supplier lewat portal `/portal` atau oleh Admin lewat Import JSON. Lalu menu Barang dan Stok, klik Tambah Barang dari Supplier: pilih supplier, pilih produknya (nama, satuan, harga sudah dari supplier), isi kode `BRG-`, kategori, dan rak.
5. Login manager, menu Purchase Order, Buat PO: pilih supplier, lalu barang yang sudah terdaftar dari supplier itu; harga dari supplier. Login staff, Cek dan Terima Barang; coba Tambah Baris Klaim untuk barang rusak. Login manager, Kelola Klaim untuk mengubah status, atau Tutup PO (Kurang).
6. Login manager, menu Stock Opname, Buat Sesi. Staff input hasil hitung, manager setujui adjustment.
7. Login manager atau admin, menu Monitoring dan Laporan, unduh Excel / PDF.

### Jika database lama masih berisi data versi sebelumnya
Role `purchasing` sudah dihapus. Akun lama dengan role itu tidak akan punya menu; hapus lewat menu Pengguna. PO lama berstatus `Menunggu Approval` tidak bisa diproses lagi; ubah statusnya menjadi `Ordered` langsung di database, atau hapus datanya. Katalog supplier versi sebelumnya (`catalog`) diubah otomatis menjadi daftar produk supplier (`products`) saat aplikasi dijalankan, dan barang yang ada di katalog dihubungkan ke supplier-nya. Cara paling bersih: pakai nama database baru lewat `MONGO_DB` di `.env`.

## 3) Deploy ke Render (gratis)

Render tidak membutuhkan Procfile maupun render.yaml. Semua pengaturan cukup diisi lewat dashboard.

1. Upload folder proyek ke repository GitHub (file `.env` jangan ikut; sudah ada di `.gitignore`).
2. Di https://render.com pilih New, Web Service, lalu hubungkan repository tersebut.
3. Isi pengaturan berikut:
   - Runtime: Python 3
   - Build Command: `pip install -r requirements.txt`
   - Start Command: `gunicorn app:app --workers 2 --threads 4 --timeout 120`
4. Tambahkan Environment Variables: `MONGO_URI` (wajib), `SECRET_KEY` (string acak), `COOKIE_SECURE=true`, dan bila perlu `PYTHON_VERSION=3.12.7`.
5. Pastikan di Atlas Network Access mengizinkan `0.0.0.0/0`.
6. Klik Create Web Service. Aplikasi live di `https://nama-app.onrender.com`.

Cold start: pada paket gratis Render, request pertama setelah aplikasi tertidur 15 menit bisa butuh sekitar 30 detik. Ini wajar.

## Struktur proyek

```
wms-warehouse/
  app.py              entry point (python app.py / gunicorn app:app)
  config.py           konfigurasi dari .env
  requirements.txt
  .env.example
  templates/
    index.html        kerangka HTML (memuat Webix, style.css, dan modul static/js/*)
  static/
    style.css         gaya tampilan
    js/               frontend Webix, dimuat berurutan oleh index.html (urutan sesuai alur)
      core/           state.js, helpers.js, api.js, ui.js (fondasi yang dipakai semua modul)
      auth.js         login / logout
      layout.js       kerangka aplikasi dan menu per role
      components/     master-crud.js (generator CRUD), baris-editor.js (editor baris Request/PO)
      pages/          dashboard, kategori, warehouse, supplier, barang, pengguna,
                      request, purchase-order, opname, kartu-stok, laporan, audit
      main.js         entry point: cek sesi lalu login / mulai aplikasi
  wms/
    __init__.py       application factory, error handler, session hook
    database.py       class Database (koneksi Atlas, index, transaksi, penomoran)
    models.py         BaseModel + Item, Category, Warehouse, Supplier, User, AuditLog
    auth.py           SessionManager, require(role)
    services.py       StockService, RequestService, POService, OpnameService, DashboardService
    reports.py        ReportBuilder, ExcelExporter, PdfExporter
    routes.py         REST API (MethodView + endpoint aksi)
    seed.py           akun dan data contoh
```

## REST API ringkas

`POST /api/auth/login|logout`, `GET /api/auth/me`, CRUD `/api/items|categories|warehouses|suppliers|users` (supplier menyimpan `products`),
`/api/requests` (+ `/<id>/approve|reject|fulfill`), `/api/purchase-orders` (+ `/<id>/cancel|close|receive`, `/<id>/claims/<no>`),
`/api/opnames` (+ `/<id>/count|approve|reject`), `GET /api/transactions`, `GET /api/dashboard`,
`GET /api/audit-logs`, `GET /api/reports/<jenis>/<json|xlsx|pdf>`

## Catatan

- Webix dimuat dari CDN resmi (`cdn.webix.com/edge`), sama seperti contoh tugas. Untuk offline atau mengunci versi, unduh `webix.js` dan `webix.css`, simpan di folder `static/`, lalu ubah dua baris link/script Webix di `templates/index.html`.
- Ambang selisih opname diatur lewat `OPNAME_THRESHOLD_PCT`; setelah satu kali hitung ulang, selisih tetap diajukan ke Manager agar tidak berputar tanpa akhir.
- Waktu pada laporan memakai WIB (UTC+7); tampilan di web mengikuti zona waktu browser.
- Cookie session bersifat HttpOnly dan SameSite=Lax; aktifkan `COOKIE_SECURE=true` saat memakai HTTPS.
- Jangan menulis MONGO_URI langsung di kode. Simpan hanya di `.env` atau environment variable Render.

## Troubleshooting

- `ServerSelectionTimeoutError` / gagal konek: IP belum diizinkan di Atlas Network Access, atau `MONGO_URI` salah (cek user/password).
- `MONGO_URI belum diisi`: file `.env` belum dibuat / belum diisi.
- Error DNS pada `mongodb+srv://`: pastikan `pip install -r requirements.txt` sukses dan koneksi internet normal.

## Format data barang
- **Kode barang** harus berformat `BRG-angka` (contoh `BRG-006`); form tambah sudah terisi awalan `BRG-`.
- **Lokasi rak** dipilih lewat dropdown blok (A sampai F) ditambah nomor rak, disimpan sebagai `A-01`, `B-12`, dan seterusnya.
- **Harga beli, nama, dan satuan** barang diambil dari produk supplier saat barang didaftarkan. Harga di PO selalu mengikuti harga supplier terbaru.
- **Menambah supplier atau produk:** lewat menu Data Supplier. Untuk memasukkan banyak supplier sekaligus dari file JSON, pakai `python tambah_supplier.py supplier_baru.json` (contoh file ada di folder proyek). Aturan: produk yang sudah terdaftar sebagai barang tidak bisa dihapus dari supplier; supplier yang punya barang atau PO tidak bisa dihapus; perubahan harga produk langsung mengikuti Harga Beli barang dan PO berikutnya.

## Import supplier, produk, dan harga dari JSON
Di menu **Data Supplier** (Admin) klik **Import JSON**, lalu tempel JSON atau pilih file `.json`. Tombol **Isi Contoh Format** menampilkan contohnya. Satu objek atau daftar supplier diterima:

```json
[
  {
    "name": "PT Contoh Supplier", "contact": "Budi", "phone": "0812", "email": "a@b.com", "address": "Jakarta",
    "products": [
      { "sku": "CS-001", "name": "Produk A", "unit": "pcs", "price": 100000 }
    ]
  }
]
```
- Supplier baru ditambahkan. Supplier yang namanya sudah ada diperbarui: produk dengan `sku` sama diperbarui, `sku` baru ditambahkan, produk lain tidak dihapus.
- `price` harus berupa angka. Seluruh data divalidasi dulu; jika ada satu yang salah, tidak ada yang disimpan.
- Harga beli barang yang sudah terdaftar ikut berubah, dan PO berikutnya memakai harga baru.
- Alternatif lewat terminal: `python tambah_supplier.py supplier_baru.json`.

## Portal Supplier (`/portal`)
Halaman terpisah untuk supplier mengelola profil dan daftar produk beserta harganya sendiri.
1. Admin membuat akun di **Data Supplier**, pilih supplier, klik **Buat Akun Portal** (username dan password).
2. Supplier membuka `http://alamat-aplikasi/portal` dan login. Akun contoh: `supplier` / `supplier123` (PT Sumber Makmur).
3. Supplier hanya dapat melihat dan mengubah data miliknya sendiri. Akun supplier tidak dapat membuka data WMS (barang, stok, PO, laporan) sama sekali.
4. Produk yang sudah didaftarkan Admin sebagai barang tidak dapat dihapus dan SKU-nya tidak dapat diubah, tetapi nama, satuan, dan harganya boleh diperbarui. Nama supplier hanya dapat diubah Admin.
# blue_warehouse
