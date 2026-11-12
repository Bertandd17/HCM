# PRD.md — Ringkasan Produk
# DOTS HCM

**Versi Dokumen:** 1.0
**Tanggal:** 2026-11-02
**Spesifikasi asli:** `dots-hcm/requirement.txt` + brosur DOTS-HCM, dirangkum di `docs/superpowers/specs/2026-10-28-dots-hcm-design.md`
**Pemilik produk:** PT Dimensi Kreasi Nusantara
**Klien:** BPR (Bank Perekonomian Rakyat)

> Dokumen ini meringkas desain sistem menjadi bentuk yang bisa dipakai sebagai
> rujukan kerja. **Dokumen desain asli tetap kontrak**; rincian teknis di
> [[Architecture]] dan [[Backend Arsitektur]].

---

## 1. Masalah yang Diselesaikan

BPR butuh pengelolaan SDM yang tertib, terdokumentasi, patuh regulasi, dan mudah
dimonitor Direksi serta Divisi Kepatuhan. Proses manual (absensi, cuti,
penilaian, pelatihan, pelaporan) menimbulkan data tidak sinkron, laporan
terlambat, dan audit yang sulit.

Ukuran keberhasilan: setiap pertanyaan auditor/OJK tentang pegawai — "siapa
pejabat X pada tanggal Y, apa SK-nya, apakah sertifikasinya masih berlaku" —
bisa dijawab dari sistem lengkap dengan buktinya.

---

## 2. Tujuan, Non-tujuan, Anti-tujuan

**Tujuan (6 modul requirement):**

| # | Modul | Isi |
|---|---|---|
| 1 | Data pegawai | Terpusat, riwayat karier/mutasi/promosi/kontrak, dokumen digital |
| 2 | Absensi | Harian (manual + mesin fingerprint), cuti/izin/lembur dengan approval berjenjang, rekap otomatis |
| 3 | Kinerja | KPI pegawai dan unit, penilaian bulanan/tahunan, catatan pengembangan |
| 4 | Pelatihan | Data pelatihan, mapping kompetensi per jabatan, sertifikasi, monitoring kewajiban |
| 5 | Payroll & benefit (opsional) | Gaji dari absensi dan jabatan, tunjangan/potongan/insentif, PPh 21, BPJS, slip digital, file transfer bank |
| 6 | Dashboard & laporan | Headcount, turnover, absensi, kinerja untuk Direksi |

**Non-tujuan (untuk saat ini):** rekrutmen/ATS, suksesi, posting jurnal ke GL CBS
(payroll cukup ekspor jurnal). Absen GPS/QR tadinya fase lanjutan, sudah dibangun
di aplikasi HP.

**Anti-tujuan (sistem dianggap gagal walaupun "jalan"):**
- Riwayat bisa diubah diam-diam: jabatan, gaji, atau nilai yang sudah disetujui berubah tanpa jejak.
- Data satu BPR terlihat oleh BPR lain, atau data cabang lain terlihat oleh HR cabang.
- NIK, gaji, atau dokumen pribadi bisa dibaca orang yang tidak berhak.
- Approval cuti macet karena atasan kosong atau sedang cuti, tanpa jalur alternatif.
- Payroll dijalankan dua kali menghasilkan dua slip atau dua kali transfer.

---

## 3. Prinsip yang Mengikat

| # | Prinsip | Konsekuensi teknis |
|---|---|---|
| P1 | **Tanpa hard delete** | Data dinonaktifkan/diakhiri; tabel riwayat append-only; koreksi = baris baru |
| P2 | **Riwayat tidak bisa berubah** | Trigger blok UPDATE/DELETE/TRUNCATE pada tabel riwayat; audit_log append-only |
| P3 | **Custom ≠ wajib** | Kolom native untuk yang menentukan state/perhitungan/approval; custom (`custom_data` JSONB) hanya dicatat/dilaporkan |
| P4 | **Satu aturan, satu tempat** | Aturan bisnis hanya di Go (usecase); FE/HP mencerminkan, bukan menyaingi |
| P5 | **Maker-checker** | Perubahan data sensitif (gaji, rekening, status, struktur) wajib persetujuan; handler memeriksa ulang keadaan record |
| P6 | **Workflow berjenjang atasan** | Cuti/izin/lembur/koreksi lewat `workflow.Engine`, atasan di-resolve dari `reports_to` saat pengajuan |
| P7 | **Payroll mati secara bawaan** | Modul diaktifkan lewat setting `modules.payroll_enabled`; tanpa itu route payroll 403 |
| P8 | **Uang rupiah bulat, tanggal `date` WIB** | Nilai uang `BIGINT` rupiah; semua perbandingan "hari ini" pakai tengah malam WIB |

