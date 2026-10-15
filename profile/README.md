# KOPDIG

> Sistem manajemen koperasi siswa digital untuk web, desktop, dan mobile. Ketiganya berbagi satu backend dan satu database PostgreSQL.

[![Web](https://img.shields.io/badge/Web-Next.js%2016-000?logo=next.js)](https://github.com/kopdig/kopdig-web)
[![Desktop](https://img.shields.io/badge/Desktop-Electron%2044-512BD4?logo=electron)](https://github.com/kopdig/kopdig-desktop)
[![Mobile](https://img.shields.io/badge/Mobile-Flutter-61DAFB?logo=flutter)](https://github.com/kopdig/kopdig-mobile)
[![Backend](https://img.shields.io/badge/Backend-ElysiaJS%20%2B%20Bun-green?logo=bun)](https://github.com/kopdig/kopdig-backend)
[![License](https://img.shields.io/badge/License-Proprietary-red)](LICENSE)

## Ringkasan platform

| Platform | Framework / Stack                      | Bahasa     | Status                | Fitur utama                                                       | Repository                                                         |
| -------- | -------------------------------------- | ---------- | --------------------- | ----------------------------------------------------------------- | ------------------------------------------------------------------ |
| Web      | Next.js 16 App Router + Tailwind CSS 4 | TypeScript | Fondasi jalan         | Kasir, auth, cart, design system, Docker ready                    | [kopdig-web](https://github.com/kopdig/kopdig-web)                 |
| Desktop  | Electron 44 + React 19 + Vite 7        | TypeScript | Scaffold              | Struktur IPC aman, context isolation, API layer boundary reserved | [kopdig-desktop](https://github.com/kopdig/kopdig-desktop)         |
| Mobile   | Flutter (default template)             | Dart       | Kosong                | Belum ada fitur, masih skeleton starter                           | [kopdig-mobile](https://github.com/kopdig/kopdig-mobile)           |
| Backend  | ElysiaJS + Bun + Supabase Postgres     | TypeScript | Siap dipakai          | Auth, produk, shift, transaksi, ledger, laporan, OTP, audit logs  | [kopdig-backend](https://github.com/kopdig/kopdig-backend)         |
| E2E      | Playwright + Bun                       | TypeScript | Jalan (web + desktop) | Smoke test Chromium/Firefox/WebKit, desktop dev+packaged          | [kopdig-e2e-testing](https://github.com/kopdig/kopdig-e2e-testing) |

## Tentang KOPDIG

KOPDIG (Koperasi Digital) menggantikan pencatatan manual koperasi siswa. Cakupannya penjualan di kasir, manajemen stok, cicilan seragam, barang titipan, dan laporan keuangan.

### Capaian inti

- Satu backend melayani tiga klien: web, desktop, dan mobile.
- Akses diatur per peran: superadmin, admin, petugas (kasir), bendahara, dan kepala sekolah.
- Shift kas, pembayaran tunai dan QRIS manual, serta pergerakan stok tercatat langsung.
- Transaksi, koreksi stok, dan riwayat shift masuk ke log audit.
- Web memakai skema warna emerald dengan komponen shadcn/ui.

### Batasan MVP

| Termasuk MVP                         | Tidak termasuk MVP                 |
| ------------------------------------ | ---------------------------------- |
| Transaksi kasir cepat                | Simpan-pinjam koperasi             |
| Master produk + kategori + harga     | Integrasi payment gateway otomatis |
| Manajemen stok (masuk/keluar/adjust) | Marketplace atau pembelian mandiri |
| Cicilan seragam + sisa tagihan       | Autentikasi biometrik              |
| Barang titipan (konsinyasi)          | Multi-cabang / multi-koperasi      |
| Laporan harian + stok + kas          | Notifikasi push                    |

## Backend

Satu backend ElysiaJS + Bun + Supabase Postgres melayani ketiga klien. Struktur modulnya per fitur: `index.ts` untuk routing, validasi, dan guard; `service.ts` untuk logika bisnis; `model.ts` untuk skema Zod.

### Modul

| Modul          | Fungsi                                                            |
| -------------- | ----------------------------------------------------------------- |
| `auth`         | Login pakai username, penerbitan token, pemetaan ke Supabase Auth |
| `users`        | CRUD pengguna, penetapan peran, aktif/nonaktif                    |
| `products`     | CRUD produk, barcode, konsinyasi, harga jual dan beli             |
| `shifts`       | Buka dan tutup shift kas, satu shift aktif                        |
| `transactions` | Buat transaksi, simpan snapshot harga item                        |
| `ledger`       | Buku kas masuk dan keluar, rekonsiliasi                           |
| `reports`      | Ringkasan penjualan, stok, dan shift                              |
| `settings`     | Konfigurasi sistem (nama toko, footer, margin)                    |
| `otp`          | Kode OTP sekali pakai untuk verifikasi                            |
| `audit-logs`   | Log append-only untuk semua operasi                               |

### Skema database

```
public.app (application schema)
├── users          — akun + role (superadmin/admin/petugas)
├── products       — item koperasi + harga beli/jual + flag konsinyasi
├── shifts         — periode kas dengan opening/closing cash
├── transactions   — header transaksi
├── transaction_items — baris transaksi (snapshot harga)
├── ledger_entries — catatan kas masuk/keluar
├── audit_logs     — append-only log (trigger)
└── settings       — kunci nilai JSONB

public.auth (Supabase bawaan)
├── users          — identitas login
└── sessions       — umur token diatur via Dashboard
```

### Kolom produk

| Field            | Tipe          | Constraint                         |
| ---------------- | ------------- | ---------------------------------- |
| `id`             | UUID          | Primary key                        |
| `barcode`        | TEXT          | UNIQUE, nullable                   |
| `name`           | TEXT          | NOT NULL                           |
| `is_consignment` | BOOLEAN       | Default false                      |
| `buy_price`      | NUMERIC(14,2) | > 0                                |
| `sell_price`     | NUMERIC(14,2) | > 0                                |
| `stock`          | INTEGER       | >= 0, default 5                    |
| `min_stock`      | INTEGER       | default 5                          |
| `expiry_date`    | DATE          | nullable (untuk produk perishable) |

Kolom `margin_percent` wajib 10 sampai 15 persen bila `is_consignment` bernilai true.

### Keamanan

- Semua pembacaan backend memakai publishable key bersama JWT pengguna, sehingga RLS berlaku sebagai user itu.
- Secret key hanya dipakai di server untuk Auth Admin API, penulisan transaksional, dan audit log.
- Policy RLS membaca `app_metadata.role` dari JWT, sedangkan backend mengecek peran terbaru dari `app.users` pada setiap request.
- RLS aktif di semua tabel, dengan 22 policy yang mengikuti matriks peran PRD.

### Kredensial development

| Peran      | Username   | Password |
| ---------- | ---------- | -------- |
| superadmin | `admin123` | `12345`  |
| petugas    | `G123`     | `12345`  |
| n/a        | `123`      | `12345`  |

## Web

Halaman kasir dan admin dibangun dengan Next.js 16 App Router. Fondasi UI dan alur kasir sudah jalan, tapi API backend belum terhubung penuh (lihat [Open Decisions](https://github.com/kopdig/kopdig-web/blob/main/docs/05-decisions/open-decisions.md)).

### Fitur

- Skema warna emerald dengan warna utama `#059669`, komponen shadcn/ui, dan glassmorphism opsional.
- Halaman kasir: cari produk lewat barcode atau nama, keranjang Zustand, hitung total, bayar tunai atau QRIS manual.
- Alur autentikasi: login, register, lupa dan reset kata sandi. UI siap, integrasi backend belum.
- Dashboard admin dengan sidebar yang menyesuaikan peran.
- Build Docker multi-stage dengan runner produksi `node:24-alpine`.
- Environment dan skema API divalidasi dengan Zod di batas modul.

### Tumpukan teknologi

| Komponen  | Teknologi                                                    |
| --------- | ------------------------------------------------------------ |
| Framework | Next.js 16 App Router                                        |
| Runtime   | React 19 + TypeScript 5                                      |
| Gaya      | Tailwind CSS 4                                               |
| State     | Zustand 5 (keranjang), TanStack React Query 5 (server state) |
| Validasi  | Zod 4                                                        |
| Animasi   | Framer Motion                                                |
| Build     | bun 1.3+                                                     |
| Container | Docker multi-stage                                           |

### Jalur lokal

```bash
cd kopdig-web
bun install
bun run dev
# → http://localhost:3000
```

### Quality gate

```bash
bun run validate   # typecheck + lint + format + docs + test:coverage + build + size
bun run audit:fast # oxlint + typecheck (cepat)
```

## Desktop

Electron 44, React 19, Vite 7, dan Tailwind CSS 4. Tahapnya masih scaffold: isolasi proses sudah diamankan, fitur bisnis belum ada.

### Arsitektur proses

```
Electron Main Process (src/main/)
        ↓
Preload / IPC Bridge (src/preload/)   ← contextBridge, satu-satunya surface Node/Electron
        ↓
React Renderer (src/renderer/)        ← UI, never touches DB or privileged APIs directly
        ↓
API Service Layer (src/renderer/src/services/)  ← boundary reserved, empty, belum diimplementasi
        ↓
Backend (ElysiaJS)
```

### Setelan keamanan

- `contextIsolation: true`
- `nodeIntegration: false`
- `sandbox: true`

Belum ada IPC channel yang terdaftar. Channel baru hanya ditambah bila memang butuh akses desktop-native.

### Tumpukan teknologi

| Komponen | Teknologi                          |
| -------- | ---------------------------------- |
| Shell    | Electron 44                        |
| Frontend | React 19 + TypeScript 5.9          |
| Build    | electron-vite 5 + Vite 7           |
| Styling  | Tailwind CSS 4                     |
| Packager | bun (satu-satunya package manager) |

### Jalur lokal

```bash
cd kopdig-desktop
bun install
bun run dev   # electron-vite dev
```

## Mobile

Masih memakai template awal Flutter. `lib/main.dart` berisi aplikasi counter bawaan dan belum ada fitur bisnis.

### Tumpukan teknologi

| Komponen  | Teknologi     |
| --------- | ------------- |
| Framework | Flutter       |
| SDK       | Dart ^3.11.5  |
| Target    | Android + iOS |

### Langkah berikutnya

1. Hubungkan ke backend lewat REST (dio atau http).
2. Tambahkan monitoring kasir.
3. Tambahkan cache offline dengan Hive atau SQLite.

## E2E testing

Repo terpisah berisi smoke test Playwright untuk semua platform.

| Service | Browser / Engine            | Coverage                                      |
| ------- | --------------------------- | --------------------------------------------- |
| web     | Chromium / Firefox / WebKit | Matriks penuh; dev dan staging hanya Chromium |
| desktop | Chromium (bawaan Electron)  | Mode dev dan build packaged                   |
| mobile  | n/a                         | Sengaja dilewati, hanya placeholder           |

### Jalur lokal

```bash
cd kopdig-e2e-testing
bun run setup          # install + playwright install --with-deps
cp .env.example .env
bun run doctor         # cek env vars
bun run test:smoke     # web + desktop smoke together
```

## Alur pengguna

### Kasir (web dan desktop)

```text
Login (username + password)
     ↓
Buka Shift Kas (tandai opening cash)
     ↓
Cari Produk (barcode / nama)
     ↓
Tambah ke Keranjang
     ↓
Pilih Pembayaran (Tunai / QRIS manual)
     ↓
Konfirmasi + Cetak Struk
     ↓
Tutup Shift (bandingkan expected vs actual cash)
```

### Pengelola stok

```text
Login → Role: admin/superadmin
     ↓
Kelola Produk (CRUD, scan barcode)
     ↓
Input Stok Masuk / Keluar / Penyesuaian
     ↓
Lihat Alert stok < min_stock
     ↓
Export laporan stok (CSV)
```

### Bendahara

```text
Login → Role: bendahara
     ↓
Pantau Shift Aktif
     ↓
Cek Ledger (masuk/keluar)
     ↓
Rekonsiliasi Cash
     ↓
Unduh Laporan Harian/Mingguan
```

### Kepala sekolah

```text
Login → Role: kepsek
     ↓
Lihat Dashboard Ringkas
     ↓
Laporan penjualan harian
     ↓
Laporan stok kritis
```

## Keamanan dan autentikasi

### Matriks peran

| Peran          | Kasir | Stok | Kas/Ledger | Laporan Penuh | User Mgmt |
| -------------- | ----- | ---- | ---------- | ------------- | --------- |
| superadmin     | ✅    | ✅   | ✅         | ✅            | ✅        |
| admin          | ✅    | ✅   | ✅         | ✅            | ❌        |
| petugas        | ✅    | ❌   | ❌         | ❌            | ❌        |
| bendahara      | ❌    | ❌   | ✅         | ✅            | ❌        |
| kepala sekolah | ❌    | ❌   | 👁️ read    | 👁️ read       | ❌        |

### Model auth

- Kasir masuk dengan username. Backend memetakannya ke email sintetis `<username>@kopdig.local` di Supabase Auth.
- Tidak perlu email, jadi cocok untuk sekolah tanpa infrastruktur email.
- Umur sesi diatur di Supabase Dashboard, bukan di kode.
- Web memakai JWT dengan refresh token; desktop dan mobile memakai bearer token.

### Policy RLS

| Tabel          | Read oleh                  | Write oleh                    |
| -------------- | -------------------------- | ----------------------------- |
| `users`        | admin/superadmin           | superadmin                    |
| `products`     | semua role                 | admin/superadmin              |
| `shifts`       | admin/superadmin           | petugas (buka), admin (tutup) |
| `transactions` | semua role (scoped)        | petugas (buka)                |
| `ledger`       | bendahara/admin/superadmin | admin/superadmin              |
| `audit_logs`   | admin/superadmin           | trigger (append-only)         |

## Perbandingan fitur per platform

| Fitur                   | Web | Desktop     | Mobile |
| ----------------------- | --- | ----------- | ------ |
| Kasir / input transaksi | ✅  | Scaffold    | ❌     |
| Autentikasi             | ✅  | UI scaffold | ❌     |
| Manajemen produk        | ✅  | ❌          | ❌     |
| Manajemen stok          | ✅  | ❌          | ❌     |
| Shift kas               | ✅  | ❌          | ❌     |
| Laporan                 | ✅  | ❌          | ❌     |
| Docker / production     | ✅  | ❌          | ❌     |
| E2E test coverage       | ✅  | ✅          | ❌     |

## Pengaturan pengembangan

### Prasyarat

- Bun 1.3 atau lebih baru (package manager sekaligus runtime).
- Node.js 24 atau lebih baru, hanya untuk runner container web.
- Akun Supabase dengan project `dbccibeljdaotavnsslx` (ap-southeast-1).
- Docker, opsional, untuk deploy web.
- Flutter SDK, opsional, untuk mobile.

### Variabel lingkungan

**Backend** (`.env`):

```env
SUPABASE_URL=https://dbccibeljdaotavnsslx.supabase.co
SUPABASE_PUBLISHABLE_KEY=your-publishable-key
SUPABASE_SECRET_KEY=your-secret-key
```

**Web** (`.env.local`):

```env
NEXT_PUBLIC_API_URL=http://localhost:3000/api/v1
```

## Struktur repo

```
kopdig/
├── .github/                 ← profil org
├── kopdig-backend/          ← ElysiaJS + Supabase (10 modul siap)
│   ├── src/modules/         ← auth, users, products, shifts, transactions, ledger, reports, settings, otp, audit-logs
│   └── supabase/migrations/ ← 0001_init.sql (DDL lengkap + RLS)
├── kopdig-web/              ← Next.js 16 + React 19 (kasir foundation jalan)
│   ├── src/app/             ← route: /, /kasir, /(auth)/*, /(dashboard)/*
│   ├── src/components/ui/   ← 9 komponen shadcn-based
│   └── src/features/kasir/  ← useCartStore, useKasirFilter, types
├── kopdig-desktop/          ← Electron 44 scaffold (IPC aman, belum ada fitur)
│   └── src/
│       ├── main/            ← BrowserWindow + lifecycle
│       ├── preload/         ← contextBridge
│       └── renderer/        ← React entry, services/ (reserved)
├── kopdig-mobile/           ← Flutter starter template (kosong)
└── kopdig-e2e-testing/      ← Playwright monorepo (web + desktop smoke)
    ├── src/tests/web/       ← LoginPage, HomePage, smoke.spec.ts
    ├── src/tests/desktop/   ← BaseWindow, PosPage, smoke.spec.ts
    └── src/tests/mobile/    ← skip placeholder
```

## Rencana

- [ ] Desktop: integrasi API backend, halaman kasir, manajemen produk, dan shift.
- [ ] Mobile: monitoring shift, notifikasi kas, dan login.
- [ ] Web: hubungkan klien mock ke API ElysiaJS yang sebenarnya.
- [ ] E2E: tambah skenario autentikasi, alur kasir, dan CRUD admin.
- [ ] Dukungan dua bahasa (Indonesia dan Inggris).
- [ ] Sakelar tema gelap dan terang di semua platform.
- [ ] E2E mobile dengan Flutter Driver atau integration test.

## Ringkasan tech stack

| Layer   | Teknologi                                                               |
| ------- | ----------------------------------------------------------------------- |
| Backend | ElysiaJS 1.4 + Bun + Supabase Postgres (RLS)                            |
| Web     | Next.js 16 + React 19 + TypeScript + Tailwind 4 + Zustand + React Query |
| Desktop | Electron 44 + React 19 + TypeScript + Vite 7                            |
| Mobile  | Flutter (target: Android + iOS)                                         |
| Testing | Vitest 5 + Testing Library (web), Playwright (e2e)                      |
| DevOps  | Docker multi-stage, GitHub Actions (CI), husky                          |
| Lint    | oxlint + ESLint 9 + Biome (format), Knip (dead code)                    |
