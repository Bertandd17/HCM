# Backend Arsitektur — dots-hcm

**Versi Dokumen:** 1.0 · **Tanggal:** 2026-11-02

Rujukan: [[Architecture]] · [[Rules]] · [[Schema]] · [[DOTS HCM Project Index]]
**Sumber kebenaran:** `dots-hcm/CLAUDE.md` — baca dulu setiap mau kerja di backend.

---

## 1. Bentuk

Go 1.26 · huma v2 + chi · pgx/v5 · goose v3 · sqlc · River jobs.
Modul Go `github.com/DotsCore/dots-hcm`, `replace` ke `../dots-platform`.

```
cmd/{api,migrate,provision,seed,worker}/   binary
migrations/                                SEMUA migrasi goose tenant dalam satu
                                           rantai bernomor (di-embed, migrations.Tenant);
                                           migrations/meta/ = rantai katalog tenant
internal/kernel/                           registry, httpx, boot (config/tenancy/migrasi/
                                           testdb), audit, dbtx, listing, refusal, numbering,
                                           pii, clock, command (kontrak maker-checker),
                                           workflow (kontrak rantai persetujuan),
                                           objstore (MinIO)
internal/common/                           phone, jsonmap, code, ptr
internal/<modul>/                          domain → usecase → repository/postgres →
                                           service/http → adapter
internal/wiring/                           merakit modul dengan dependensi nyata;
                                           wiringtest = harness HTTP untuk test
internal/schema/                           daftar modul tanpa dependensi (permission, bentuk route)
internal/archtest/                         test arsitektur (larangan import, route tanpa auth)
```

---

## 2. Modul

| Modul | Tanggung jawab |
|---|---|
| `identity` | Login, sesi, peran, cakupan unit |
| `reference` | Daftar kode, kolom custom |
| `org` | Unit, grade, jabatan |
| `employee` | Data pegawai, kontrak, keluarga, pendidikan, karier, dokumen |
| `command` | Antrean persetujuan maker-checker |
| `workflow` | Persetujuan berjenjang + delegasi |
| `document` | Dokumen digital (MinIO, presigned URL) |
| `leave` | Cuti/izin/lembur, ledger, ESS |
| `attendance` | Jadwal, punch, impor fingerprint, hari, tutup periode |
| `performance` | Periode, template KPI, penilaian, KPI unit |
| `training` | Kompetensi, katalog/sesi pelatihan, sertifikasi, kewajiban + monitor kepatuhan |
| `payroll` | Opsional: komponen, tarif statutori, struktur gaji, run + slip |
| `report` | Runner dataset, ekspor xlsx, dashboard |

Migrasi: platform `0001`, identity `0010`, reference `0020`, org `0030`,
employee `0040`. Versi tidak boleh dipakai dua modul (boot menolak).
Repository `employee` ditulis tangan dengan pgx (daftar dinamis + cakupan);
modul lain memakai sqlc.

---

## 3. Lapisan

| Lapisan | Isi | Larangan |
|---|---|---|
| `domain/` | Tipe murni, aturan kalender (`lastDayOf`, `today()`) | Tanpa import huma/pgx |
| `usecase/` | Aturan bisnis, orkestrasi transaksi, `refusal.Fields` untuk validasi spesifik | Tidak parsing HTTP |
| `repository/postgres/` | SQL sqlc/pgx, scan, mapping error constraint | Tidak menyimpan status turunan |
| `service/http/` | `httpx.Register`, DTO huma, mapping ke HTTP | Tidak `switch` atas `pgx.ErrNoRows` |
| `adapter/` | Jembatan antar modul (satu-satunya tempat boleh import usecase modul lain) | — |

`internal/wiring/` = composition root — satu-satunya tempat rakit modul.
`archtest` menolak import `internal/<modul lain>/usecase` di luar `adapter/`.

---

## 4. Rute HTTP

Semua di bawah `/hcm/v1`, docs di `/hcm/v1/docs`. Route wajib lewat
`httpx.Register` (permission) / `httpx.Authenticated` / `httpx.Public`.

Rute tiap modul lihat README modul di `internal/<modul>/README.md` dan
`dots-hcm/CLAUDE.md`. Contoh penting:

