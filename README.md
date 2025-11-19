# Dokumentasi Arsitektur Backend Modular Monolith

Dokumen ini menjelaskan struktur teknis dan alur logika untuk backend aplikasi berbasis **NestJS**. Aplikasi ini dirancang menggunakan arsitektur **Modular Monolith** untuk mengelola ekosistem perumahan cerdas, mencakup manajemen properti, pasar jasa/produk, dan integrasi AI.

---

## Struktur Modul

Berikut adalah rincian pembagian modul dalam sistem:

### 1. AuthModule
Modul ini menangani autentikasi dan otorisasi pengguna.
* **Fitur:** Registrasi, Login (JWT), Manajemen Peran.
* **Role:** `HOMEOWNER` (Pemilik Rumah), `CONTRACTOR` (Kontraktor), `VENDOR` (Penyedia Barang), `ADMIN`.

### 2. PropertyModule ("Paspor Rumah Digital")
Modul inti yang menyimpan identitas digital setiap unit rumah.
* **Manajemen Properti:** Menyimpan spesifikasi rumah dalam format `JSONB` (kode cat, tipe lantai, dll).
* **Dokumen:** "Loker dokumen" untuk cetak biru (*blueprints*) dan kartu garansi digital.
* **Jadwal Perawatan:** Mengelola `maintenance_schedule` (misal: pengingat servis AC rutin).

### 3. MarketplaceModule (Parent Module)
Modul induk yang menaungi dua sub-domain bisnis utama:

#### a. Services (Sub-Modul Jasa)
Mengelola pasar layanan perbaikan dan renovasi.
* `service_requests`: "Pekerjaan" yang diposting oleh pemilik rumah (Client).
* `service_quotes`: "Penawaran" yang diajukan oleh kontraktor.
* `contractors`: Profil lengkap Kontraktor.

#### b. ECommerce (Sub-Modul Material)
Mengelola pasar jual-beli material bangunan.
* `products`: Data produk (dari Vendor).
* `orders` & `order_items`: Manajemen pesanan dan detail item.
* `vendors`: Profil Vendor (Toko Material).

### 4. PaymentModule
* **Integrasi:** Menghubungkan sistem dengan *payment gateway* (contoh: Midtrans, Xendit).
* **Escrow:** Mengelola alur Rekening Bersama (*Escrow*) secara khusus untuk modul Services guna mengamankan transaksi antara pemilik rumah dan kontraktor.

### 5. CommunityModule
* **Pengumuman:** Distribusi informasi dari Admin ke penghuni.
* **Fasilitas:** Sistem pemesanan fasilitas umum (*booking system*) seperti *clubhouse* atau kolam renang.

### 6. ChatModule (WebSocket)
Sistem komunikasi *real-time*.
* **Chat Threads:** Mengelola ruang obrolan yang terikat spesifik pada `service_requests` atau `orders`.
* **Chat Messages:** Penyimpanan riwayat pesan.

### 7. AiModule
Fitur kecerdasan buatan untuk meningkatkan pengalaman pengguna.
* **SmartRequestService:** Menggunakan NLP (*Natural Language Processing*) untuk menerjemahkan teks keluhan menjadi kategori layanan.
* **VisionController:** *Endpoint* API untuk memproses unggahan foto ke layanan AI eksternal (Computer Vision).

---

##  Alur Pengguna Utama (Core User Flows)

Bagian ini menjelaskan implementasi logika bisnis utama sistem.

### Flow 1: "Serah Terima Digital" (Homeowner Onboarding)
Proses ini mengubah serah terima fisik menjadi pengalaman digital yang mulus.

1.  **Pra-Kondisi:** Admin membuat entri properti di panel web, mengunggah spesifikasi dan dokumen. Sistem menghasilkan `purchase_code` unik (QR Code).
2.  **Hari Serah Terima:** Pemilik rumah (Sarah) menerima kunci fisik dan kode QR.
3.  **Onboarding:** Sarah mengunduh aplikasi, mendaftar akun, dan memindai kode QR.
4.  **The Magic Moment:**
    * Aplikasi memanggil `POST /auth/verify-homeowner`.
    * Backend memvalidasi `purchase_code`.
    * Backend menautkan `user.id` Sarah dengan `property.id`.
    * Status diubah menjadi `ownership_verified = true`.
5.  **Hasil:** Aplikasi Sarah langsung terisi data. Menu "Properti Saya" menampilkan cetak biru, garansi, dan jadwal perawatan rumahnya secara otomatis.

### Flow 2: "Permintaan Pintar" & Bidding
Solusi untuk masalah perbaikan rumah menggunakan bantuan AI.

1.  **Masalah:** Keran dapur Sarah bocor.
2.  **Smart Request:** Sarah mengetik keluhan: *"Keran dapur saya menetes dan tidak mau berhenti."*
3.  **Analisis AI (AiModule):**
    * Teks dikirim ke `POST /ai/smart-request`.
    * Service menggunakan `pgvector` untuk mencari kecocokan semantik terdekat.
    * Output: `{ "category": "plumbing" }`.
4.  **Posting Pekerjaan:** Aplikasi membuka form dengan kategori "Plumbing" terpilih otomatis. Sarah melengkapi foto dan submit.
5.  **Backend:** Membuat `service_request` baru dengan status `OPEN_FOR_BIDS`.
6.  **Notifikasi:** Dikirim ke semua kontraktor kategori *Plumbing* yang terverifikasi.
7.  **Bidding:** Kontraktor (Budi) melihat job dan mengajukan penawaran (`service_quote`).
8.  **Penerimaan:** Sarah memilih Budi. Status request berubah menjadi `AWAITING_PAYMENT`, ID kontraktor disimpan, dan alur pembayaran dipicu.

### Flow 3: "Escrow & Penyelesaian"
Mekanisme keamanan transaksi jasa.

1.  **Pembayaran:** Sarah membayar dana *escrow* via aplikasi.
2.  **Notifikasi Mulai:** Budi menerima info: *"Pembayaran aman di escrow. Silakan mulai kerja."*
3.  **Pengerjaan:** Budi memperbaiki keran, lalu mengunggah foto "Before & After". Ia mengubah status menjadi `AWAITING_APPROVAL`.
4.  **Approval:** Sarah menerima notifikasi, mengecek hasil kerja, dan menekan tombol **"Approve Job"**.
5.  **Pencairan (Payout):** `PaymentModule` mendeteksi *event* approval dan otomatis mentransfer dana ke rekening Budi.
6.  **Review:** Sarah memberikan rating bintang 1-5 untuk kinerja Budi.
