# ⚡ Fulfillment Service (Server Topup)

[![Rust](https://img.shields.io/badge/Rust-1.85%2B-orange.svg)](https://www.rust-lang.org/)
[![Zero-Inbound](https://img.shields.io/badge/Ports-Zero--Inbound-success.svg)]()

**Fulfillment Service** adalah otak eksekusi platform topup: **sumber kebenaran harga, voucher, promo, level user, dan saldo**. Service ini TIDAK membuka port HTTP apa pun — seluruh komunikasi masuk/keluar melalui antrean Telegram terenkripsi (HMAC-SHA256 + anti-replay 120 detik) dengan database MySQL miliknya sendiri yang terpisah total dari Server Web.

> ⚠️ **Repo ini HANYA Server Topup.** Website publik ada di proyek terpisah — **TopUp Store Gateway** (`rust_backend`). Web tidak dipercaya untuk menentukan harga.

---

## 🏛️ Prinsip Utama

1. **Otoritas penuh di sini** — setiap order divalidasi ulang: harga dihitung dari DB Topup (base + margin level − flashsale − voucher), bukan nilai kiriman web.
2. **Two-phase checkout** — web baru boleh membuat tagihan gateway SETELAH server ini mengirim `[ORDER_ACCEPTED]`. Order ditolak → invoice tidak pernah jadi.
3. **Pengamanan voucher 3 level**
   - **L1**: dedup per-batch — dari N pesanan identik (user+voucher), hanya yang pertama diproses.
   - **L2**: validasi penuh — stok atomik (`stock > 0` row-lock), kategori produk, minimum belanja, eksklusif flashsale.
   - **L3**: anti-replay — voucher yang masih tertaut transaksi aktif/sukses lain otomatis kehilangan diskon.
4. **Idempoten penuh** — pesan duplikat/replay tidak akan mengeksekusi topup dua kali (atomic claim + dedup per-order).

---

## 🤖 Bot Telegram & Pembagian Kuota

| Variabel | Arah | Fungsi |
|---|---|---|
| `TELEGRAM_BOT_2_TOKEN` | masuk | Listener perintah `[NEW_ORDER]`, `[REQ_OTP]`, `[VERIFY]`, `[API_ORDER]`, `[REPORT]` |
| `TELEGRAM_BOT_1_TOKEN` | masuk | Listener mutasi GoPay |
| `TELEGRAM_BOT_3_TOKEN` | keluar | Sinkronisasi harga/promo/voucher/level ke web (Bot 3) |
| `TELEGRAM_BOT_SENDER_TOKEN` + `TELEGRAM_BOT_4_TOKEN` | keluar | Pengirim antrean batch round-robin (2×20 = 40 call/menit → siklus 5 detik aman) |

Semua bot admin di grup yang sama; tanpa anggota manusia. `TELEGRAM_ENCRYPTION_KEY` **wajib identik** dengan Server Web. Token listener ini **wajib berbeda** dengan token listener web (getUpdates 409).

## 📨 Kontrak Perintah Utama

```
[NEW_ORDER]      oid user code target provider price nama voucher   → validasi + ACC/tolak
[REQ_OTP]        oid username code target total voucher               → pembayaran saldo
[VERIFY]         oid username otp                                     → verifikasi OTP saldo
[API_ORDER]      oid user code target provider price nama -           → order API reseller (saldo)
[REPORT]         oid                                                  → konfirmasi lunas gateway
Keluar: [ORDER_ACCEPTED] [ORDER_REJECTED] [STATUS_UPDATE] [REFUND_USER]
        [SYNC_PRICE] [SYNC_FLASH] [SYNC_VOUCHER] [SYNC_USER]
```

---

## ⚙️ Prasyarat

- Rust toolchain 1.85+
- MySQL 8.0+ / MariaDB 10.5+ (**database Topup terpisah** — bukan database web)

## 🚀 Konfigurasi & Menjalankan

```bash
cp .env.example .env   # lalu sesuaikan
```

```env
DATABASE_URL=mysql://user:pass@localhost:3306/db_topup

# Supplier
DIGIFLAZZ_USERNAME=...
DIGIFLAZZ_APIKEY=...

# Telegram bus (kunci wajib sama dengan Server Web)
TELEGRAM_ENCRYPTION_KEY=kunci-rahasia-sama-dengan-server-web
TELEGRAM_BOT_1_TOKEN=...
TELEGRAM_BOT_2_TOKEN=...
TELEGRAM_BOT_3_TOKEN=...
TELEGRAM_GROUP_1_ID=-100xxxxxxxxxx
TELEGRAM_GROUP_2_ID=-100xxxxxxxxxx

# Sinkronisasi katalog ke web (detik)
SYNC_INTERVAL_SECONDS=300
```

```bash
# Worker utama (zero-inbound)
cargo run --bin fulfillment_service --release

# Dashboard terminal interaktif
cargo run --bin fulfillment_cli
```

### 🖥️ fulfillment_cli — Kontrol Panel Terminal

Menu interaktif langsung ke DB Topup:
1. **Dashboard** — statistik harian, revenue, grafik order 7 hari
2. **Kelola Layanan** — ubah harga, margin member/reseller, status
3. **Kelola Flashsale** — tambah/aktifkan/matikan/hapus
4. **Kelola Voucher** — kode, kategori, diskon, minimum, stok
5. **Transaksi Terbaru**
6. **User & Saldo**
7. **🔄 Sync ke Server Web** — dorong snapshot katalog+promo via Bot 3

Setelah mengubah harga/promo lewat CLI, jalankan menu **[7]** agar web langsung selaras (atau tunggu sync periodik).

## 📄 Lisensi

Proyek ini dilindungi di bawah lisensi **GNU AGPLv3**.
