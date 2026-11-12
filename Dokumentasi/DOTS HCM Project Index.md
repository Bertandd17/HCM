# DOTS HCM Project Index

**DOTS HCM** — sistem HR untuk BPR. **Empat repo** (BE, web FE, mobile, library).

📍 `D:/Semester 7/Magang/HCM/Pengerjaan/`
🗓️ Diperbarui: 2 November 2026
✅ Status: **berjalan** di laptop ini

---

## Mulai dari sini

| Kalau Anda mau… | Baca |
|---|---|
| **Menjalankan** | [[Cara Menjalankan]] |
| **Memahami bentuk sistem** | [[Architecture]] |
| **Memahami aturan yang mengikat** | [[Rules]] |
| **Memahami produknya** | [[PRD]] |
| **Menambah fitur ke depan** | [[Roadmap]] · [[Backlog]] |
| **Melihat tabel dan kolom** | [[Schema]] |
| **Tahu apa yang belum selesai** | [[Known Debt]] |
| **Tahu kenapa begitu** | [[Decision Log]] |

---

## Dokumen

### Dasar
- **[[Cara Menjalankan]]** — cara nyalakan stack, akun, import data, troubleshooting
- **[[Architecture]]** — arsitektur teknis, modul, alur request, pattern
- **[[Backend Arsitektur]]** — arsitektur backend Go, modul, rute HTTP
- **[[Frontend Arsitektur]]** — arsitektur web Next.js, routes, fitur
- **[[Mobile Arsitektur]]** — arsitektur aplikasi HP Expo, layar, alur absen
- **[[Schema]]** — tabel, constraint, index, jebakan skema
- **[[Rules]]** — konvensi backend/frontend/database per repo
- **[[Design]]** — desain produk & antarmuka
- **[[Roadmap]]** — peta fitur fase, urutan kebergantungan
- **[[Backlog]]** — fitur yang sudah ada vs belum dibuat (dari `BACKLOG-DOTS-HCM.xlsx`)

### Rujukan
- **[[Known Debt]]** — yang belum selesai, yang menunggu keputusan manusia
- **[[Decision Log]]** — keputusan arsitektur, yang menunggu
- **[[Skill Setup]]** — setup agent skills, workflow sebelum mengerjakan fitur

---

## Gambaran satu halaman

```
DOTS HCM  ·  HR for BPR
     struktur organisasi → pegawai → absensi → cuti/lembur → kinerja → pelatihan → payroll → laporan
```

| | |
|---|---|
| **Pengguna** | HR, atasan, pegawai (self-service via `/me`) |
| **Backend** | Go 1.26 · chi + huma v2 · pgx/v5 · goose · sqlc · River jobs · multi-tenant |
| **Web FE** | Next.js 15 App Router · React 19 · Tailwind 4 · TanStack Query 5 · `@dots/ui` |
| **Mobile** | Expo SDK 58 · React Native 0.88 · Expo Router · self-service absen |
| **Database** | PostgreSQL (lokal dev :5432) · satu DB per tenant `dots_hcm_<slug>` |
| **Lampiran** | MinIO (bucket priv per tenant `dots-hcm-<slug>`) |
| **Port** | API `:8096` (`/hcm/v1`) · UI `:3130` · Postgres `:5432` · MinIO `:9010` |
| **Modul** | identity, reference, org, employee, leave, attendance, performance, training, payroll, report, workflow, command, document |

---

## Repositori

```
Pengerjaan/
├── dots-hcm/          BE — Go 1.26 (root module github.com/DotsCore/dots-hcm)
├── dots-hcm-ui/       Web FE — Next.js 15 (repo terpisah)
├── dots-hcm-mobile/   Mobile — Expo SDK 58 (repo terpisah)
├── dots-platform/     Library Go bersama (WAJIB sibling, read-only)
├── dots-platform-ui/  @dots/ui design system (WAJIB sibling, read-only)
└── Dokumentasi/       Dokumentasi ini + BACKLOG-DOTS-HCM.xlsx
```

Library `dots-platform` dan `dots-platform-ui` serta semua repo CBS/CBSV2 **read-only** — jangan diedit.

---

## Tiga hal yang paling sering bikin salah

1. **Baca `dots-hcm/CLAUDE.md` dulu setiap mau kerja.** Backend memakai per-modul `internal/<modul>/{domain,usecase,repository/postgres,service/http,adapter}`; ada puluhan jebakan huma/pgx/archtest yang merusak fitur kalau dilewat.
2. **Teks UI bahasa Indonesia; identifier, path, komentar bahasa Inggris.** Web: `pnpm vitest run && pnpm typecheck && pnpm lint && pnpm build` sebelum selesai. Mobile: `npm run typecheck && npm run lint`.
3. **Jangan diperbaiki kalau bukan bug.** Daftar "Sengaja Tidak Dibuat" di [[Backlog]] adalah keputusan pemilik produk — THR tidak boleh ditahan, payroll tidak bisa dibuka ulang setelah disetujui, dsb.
