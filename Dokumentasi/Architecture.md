# Architecture — DOTS HCM

**Versi Dokumen:** 1.0 · **Tanggal:** 2026-11-02

Rujukan: [[PRD]] · [[Backend Arsitektur]] · [[Frontend Arsitektur]] · [[Mobile Arsitektur]] · [[DOTS HCM Project Index]]

---

## 1. Peta Sistem

```
┌─────────────┐     ┌─────────────┐     ┌──────────────┐
│ dots-hcm-ui │     │dots-hcm-    │     │   Admin      │
│  Next.js 15 │     │  mobile     │     │  (web sama)  │
│    :3130    │     │  Expo 58    │     │              │
└──────┬──────┘     └──────┬──────┘     └──────┬───────┘
       │  HTTP + JWT       │                   │
       └──────────┬────────┴───────────────────┘
                  ▼
         ┌─────────────────┐
         │   dots-hcm API  │  Go 1.26 · :8096 · prefix /hcm/v1
         │  huma v2 + chi  │  worker: jobs per jam (kontrak H-30,
         │  + worker       │  kuota cuti tahunan, hitung absensi harian)
         └───┬─────────┬───┘
             │         │
     ┌───────▼──┐   ┌──▼─────┐
     │ Postgres │   │ MinIO  │
     │  :5432   │   │ :9010  │
     └──────────┘   └────────┘
```

Satu database per BPR (tenant): `dots_hcm_<slug>`, katalog `dots_hcm_meta`.
Multi-tenancy memakai `dots-platform/multitenancy` (`Catalog`, `Registry`,
`Router`, `Middleware`).

---

## 2. Keputusan Arsitektur (dari desain 28 Okt 2026)

| # | Keputusan | Alasan |
|---|---|---|
| A1 | **HCM produk & repo terpisah dari CBS** (`dots-hcm`, `dots-hcm-ui`, `dots-hcm-mobile`), DB sendiri per tenant | Siklus rilis & pembeli berbeda; HCM dijual juga ke BPR yang tidak memakai CBS DOTS. Integrasi dengan CBS lewat API, bukan JOIN |
| A2 | **Multi-tenant, satu DB per BPR** lewat `dots-platform/multitenancy` | Pola yang sudah terbukti di CBS dan Sobat |
| A3 | **Tanpa dots-sso; HCM punya login sendiri** (keputusan pemilik, 28 Okt 2026) | Kode BPR + username/email + password, argon2id, JWT HS256 15 menit + refresh rotasi 7 hari. Reset password oleh admin mencabut semua sesi |
| A4 | **Struktur backend menyalin pola dots-cbs-backend** | `internal/<module>/{domain,usecase,repository,service/http,adapter}`, kontrak `Module`, goose, sqlc, huma + chi, jobs |
| A5 | **Auth/storage/jobs dari `dots-platform`** | `auth.Actor`, `multitenancy`, `storage.MinIO`, `jobs`. Verifier SSO tidak dipakai; token diverifikasi modul `identity` HCM |
| A6 | **Frontend Next.js 15 + `@dots/ui` + `@dots/utils`** lewat `link:../../dots-platform-ui/packages/*`, pnpm | Sesuai pola CBS; `@dots/auth` tidak dipakai, sesi dikelola sendiri di `src/shared/lib/session.ts` |
| A7 | **Mobile Expo, self-service pegawai** | Fokus absen kantor/WFH/tugas luar/acara QR, pengajuan, profil. Sesi pakai SecureStore, lokasi + selfie |
| A8 | **Payroll dibangun paling akhir & mati secara bawaan** | Paling berisiko regulasi; diaktifkan lewat setting |

---

## 3. Pola Lintas Modul

### 3.1 Native vs custom (spec §4)

- **Native** bila kolom menentukan state, perhitungan, atau approval (status,
  tanggal masuk, unit, jabatan, grade, PTKP, NPWP, BPJS, rekening, atasan).
- **Custom** (`custom_data` JSONB via `custom_fields`) hanya untuk dicatat/
  dilaporkan (agama, golongan darah, ukuran seragam). Tidak punya FK, tidak bisa
  wajib di DB.
- Entitas extensible: `employee`, `contract`, `org.unit`, `org.position`,
  `training`, `leave.request`.
- Modul pemilik memanggil `reference.UC.ValidateCustomData` / `ValidateEditedCustomData`;
  kunci tak dikenal dibuang.

### 3.2 Approval — dua mekanisme

| Mekanisme | Kontrak | Dipakai untuk |
|---|---|---|
| **maker-checker** | `kernel/command` + `command_policies` (milik bank) | Perubahan data master sensitif: gaji, rekening, status pegawai, struktur organisasi. 1–3 penyetuju |
| **workflow atasan** | `kernel/workflow.Engine.Start` + `workflow.Owner{Kind, Decided}` di wiring | Cuti/izin/lembur, koreksi absensi. Rantai atasan di-resolve dari `reports_to`; fallback ke HR bila kosong |

Keduanya **dalam transaksi yang sama** dengan record pemiliknya, ditulis dalam
`dbtx.Runner.Run` bersama `audit.Write`.

### 3.3 Audit & optimistic lock

- Semua mutasi lewat `audit.Write` — siapa, kapan, dari mana, diff.
- Optimistic lock lewat kolom `version` → 409 "Data sudah diubah pengguna lain, muat ulang".

### 3.4 Self-service (ESS)

- Route ditandai `registry.Permission{Employee: true}`; boot menyinkronkan ke peran `EMPLOYEE`.
- Identitas pegawai selalu dari `employees.user_id` aktor, tidak pernah dari id di URL.

### 3.5 Permission

- Enum di kode, contoh: `employee.employee.read`, `employee.salary.read`, `leave.request.approve`, `payroll.run.approve`.
- Setiap route wajib salah satu: `httpx.Register` (permission) / `httpx.Authenticated` / `httpx.Public`. `archtest` menolak route tanpa itu.
- `role_permissions.permission_code` ber-FK ke `permissions` yang baru terisi saat boot — migrasi tidak bisa memberi permission ke peran.

---

## 4. Alur Request

```
Klien (web/mobile)
  │ Authorization: Bearer <JWT>
  ▼
chi router ── /hcm/v1/*
  ▼
middleware multitenancy  ── pilih pool DB dari katalog (sub claim)
  ▼
middleware auth (identity) ── verifikasi JWT HS256, isi auth.Actor
  ▼
httpx handler (huma) ── DTO validasi, refusal.* untuk error
  ▼
usecase ── aturan bisnis, orchestrasi transaksi
  ▼
dbtx.Runner.Run ── audit.Write + repository postgres (pgx/sqlc)
  ▼
Postgres tenant
```

Error ke klien selalu `refusal.*` (pesan Indonesia); error lain jadi 500 generik
dan dicatat di log. Data pegawai selalu dicek cakupan unit.

---

## 5. Jobs (worker)

Worker jalan tiap jam + sekali saat start (`make worker`):

| Job | Fungsi |
|---|---|
| Peringatan kontrak H-30 | Kontrak akan berakhir 30 hari lagi |
| Kuota cuti tahunan | GRANT awal tahun, EXPIRE hanguskan sisa di atas `carry_over_max` |
| Hitung absensi harian | `attendance_days` dari punches + jadwal + cuti yang disetujui |

Job idempoten per (pegawai, tahun, jenis) — aman diulang.

---

## 6. Referensi

[[DOTS HCM Project Index]] · [[PRD]] · [[Backend Arsitektur]] · [[Frontend Arsitektur]] · [[Mobile Arsitektur]] · [[Schema]] · [[Rules]]
