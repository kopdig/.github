# KOPDIG Organization 🏪

> **Sistem Manajemen Koperasi Siswa Digital** — multi-platform: web kasir, desktop POS, dan mobile monitoring. Dibangun di atas satu backend terpusat dengan database PostgreSQL.

[![Web](https://img.shields.io/badge/Web-Next.js%2016-000?logo=next.js)](https://github.com/kopdig/kopdig-web)
[![Desktop](https://img.shields.io/badge/Desktop-Electron%2044-512BD4?logo=electron)](https://github.com/kopdig/kopdig-desktop)
[![Mobile](https://img.shields.io/badge/Mobile-Flutter-61DAFB?logo=flutter)](https://github.com/kopdig/kopdig-mobile)
[![Backend](https://img.shields.io/badge/Backend-ElysiaJS%20%2B%20Bun-green?logo=bun)](https://github.com/kopdig/kopdig-backend)
[![License](https://img.shields.io/badge/License-Proprietary-red)](LICENSE)

---

## 📊 Ringkasan Platform

| Platform   | Framework / Stack                              | Bahasa       | Status            | Fitur Utama                                                          | Repository                        |
| ---------- | ---------------------------------------------- | ------------ | ----------------- | -------------------------------------------------------------------- | --------------------------------- |
| 🌐 Web     | Next.js 16 App Router + Tailwind CSS 4         | TypeScript   | Fondasi jalan ✅   | Kasir, auth, cart, design system, Docker ready                       | [kopdig-web](./kopdig-web)        |
| 🖥️ Desktop | Electron 44 + React 19 + Vite 7                | TypeScript   | Scaffold ⚙️        | Struktur IPC aman, context isolation, API layer boundary reserved      | [kopdig-desktop](./kopdig-desktop) |
| 📱 Mobile  | Flutter (default template)                     | Dart         | Kosong 🏗️         | Belum ada fitur — hanya skeleton starter                             | [kopdig-mobile](./kopdig-mobile)  |
| ⚙️ Backend | ElysiaJS + Bun + Supabase Postgres             | TypeScript   | Siap dipakai ✅    | Auth, produk, shift, transaksi, ledger, laporan, OTP, audit logs     | [kopdig-backend](./kopdig-backend) |
| 🧪 E2E     | Playwright + Bun                               | TypeScript   | Web & Desktop 🟢  | Smoke test Chromium/Firefox/WebKit, desktop dev+packaged             | [kopdig-e2e-testing](./kopdig-e2e-testing) |

---

## 🎯 Tentang KOPDIG

**KOPDIG (Koperasi Digital)** adalah sistem manajemen koperasi siswa sekolah yang menggantikan pencatatan manual. Cakupan: penjualan langsung (kasir), manajemen stok, cicilan seragam, barang titipan, dan laporan keuangan.

### Capaian Inti

- **Arsitektur satu backend, tiga klien** — Web, Desktop, Mobile berbagi API Go/SupaBase yang sama
- **Kontrol akses berbasis peran** — superadmin, admin, petugas (kasir), bendahara, kepala sekolah
- **Transaksi real-time** — shift kas, penjualan tunai & QRIS manual, pencatatan stok
- **Audit lengkap** — log per transaksi, koreksi stok, riwayat shift
- **Desain sistem konsisten** — skema warna emerald, komponen shadcn/ui di web

### Batasan MVP

| Termasuk MVP                           | Tidak termasuk MVP                   |
| -------------------------------------- | ------------------------------------ |
| Transaksi kasir cepat                  | Simpan-pinjam koperasi               |
| Master produk + kategori + harga       | Integrasi payment gateway otomatis   |
| Manajemen stok (masuk/keluar/adjust)   | Marketplace atau pembelian mandiri   |
| Cicilan seragam + sisa tagihan         | Autentikasi biometrik                |
| Barang titipan (konsinyasi)            | Multi-cabang / multi-koperasi        |
| Laporan harian + stok + kas            | Notifikasi push                      |

---

## ⚙️ Backend (Source of Truth)

Single backend **ElysiaJS + Bun + Supabase Postgres** melayani ketiga klien. Feature-based module structure: tiap fitur punya `index.ts` (routing + validasi + guard), `service.ts` (logika bisnis), `model.ts` (skema Zod).

### Modul yang sudah berjalan

| Modul       | Endpoint / Fungsi                                  | Status   |
| ----------- | -------------------------------------------------- | -------- |
| **auth**    | Login (username), token issuance, Supabase Auth map | ✅       |
| **users**   | CRUD pengguna, role assignment, aktivas nonaktif   | ✅       |
| **products**| CRUD produk, barcode, konsinyasi, harga jual/beli  | ✅       |
| **shifts**  | Buka/tutup shift kas, one-active-shift constraint  | ✅       |
| **transactions** | Buat transaksi, simpan item snapshot harga    | ✅       |
| **ledger**  | Buku kas masuk/keluar, rekonsiliasi                | ✅       |
| **reports** | Ringkasan penjualan, stok, shift                   | ✅       |
| **settings**| Konfigurasi sistem (nama toko, footer, margin)     | ✅       |
| **otp**     | Kode OTP satu kali untuk verifikasi                | ✅       |
| **audit-logs** | Append-only log semua operasi                   | ✅       |

### Skema Database (Supabase)

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

### CRUD Produk — Bidang Kunci

| Field           | Tipe            | Constraint                                  |
| --------------- | --------------- | ------------------------------------------- |
| `id`            | UUID            | Primary key                                 |
| `barcode`       | TEXT            | UNIQUE, nullable                            |
| `name`          | TEXT            | NOT NULL                                    |
| `is_consignment`| BOOLEAN         | Default false                               |
| `buy_price`     | NUMERIC(14,2)   | > 0                                       |
| `sell_price`    | NUMERIC(14,2)   | > 0                                       |
| `stock`         | INTEGER         | >= 0, default 5                             |
| `min_stock`     | INTEGER         | default 5                                   |
| `expiry_date`   | DATE            | nullable (untuk produk perishable)          |

**Konsinyasi check**: `margin_percent` wajib 10–15% bila `is_consignment = true`.

### Keamanan

- **Publishable key** → akses user-scoped; semua *read* backend pakai publishable key + JWT, RLS berlaku penuh
- **Secret key** → server-only, hanya dipakai untuk Auth Admin API, write transaksional, audit log
- **Peran ganda**: RLS baca `app_metadata.role` di JWT; backend cek fresh dari `app.users` tiap request
- **RLS aktif** di semua tabel + 22 policies mencerminkan matriks peran PRD

### Kredensial Development

| Peran      | Username | Password |
| ---------- | -------- | -------- |
| **superadmin** | `admin123`    | `12345`   |
| **petugas**    | `G123`      | `12345`   |
| —              | `123`       | `12345`   |

---

## 🌐 Aplikasi Web

Client-side kasir & admin dengan Next.js 16 App Router. **Fondasi UI & alur kasir sudah jalan**; backend API belum terhubung penuh (lihat [Open Decisions](kopdig-web/docs/05-decisions/open-decisions.md)).

### Fitur Web

- **Sistem Desain Emerald** — warna utama `#059669`, komponen shadcn/ui, glassmorphism opsional
- **Halaman Kasir** — pencarian produk via barcode/nama, keranjang Zustand, hitung total, bayar tunai/QRIS manual
- **Auth Flow** — login, register, forgot/reset password (UI siap; backend integrasi TBD)
- **Dashboard** — layout admin dengan sidebar navigasi berbasis role
- **Docker Support** — multi-stage build; production runner `node:24-alpine`
- **Validasi Zod** — env validated di import; API schemas di boundary

### Tumpukan Teknologi Web

| Komponen      | Teknologi                |
| ------------- | ------------------------ |
| **Framework** | Next.js 16 App Router    |
| **Runtime**   | React 19 + TypeScript 5  |
| **Gaya**      | Tailwind CSS 4           |
| **State**     | Zustand 5 (keranjang), TanStack React Query 5 (server state) |
| **Validasi**  | Zod 4                    |
| **Animasi**   | Framer Motion            |
| **Build**     | bun 1.3+                 |
| **Container** | Docker multi-stage       |

### Jalur Lokal

```bash
cd kopdig-web
bun install
bun run dev
# → http://localhost:3000
```

### Quality Gate

```bash
bun run validate   # typecheck + lint + format + docs + test:coverage + build + size
bun run audit:fast # oxlint + typecheck (cepat)
```

---

## 🖥️ Aplikasi Desktop

**Electron 44 + React 19 + Vite 7 + Tailwind CSS 4**. Saat ini dalam tahap **scaffold**: struktur process isolation sudah diamankan, belum ada fitur bisnis.

### Arsitektur Process

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

### Security Defaults

- `contextIsolation: true`
- `nodeIntegration: false`
- `sandbox: true`

Tidak ada IPC channel yang terdaftar — tambahan channel hanya jika butuh akses desktop-native yang memang diperlukan.

### Tumpukan Teknologi Desktop

| Komponen       | Teknologi                     |
| -------------- | ----------------------------- |
| **Shell**      | Electron 44                   |
| **Frontend**   | React 19 + TypeScript 5.9     |
| **Build**      | electron-vite 5 + Vite 7      |
| **Styling**    | Tailwind CSS 4                |
| **Packager**   | bun (satu-satunya package manager) |

### Jalur Lokal

```bash
cd kopdig-desktop
bun install
bun run dev   # electron-vite dev
```

---

## 📱 Aplikasi Mobile

**Flutter starter template** — belum ada fitur bisnis. `lib/main.dart` berisi default counter app.

### Tumpukan Teknologi Mobile

| Komponen      | Teknologi           |
| ------------- | ------------------- |
| **Framework** | Flutter             |
| **SDK**       | Dart ^3.11.5        |
| **Target**    | Android + iOS       |

### Next Step (Rencana)

1. Hubungkan ke backend (REST via dio/http)
2. Implementasi fitur monitoring kasir real-time
3. Tambah offline cache dengan Hive/SQLite

---

## 🧪 E2E Testing

Monorepo Playwright terpisah — satu source of truth untuk smoke test semua platform.

| Service     | Browser / Engine     | Coverage                          |
| ----------- | -------------------- | --------------------------------- |
| **web**     | Chromium / Firefox / WebKit | Full matrix; dev+staging Chromium-only |
| **desktop** | Chromium (Electron bundles it) | Dev mode + packaged build |
| **mobile**  | —                    | Di-skip by design (placeholder)   |

### Jalur Lokal

```bash
cd kopdig-e2e-testing
bun run setup          # install + playwright install --with-deps
cp .env.example .env
bun run doctor         # cek env vars
bun run test:smoke     # web + desktop smoke together
```

---

## 🔄 Alur Pengguna

### Kasir (Web & Desktop)

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

### Pengelola Stok

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

### Kepala Sekolah

```text
Login → Role: kepsek
     ↓
Lihat Dashboard Ringkas
     ↓
Laporan penjualan harian
     ↓
Laporan stok kritis
```

---

## 🔐 Keamanan & Autentikasi

### Kontrol Akses Berbasis Peran (Matriks)

| Peran           | Kasir | Stok | Kas/Ledger | Laporan Penuh | User Mgmt |
| --------------- | ----- | ---- | ---------- | ------------- | --------- |
| **superadmin**  | ✅    | ✅   | ✅         | ✅            | ✅        |
| **admin**       | ✅    | ✅   | ✅         | ✅            | ❌        |
| **petugas**     | ✅    | ❌   | ❌         | ❌            | ❌        |
| **bendahara**   | ❌    | ❌   | ✅         | ✅            | ❌        |
| **kepala sekolah** | ❌ | ❌   | 👁️ read  | 👁️ read       | ❌        |

### Model Auth

- Username login → backend memetakan ke email sintetis `<username>@kopdig.local` di Supabase Auth
- **Tidak perlu email** — cocok untuk lingkungan sekolah tanpa infrastruktur email
- Session duration diatur via Supabase Dashboard (bukan hardcode)
- JWT + refresh token (web); bearer token (desktop/mobile)

### RLS Policies (Ringkasan)

| Tabel           | Read oleh            | Write oleh                  |
| --------------- | -------------------- | --------------------------- |
| `users`         | admin/superadmin     | superadmin                  |
| `products`      | semua role           | admin/superadmin            |
| `shifts`        | admin/superadmin     | petugas (buka), admin (tutup) |
| `transactions`  | semua role (scoped)  | petugas (buka)              |
| `ledger`        | bendahara/admin/superadmin | admin/superadmin       |
| `audit_logs`    | admin/superadmin     | trigger (append-only)       |

---

## 📊 Perbandingan Fitur per Platform

| Fitur                   | Web  | Desktop | Mobile |
| ----------------------- | ---- | ------- | ------ |
| Kasir / input transaksi | ✅   | ⚙️ (scaffold) | ❌     |
| Autentikasi             | ✅   | ✅ (UI scaffold) | ❌     |
| Manajemen produk        | ✅   | ❌       | ❌     |
| Manajemen stok          | ✅   | ❌       | ❌     |
| Shift kas               | ✅   | ❌       | ❌     |
| Laporan                 | ✅   | ❌       | ❌     |
| Docker / production     | ✅   | ❌       | ❌     |
| E2E test coverage       | ✅   | ✅       | ❌     |

---

## 🛠️ Pengaturan Pengembangan

### Prasyarat Umum

- **Bun** 1.3+ (package manager & runtime — wajib)
- **Node.js** 24+ (hanya untuk runner container web)
- **Supabase** account + project `dbccibeljdaotavnsslx` (ap-southeast-1)
- **Docker** (opsional, untuk web deployment)
- **Flutter SDK** (opsional, untuk mobile)

### Variabel Lingkungan

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

---

## 🗄️ Struktur Organisasi

```
kopdig/
├── .github/                 ← profil org (file ini)
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

---

## 🚀 Rencana Masa Depan

- [ ] **Desktop fitur** — integrasi API backend, halaman kasir, manajemen produk, shift
- [ ] **Mobile implementasi** — monitoring shift real-time, notifikasi kas, login
- [ ] **Web backend connection** — hubungkan mock client ke ElysiaJS API sebenarnya
- [ ] **E2E full suite** — tambah test kasus auth, kasir flow, admin CRUD
- [ ] **Multi-bahasa (i18n)** — dukungan bahasa Indonesia + Inggris
- [ ] **Dark mode toggle** — sakelar tema gelap/terang di semua platform
- [ ] **Mobile e2e** — Flutter Driver atau integration test untuk mobile smoke

---

## 🙏 Tech Stack Ringkasan

| Layer       | Teknologi                                         |
| ----------- | ------------------------------------------------- |
| **Backend** | ElysiaJS 1.4 + Bun + Supabase Postgres (RLS)      |
| **Web**     | Next.js 16 + React 19 + TypeScript + Tailwind 4 + Zustand + React Query |
| **Desktop** | Electron 44 + React 19 + TypeScript + Vite 7      |
| **Mobile**  | Flutter (target: Android + iOS)                    |
| **Testing** | Vitest 5 + Testing Library (web), Playwright (e2e) |
| **DevOps**  | Docker multi-stage, GitHub Actions (CI), husky    |
| **Lint**    | oxlint + ESLint 9 + Biome (format), Knip (dead code) |

---

Mempercepat operasional koperasi sekolah melalui digitalisasi kasir, stok, dan laporan.
