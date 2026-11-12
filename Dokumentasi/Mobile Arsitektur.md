# Mobile Arsitektur — dots-hcm-mobile

**Versi Dokumen:** 1.1 · **Tanggal:** 2026-11-02  
**Repo:** `dots-hcm-mobile/FE/` (Expo mobile) + `dots-hcm-mobile/BE/` (Go backend)  
**Rujukan:** [[Architecture]] · [[PRD]] · [[DOTS HCM Project Index]]  
**Sumber kebenaran:** `FE/AGENTS.md` + `BE/CLAUDE.md` — Expo berubah tiap SDK; jangan percaya ingatan.

---

## 1. Struktur Repo

```
dots-hcm-mobile/
├── FE/                          # Aplikasi mobile Expo (Employee Self-Service)
│   ├── app/                     route Expo Router (setiap file = layar)
│   │   ├── _layout.tsx          root navigator
│   │   ├── login.tsx            login + ganti password
│   │   ├── (tabs)/              tab utama: index, requests, history, profile
│   │   ├── attendance/          office, wfh, field, event (scan QR), confirmation, success
│   │   ├── requests/           new, [id]
│   │   ├── history/[id]         riwayat pengajuan
│   │   ├── approvals/           kotak masuk persetujuan atasan
│   │   ├── profile/             info, edit, schedule, settings
│   │   ├── payslips/            slip gaji + unduh PDF
│   │   ├── loans/               kasbon
│   │   ├── claims/              reimbursement
│   │   ├── overtime/            lembur
│   │   ├── corrections/         koreksi absensi
│   │   ├── performance/        penilaian kinerja
│   │   ├── discipline/          surat peringatan
│   │   ├── notifications.tsx
│   │   └── announcements.tsx
│   ├── features/               logika fitur (attendance/verification.ts, requests/validation.ts)
│   ├── services/api/           klien API per modul (client, hcm, leave, modes, payroll, dll.)
│   ├── components/             komponen bersama (ui/, forms/, attendance/, brand/)
│   ├── hooks/                  hooks React
│   ├── utils/                  date.ts, geo.ts (jarak/akurasi GPS), money.ts, retry.ts
│   ├── types/                  tipe bersama
│   ├── constants/              theme
│   ├── app.config.ts          Expo config
│   ├── package.json            deps + scripts
│   └── tsconfig.json
├── BE/                          # Backend Go (DOTS HCM API)
│   ├── cmd/{api,migrate,provision,seed,worker}/   binary
│   ├── internal/               modul: attendance, leave, identity, workflow, dll.
│   ├── migrations/             migrasi goose
│   └── go.mod                  replace ke ../../dots-platform
└── Dokumentasi/                dokumentasi proyek
```

**Modul ESS Mobile (FE/):**

| Layar | Fungsi | Status |
|---|---|---|
| Login + ganti password | Akun sama dengan web | ✅ |
| Absen kantor | GPS + radius + selfie + jam buka | ✅ |
| Absen WFH / tugas luar | Dengan persetujuan atasan | ✅ |
| Absen acara | Scan QR + undangan | ✅ |
| Riwayat absen | Status hari, telat/alpa/lembur | ✅ |
| Pengajuan cuti/izin/lembur | Lihat saldo, ajukan, lampiran | ✅ |
| Kotak masuk persetujuan | Keputusan + alasan atasan | ✅ |
| Slip gaji | Lihat + unduh PDF | ✅ |
| Kasbon | Ajukan + lacak | ✅ |
| Reimbursement | Ajukan + lacak | ✅ |
| Koreksi absensi | Ajukan + lacak | ✅ |
| Penilaian kinerja | Self-assessment + lihat hasil | ✅ |
| Surat peringatan | Lihat, konfirmasi, keberatan | ✅ |
| Profil sendiri | Lihat + ubah, jadwal kerja | ✅ |
| Notifikasi + pengumuman | In-app | ✅ |
| Pelatihan | Lihat + lacak kewajiban | ✅ |
| Dokumentasi | Unduh | ✅ |

Daftar lengkap: [[Backlog]] kolom "Aplikasi HP".

---

## 2. Konvensi FE

- **Expo Router** untuk semua navigasi. Route di `app/`; kode non-route di luar `app/`.
- Import `Link`, `router`, `useLocalSearchParams` dari `expo-router`.
- `@/` resolved ke root `FE/` (bukan `src/`).
- Sesi di `expo-secure-store`.
- Verifikasi absen di `features/attendance/verification.ts`: lokasi vs radius, akurasi GPS, root/mock detection, selfie wajib.
- Teks UI bahasa Indonesia; identifier, path, komentar bahasa Inggris.
- `ios/` dan `android/` = Continuous Native Generation — **jangan edit manual**; konfigurasikan di `app.json` + config plugin.
- Tambahan library native → development build: `npx expo run:android` / `eas build --profile development`.