| Modul | Rute inti |
|---|---|
| identity | `POST /auth/login` · `POST /auth/refresh` · `GET/PUT /me` · `/users`, `/roles` |
| employee | `/employees`, `/employees/{id}` + sub-resource (contracts, families, educations, career, documents, bank-account, status) |
| leave | `/leave/types`, `/leave/requests`, `/leave/ledger`, `/overtime/requests`, `/holidays` |
| attendance | `/attendance/shifts`, `/schedules`, `/punches`, `/days`, `/imports`, `/periods`, `/corrections`, `/mobile/*` (absen HP) |
| workflow | `/inbox`, `/approvals` |
| command | `/commands`, kebijakan per jenis |
| performance | `/performance/periods`, `/templates`, `/unit-kpis`, `/appraisals` |
| training | `/training/competencies`, `/courses`, `/sessions`, `/certifications`, `/obligations`, `/compliance` |
| payroll | `/payroll/settings`, `/components`, `/rates`, `/structures`, `/runs`, `/payslips` (mati bawaan) |
| report | `/reports/datasets/{key}`, `/reports/tiles` |

---

## 5. Aturan Bisnis — di mana kodenya

| Area | Aturan | Berkas |
|---|---|---|
| Kontrak | Batas akhir = `lastDayOf` (potong ke akhir bulan, anti `AddDate` normalisasi) | `employee/domain/contract.go` |
| Cuti | Hari kerja = Sen–Jat dikurangi `holidays`; jumlah hari disimpan saat pengajuan | `leave/usecase` |
| Cuti | GRANT/EXPIRE = tahun hak; TAKE/REVERSE/ADJUST = tahun dibukukan | `leave_ledger` |
| Lembur | Menit diakui = min(pengajuan, punch) | `attendance/usecase` |
| Absensi | `punched_at` timestamptz; hari = tanggal WIB; rentang pakai `($d::date)::timestamp AT TIME ZONE 'Asia/Jakarta'` | `attendance/repository` |
| Kinerja | Total bobot KPI = 100 (deferred constraint — error saat commit) | `performance` |
| Kepatuhan pelatihan | Dihitung saat dibaca, bukan tabel hasil | `training/repository/postgres/compliance.go` |
| Payroll | Tarif tidak pernah hard-coded; `statutory_rates` dengan `effective_from` | `payroll/usecase` |
| Payroll | Termination: headcount d = `hire_date <= d AND (termination_date IS NULL OR termination_date > d)` | `report` |

Semua fungsi aturan murni, tanpa side effect.

---

## 6. Ports & Adapters

Modul A tidak pernah mengimpor modul B langsung. Jembatan:

| Adapter | Mengadaptasi | Ke |
|---|---|---|
| `payroll/adapter.Attendance` | `attendance_days` | `payroll` (hari belum dihitung job ≠ hadir; hitung ulang dulu) |
| `workflow.Owner{Kind, Decided}` | modul pemilik pengajuan | `kernel/workflow` (di `wiring`) |
| `command.Handler` | modul pemilik data master | `kernel/command` |
| `reference.UC.ValidateCustomData` | modul pemilik entitas extensible | `reference` |

---

## 7. Infrastruktur Bersama (`internal/kernel`)

| Komponen | Isi |
|---|---|
| `boot` | Config (`HCM_ENV` dev/prod), tenancy, migrasi otomatis, testdb |
| `dbtx` | `Runner.Run` — satu transaksi + audit |
| `audit` | `audit.Write` append-only |
| `httpx` | Register/Authenticated/Public, error mapper, `refusal.*` |
| `listing` | Paginasi + filter standar |
| `refusal` | Pesan error Indonesia |
| `numbering` | Nomor dokumen (NIP, nomor surat) |
| `pii` | Enkripsi NIK/tanggal lahir (`nik_enc` + `nik_hash`) |
| `clock` | `Today()` = tengah malam WIB (bukan UTC) |
| `command` | Kontrak maker-checker |
| `workflow` | Kontrak rantai persetujuan |
| `objstore` | MinIO, bucket per tenant `dots-hcm-<slug>` |

---

## 8. Error Mapping

| Bentuk | HTTP |
|---|---|
| `refusal.*` | Sesuai jenis (400/404/409/422) |
| Optimistic lock (`version` beda) | 409 "Data sudah diubah pengguna lain, muat ulang" |
| Di luar cakupan unit | 404 yang sama dengan "tidak ada" |
| Error lain | 500 generik, dicatat log |
| Validasi huma | Diterjemahkan generik oleh `httpx/errors.go`; untuk pesan spesifik kembalikan `refusal.Fields` dari use case |

---

## 9. Referensi

[[DOTS HCM Project Index]] · [[Architecture]] · [[Schema]] · [[Rules]] · [[Backlog]] · [[Frontend Arsitektur]] · [[Mobile Arsitektur]] · [[Cara Menjalankan]]