---

## 4. Lingkup Platform

### 4.1 Tiga kanal

| Kanal | Repo | Pengguna |
|---|---|---|
| **Backend** | `dots-hcm` | Satu API melayani web + mobile |
| **Web** | `dots-hcm-ui` | HR, atasan, pegawai — antarmuka penuh |
| **Mobile** | `dots-hcm-mobile` | Pegawai — self-service: absen (kantor/WFH/tugas luar/acara QR), pengajuan, lihat profil/jadwal/slip |

### 4.2 Cakupan fitur

Inventaris lengkap per kanal (Ada / Sebagian / Belum) ada di [[Backlog]],
disalin dari `BACKLOG-DOTS-HCM.xlsx`. Per 2 Nov 2026:

- Backend: **53 fitur Ada, 1 Sebagian, 5 Belum**
- Web: **55 Ada, 2 Sebagian, 2 Belum**
- Mobile: **15 Ada, 9 Sebagian, 35 Belum** — mobile masih fokus absen & self-service

---

## 5. Aturan Domain Utama

Ringkas; implementasi lengkap di [[Architecture]] dan [[Backend Arsitektur]].

| Area | Aturan |
|---|---|
| Status pegawai | Hanya lewat transisi eksplisit (HIRE → MUTATION → PROMOTION → … → TERMINATION) dengan maker-checker; `termination_date` = hari pertama tidak bekerja |
| Cuti | Saldo = jumlah entri `leave_ledger` (GRANT/TAKE/REVERSE/EXPIRE/ADJUST); potong saat APPROVED; kembalikan lewat REVERSE saat batal |
| Lembur | Menit diakui = min(pengajuan, hasil punch) |
| Absensi | `attendance_days` hasil olahan job; tutup periode = final, tidak bisa dibuka ulang |
| Kinerja | Total bobot KPI = 100 (deferred constraint); setelah FINAL tidak bisa diubah, koreksi = revisi baru |
| Payroll | Run: DRAFT → CALCULATED → APPROVED → PAID → LOCKED; rekap 1721-A1 bukan bukti potong resmi; THR tidak bisa ditahan |
| Kepatuhan | Dihitung saat dibaca, bukan tabel hasil; sertifikasi kedaluwarsa 90 hari diperingatkan |

---

## 6. Pengguna & Akses

Tiga peran sistem: `ADMIN`, `EMPLOYEE`, dan peran kustom bank. Permission berbasis
enum di kode (`employee.employee.read`, `leave.request.approve`, …), di-seed ke RBAC
saat boot.

- **HR/Atasan** — web penuh.
- **Pegawai** — web `(app)/me/*` dan aplikasi HP. ESS selalu lewat `employees.user_id` dari aktor, tidak pernah dari id di URL.
- **Cakupan unit** — HR cabang hanya melihat data di unit dan sub-unitnya; di luar cakupan = 404 yang sama dengan "tidak ada".

Login: Kode BPR + username/email + password. Argon2id hash, JWT HS256 15 menit +
refresh rotasi 7 hari di `user_sessions`. Reset password oleh admin mencabut semua sesi.

---

## 7. Data Awal (Seeder)

Tiga profil seeder (`make seed`, `seed-uat`, `seed-prod`):

| Profil | Isi |
|---|---|
| **demo** | Data minimum untuk uji lokal; password tetap: `admin`/`AdminDemo2026!`, lainnya `DemoHcm2026!` |
| **UAT** | Sekitar 100 pegawai, absensi 100 hari semua kondisi (telat, alpa, lembur, cuti) |
| **go-live** | Data nyata + laporan kesiapan |

Lihat `dots-hcm/docs/SEEDER.md`.

---

## 8. Yang Harus Dijawab Sebelum Go-Live

1. Siapa klien pertama, berapa jumlah pegawai dan cabang (menentukan performa & cakupan unit)?
2. Merek dan model mesin fingerprint yang dipakai.
3. Apakah klien pertama butuh payroll di HCM, atau payroll tetap di sistem lain?
4. Format penilaian kinerja yang berlaku (skala dan predikat).

---

## 9. Referensi

[[DOTS HCM Project Index]] · [[Architecture]] · [[Backlog]] · [[Roadmap]] · [[Known Debt]] · [[Decision Log]] · [[Backend Arsitektur]] · [[Frontend Arsitektur]] · [[Mobile Arsitektur]]