---

## 3. Konvensi BE

**Sumber kebenaran:** `BE/CLAUDE.md` (Wajib dibaca tiap mau kerja di backend).

- Go 1.26 · huma v2 + chi · pgx/v5 · goose v3 · MinIO
- Semua route di bawah `/hcm/v1`
- ESS selalu lewat `employees.user_id` dari token JWT, tidak dari id URL
- Error ke klien: `refusal.*` (pesan Indonesia)
- Worker: `HCM_ENV=dev go run ./cmd/worker`

---

## 4. Menjalankan

### Backend (BE/)
```bash
cd BE
# 1. MinIO via Docker (wajib untuk provision & seed)
docker compose up -d minio

# 2. Provision + seed (sekali saja)
go run ./cmd/migrate -meta
go run ./cmd/provision -slug demo -name "BPR Demo"
go run ./cmd/seed -tenant demo

# 3. Jalankan API
make run        # atau: HCM_ENV=dev go run ./cmd/api
# API: http://localhost:8096/hcm/v1
# Docs: http://localhost:8096/hcm/v1/docs
```

**Login demo:** Kode BPR `demo`, username `pegawai1`, password `password123`.

### Frontend (FE/)
```bash
cd FE
npm ci                  # install deps
npm run start           # Expo dev server (port 8081)
npm run typecheck       # tsc --noEmit
npm run lint            # eslint
npx jest                # unit tests (153 test, 15 suite)
```

### Expo (HP nyata)
```bash
cd FE
# Development build (HP)
npx expo run:android    # Android
npx expo run:ios        # iOS

# OTA update
npx eas update --branch preview
```

---

## 5. API Endpoints Mobile (/hcm/v1)

Rute ESS (pegawai) prefix `/me/`:

| Metode | Path | Fungsi |
|---|---|---|
| POST | /auth/login | Login |
| POST | /auth/change-password | Ganti password |
| GET | /me/employee | Profil sendiri |
| GET | /me/attendance/today | Absen hari ini |
| GET | /me/attendance/mobile-config | Konfigurasi mobile (jam buka, lokasi, radius) |
| POST | /me/attendance/punches | Absen (GPS + selfie) |
| GET | /me/attendance | Riwayat absen |
| GET | /me/attendance/assignments | Penugasan WFH / tugas luar |
| GET | /me/attendance/events | Acara dengan QR |
| POST | /me/attendance/events/validate | Validasi QR acara |
| POST | /me/attendance/wfh-requests | Ajukan WFH |
| GET | /me/attendance/corrections | Riwayat koreksi |
| GET | /me/leave/balances | Saldo cuti |
| GET | /me/leave/requests | Riwayat pengajuan cuti |
| POST | /me/leave/requests | Ajukan cuti/izin |
| GET | /leave/types | Jenis cuti |
| GET | /leave/portions | Opsi sebagian hari |
| GET | /me/leave/preview | Preview hari kerja |
| GET | /me/overtime | Riwayat lembur |
| POST | /me/overtime | Ajukan lembur |
| GET | /me/payslips | Daftar slip gaji |
| GET | /me/payslips/{id} | Detail slip gaji |
| GET | /me/loans | Riwayat kasbon |
| POST | /me/loans | Ajukan kasbon |
| GET | /me/claim-types | Jenis reimbursement |
| GET | /me/claims | Riwayat reimbursement |
| POST | /me/claims | Ajukan reimbursement |
| POST | /me/claims/receipts | Unggah bukti |
| GET | /me/appraisals | Riwayat penilaian |
| GET | /me/training | Data pelatihan |
| GET | /me/discipline | Catatan perilaku |
| GET | /me/documents | Dokumen pribadi |
| POST | /me/documents/{id}/download | Link unduh dokumen |
| GET | /me/announcements | Pengumuman |
| GET | /me/notifications | Notifikasi |
| POST | /me/notifications/read | Tandai baca |
| POST | /me/push-tokens | Register push token |
| GET | /approval-requests?view=mine | Kotak masuk |
| POST | /approval-requests/{id}/approve | Setujui |
| POST | /approval-requests/{id}/reject | Tolak |

---

## 6. Selesai Berarti (FE)

```bash
cd FE && npm run typecheck && npx jest && npx eslint .
```

- TSC hijau, 153 test hijau, ESLint hijau — baru tugas selesai.
- Test untuk: `utils/date.ts`, `utils/geo.ts`, `features/attendance/verification.ts`, `features/requests/validation.ts`.

---

## 7. Git Workflow

- **Jangan push ke gitea** — hanya owner yang push.
- Commit lokal biasa, push ke remote owner nanti.

---

## 8. Referensi

[[DOTS HCM Project Index]] · [[Architecture]] · [[PRD]] · [[Backend Arsitektur]] · [[Frontend Arsitektur]] · [[Backlog]] · [[Rules]]
