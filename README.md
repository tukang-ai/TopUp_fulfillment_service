# ⚡ TopUp Fulfillment Service (Server Topup & Bot)

[![Rust](https://img.shields.io/badge/Rust-1.75%2B-orange.svg)](https://www.rust-lang.org/)
[![License: AGPL v3](https://img.shields.io/badge/License-AGPL_v3-blue.svg)](https://www.gnu.org/licenses/agpl-3.0)
[![Engine: Tokio](https://img.shields.io/badge/Engine-Tokio_Async-blueviolet.svg)](https://tokio.rs/)

**TopUp Fulfillment Service** adalah server eksekusi pemrosesan pesanan otomatis (*Headless Fulfillment Engine*) berbasis Rust dan Tokio. Server ini bertindak sebagai **Server Topup** yang menjalankan bot Telegram, menangani notifikasi mutasi GoPay, mengeksekusi pesanan topup ke API supplier DigiFlazz, melakukan pengecekan status background poller secara berkala, dan mengembalikan Serial Number (SN) atau refund ke Server Web.

---

## 🔒 Fitur Keamanan Utama (Zero-Inbound Port)

- **Zero-Inbound Port**: Server ini sama sekali **tidak membuka port HTTP publik (0 open ports)**. Seluruh perintah diterima secara *outbound polling* melalui Telegram Bot API.
- **End-to-End Encryption**: Setiap payload yang diterima diverifikasi menggunakan tanda tangan digital **HMAC-SHA256** dan perlindungan **Anti-Replay Attack (toleransi timestamp $\le 120$ detik)**.
- **Exclusive Atomic Claim Locking**: Menjamin **100% Anti Dobel Topup** dengan mengunci status pesanan di database lokal sebelum memanggil supplier.

---

## 🤖 Panduan Mendapatkan Telegram Bot Token & Group ID

Server Topup memerlukan 2 Bot Telegram:

| Variabel `.env` | Nama Bot | Fungsi |
|---|---|---|
| `TELEGRAM_BOT_1_TOKEN` & `TELEGRAM_GROUP_1_ID` | **GoPay Bot** | Menangkap pesan SMS mutasi uang masuk dari aplikasi *SMS Forwarder* di HP toko. |
| `TELEGRAM_BOT_2_TOKEN` & `TELEGRAM_GROUP_2_ID` | **Report Bot** | Jalur bus terenkripsi antar-server (order baru, OTP saldo, Serial Number). |

### 1. Cara Membuat Bot & Mendapatkan `Bot Token`:
1. Buka aplikasi Telegram, cari bot resmi **`@BotFather`**.
2. Kirim perintah `/newbot`.
3. Masukkan nama bot (contoh: `Aruteru Topup Engine`) dan username bot (contoh: `aruteru_topup_bot`).
4. `@BotFather` akan memberikan **Bot Token** (contoh: `7123456789:AAF_AbCdEfGhIjKlMnOpQrStUvWxYz12345`).

### 2. Cara Membuat Grup & Mendapatkan `Group Chat ID`:
1. Buat **Grup Baru** di Telegram dan undang bot Anda ke dalam grup tersebut.
2. Jadikan bot sebagai **Admin Grup**.
3. Masukkan bot pembantu **`@raw_data_bot`** ke grup, lalu catat ID grup yang muncul pada field `"chat": { "id": -100xxxxxxxxxx }`.
4. Masukkan ID tersebut (lengkap dengan tanda minus `-100`) ke file `.env`.

---

## 🏛️ Arsitektur Alur Pemenuhan Pesanan

```
                                  ┌───────────────────────────┐
                                  │   SERVER WEB (STORE)      │
                                  └─────────────┬─────────────┘
                                                │ Telegram Encrypted Bus
                                                ▼
┌────────────────────────────────────────────────────────────────────────────────────────┐
│                        TOPUP FULFILLMENT SERVICE (SERVER TOPUP)                        │
│                                                                                        │
│  1. Report Bot Listener:                                                               │
│     - Menerima [NEW_ORDER], [REQ_OTP], [VERIFY], [REPORT].                             │
│     - Memverifikasi Signature HMAC-SHA256 & Anti-Replay Guard.                         │
│  2. GoPay Bot Listener:                                                                │
│     - Membaca mutasi SMS GoPay via awalan kata "rp" (anti false-positive).             │
│     - Pencocokan FIFO (ORDER BY id ASC) untuk nominal yang sesuai.                     │
│  3. Atomic Claim Guard (Anti Double Topup):                                            │
│     - Mengunci status pesanan (unpaid -> processing) secara atomik di DB Topup lokal.  │
│  4. Supplier Execution Engine (DigiFlazz API):                                         │
│     - Sukses Instan -> Kirim [STATUS_UPDATE] order_id success <SN> ke Server Web.      │
│     - Pending -> Diserahkan ke Status Poller (30s).                                    │
│     - Gagal -> Kirim [REFUND_USER] ke Server Web (khusus pembayaran Saldo).            │
│  5. Background Cron Workers:                                                           │
│     - Status Poller (30 detik): Mengecek pesanan pending via commands "status".        │
│     - Tokopay Reconciler (60 detik): Menangkap deposit & order Tokopay unpaid.         │
│     - Expired Cleaner (60 detik): Meng-cancel order kadaluwarsa > 15 menit.             │
└────────────────────────────────────────────────────────────────────────────────────────┘
```

---

## ⚙️ Prasyarat Sistem

- **Rust Toolchain**: `rustc` & `cargo` versi 1.75 atau lebih baru.
- **Database**: MySQL 8.0+ atau MariaDB 10.5+ (Lokal untuk Server Topup).
- **Akun Telegram Bot**: Bot Token dari `@BotFather` & Group Chat ID Telegram.

---

## 🚀 Panduan Konfigurasi & Menjalankan Server

### 1. Konfigurasi Environment (`.env`)
Salin file `.env.example` menjadi `.env`:
```bash
cp .env.example .env
```

Sesuaikan parameter `.env`:
```env
# Database Topup Lokal (MySQL)
DATABASE_URL=mysql://db_user:db_password@localhost:3306/db_fulfillment_topup

# Telegram Bots Configuration
TELEGRAM_BOT_1_TOKEN=7123456789:AAF_TokenBot1_Gopay
TELEGRAM_GROUP_1_ID=-1001111111111

TELEGRAM_BOT_2_TOKEN=7123456789:AAF_TokenBot2_Report
TELEGRAM_GROUP_2_ID=-1002222222222

TELEGRAM_ENCRYPTION_KEY=ARUTERU_SECRET_KEY_SUPER_SECURE_2026

# Kredensial Supplier DigiFlazz
DIGIFLAZZ_USERNAME=your_digi_username
DIGIFLAZZ_APIKEY=your_digi_production_apikey

# WhatsApp Gateway untuk OTP Transaksi (OpenWA / MPWA)
WA_GATEWAY_URL=http://127.0.0.1:3000/api/v1/send-message
MPWA_API_KEY=your_wa_token
MPWA_SENDER_PHONE=default
```

### 2. Menjalankan Server
```bash
# Mode Development
cargo run --bin fulfillment_service

# Mode Produksi (Optimasi Penuh)
cargo run --bin fulfillment_service --release
```

Server Topup akan langsung aktif, memantau antrean bot Telegram, dan menjalankan worker cron di latar belakang.

---

## 📄 Lisensi
Proyek ini dilindungi di bawah lisensi **GNU AGPLv3 (Affero General Public License v3.0)**.
