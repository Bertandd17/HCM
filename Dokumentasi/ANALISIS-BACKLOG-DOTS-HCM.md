# Analisis Backlog DOTS HCM — Verifikasi terhadap Kode

Tanggal: 1 Nov 2026 (tanggal sumber backlog). Verifikasi: `BACKLOG-DOTS-HCM.xlsx` (49 item) dan `BACKLOG-MOBILE-HRD-KEPALA-CABANG.xlsx` dibandingkan dengan isi repositori `dots-hcm-mobile` (BE Go, FE Expo/RN).

Metode: setiap klaim yang bisa diperiksa lewat kode dicek langsung (grep/inspeksi berkas). Item yang tidak bisa diverifikasi dari kode (UAT manual, perangkat asli, kebijakan) ditandai demikian.

---

- 49 item di backlog; **11 item klaimnya sudah tidak akurat** dengan kode saat ini (fitur ternyata sudah ada atau sudah jauh lebih lengkap), termasuk 2 dari 6 kategori "Cek dulu".
- **8 item terkonfirmasi masih benar** belum ada.
- Kategori "Cek dulu" yang tersisa (B-035, B-041, B-042) butuh pemeriksaan layar/perangkat; B-004 dan B-013 sudah terjawab dari kode (lihat §2).
- Workbook kedua (HRD/Kepala Cabang) akurat: temuan utamanya — FE belum membatasi menu berdasar permission — terkonfirmasi dari kode FE.
- Rekomendasi: perbarui status 11 item di sheet "Belum Dibuat" sebelum backlog dipakai untuk prioritisasi, agar tim tidak mengerjakan ulang fitur yang sudah ada.

---

## 2. Item yang klaimnya sudah tidak akurat (verifikasi kode)

| ID | Klaim backlog | Temuan di kode | Bukti |
|----|---------------|----------------|-------|
| B-004 | Push langsung dari mesin absen (ADMS) perlu dicek | **Sudah ada.** Server push ADMS ZKTeco di `/iclock/cdata` (handshake + ATTLOG), idempoten, serial harus terdaftar, resolusi tenant per serial; seed demo punya mesin ADMS | `BE/internal/attendance/service/http/adms.go:22-77`, `usecase/devices.go:171-177`, `wiring/adms.go:92-104`, `cmd/seed/features.go` |
| B-013 | Format berkas bank per bank belum ada (baru CSV generik) | **Sudah ada.** `SaveBankFormat` dengan kode/format per bank (uji: "BCA KlikBCA Bisnis", pemisah, kolom, ekstensi); unduhan bank file diaudit `bank_file_downloaded` dengan checksum SHA-256 | `BE/internal/payroll/usecase/outputs_test.go:161-163`, `outputs.go:138-139`, README payroll |
| B-017 | UI belum memberi tahu pengguna sudah menyetujui dan menunggu penyetuju lain | **Sudah ada.** Layar approval command menyembunyikan tombol (`canAct = PENDING && !i_approved`) dan menampilkan pesan "Anda sudah menyetujui pengajuan ini… menunggu persetujuan dari penyetuju lain (x dari y)" | `FE/app/approvals/command/[id].tsx:51-76` |
| B-018 | Eskalasi otomatis alur persetujuan cuti/izin/lembur belum ada; hanya ambil alih manual HR | **Sudah ada.** Job `workflow.escalate` per jam + `escalate_after_hours` di `approval_policies` + notifikasi `APPROVAL_ESCALATED` ke pemegang `workflow.hr.decide`; takeover HR dengan catatan wajib sudah ada | `BE/internal/workflow/module.go:37-38`, `repo/notify.go:53-72`, `usecase/escalate_test.go`, `uc_test.go:TestHRTakeoverNeedsNote` |
| B-026 | Layanan mandiri pelatihan: perlu cek cakupan | **Sudah cukup lengkap.** Katalog kursus, sesi terbuka, daftar/undur mandiri, evaluasi pasca-pelatihan dari HP | `FE/services/api/info.ts:163-176` |
| B-033 | Peringatan dokumen kedaluwarsa belum ada | **Sebagian sudah ada di HP.** Badge `EXPIRING` (30 hari) / `EXPIRED`, pengurutan dokumen bermasalah lebih dulu, saringan `needsAttention`. Yang belum: notifikasi/proses sisi server (dihitung saat baca, tanpa job) | `FE/features/documents/expiry.ts`, `__tests__/expiry.test.ts` |
| B-039 | Mode gelap: token ada, belum diaktifkan | **Sudah aktif.** Palet gelap lengkap, `ThemeToggle` di layar Pengaturan, ikut mode sistem bila belum dipilih, ada test tema | `FE/features/profile/ThemeContext.tsx`, `FE/components/ui/ThemeToggle.tsx`, `FE/constants/colors.ts:99`, `app/profile/settings.tsx:51` |
| B-044 | Pesan error ramah berbahasa Indonesia di HP masih bisa bocor pesan teknis | **Sudah ditangani.** `errorMessage`/`statusMessage`/`humanError` memetakan error teknis ke teks Indonesia; server refusal lolos apa adanya; ada test khusus yang menyebut B-044 | `FE/services/api/hcm.ts:334-392`, `__tests__/hcm.test.ts:86-89` |
| B-049 | Pratinjau PDF belum ada di HP (unduh langsung) | **Sudah ada.** Modal pdf-viewer dengan PDF.js (zoom, rotasi, navigasi halaman) di WebView | `FE/app/(modals)/pdf-viewer.tsx` |

Catatan B-007 (kolom custom cuti "belum diaktifkan"): README modul leave masih menulis belum diaktifkan, tetapi FE sudah memanggil `/me/leave/custom-fields` dan mengirim `custom_data` saat mengajukan. Kemungkinan README/backlog basi — cek konfigurasi tenant sebelum memutuskan.

---

## 3. Item terkonfirmasi masih belum ada

| ID | Item | Bukti |
|----|------|-------|
| B-005 | Cuti bersama tidak otomatis memotong saldo | `BE/internal/leave/README.md:20` |
| B-006 | Lampiran cuti hanya terlihat HR; atasan hanya ringkasan | `BE/internal/leave/README.md:21` |
| B-041 / B-042 | UAT tampilan 375/1440 px, uji perangkat asli | Belum pernah dilakukan (klaim proses, bukan kode) |
| B-045 / B-046 | Pembatalan WFH massal; tugas luar massal | Penugasan massal baru untuk WFH (sesuai backlog) |
| B-035 | Pemindaian virus aktif bawaan perlu dicek | **Sudah aktif bawaan.** `FileScanner` deteksi otomatis ClamAV (`hcmscan` di `127.0.0.1:3310`) sebelum fallback engine bawaan EICAR; file terdeteksi langsung dihapus | `BE/internal/document/README.md`, `BE/cmd/hcm/main.go` |
| B-047 | Logo & nama bank di header aplikasi | Tidak ditemukan di FE |

Item lain (B-001..B-003, B-010..B-012, B-015, B-016, B-019..B-025, B-027..B-032, B-034, B-036..B-038, B-043, B-048) tidak dibantah temuan kode; anggap valid sampai dicek saat mulai dikerjakan.

---

## 4. Workbook HRD / Kepala Cabang (Peta Fitur per Role)

Sheet "Peta Fitur per Role" akurat. Verifikasi kode:

- **Baris 23 — FE belum gate menu by permission: BENAR.** Tidak ada `hasPermission`/pembacaan permission di FE; FE hanya mengenal `roles` dari `/me/employee` (`FE/services/api/hcm.ts:16-18`). Matriks izin menunjukkan `HR_CABANG` sudah memegang banyak permission (termasuk `command.command.read/approve`, `workflow.hr.decide`, `attendance.punch.write`) sehingga satu kali implementasi gate membuka 6–10 fitur tanpa perubahan backend.
- **Baris 2 — Antrean command belum ada pintu masuk di menu: BENAR.** Layar `FE/app/approvals/command/` ada; tidak ada tautan dari tab/profile.
- **Baris 3 — Ambil alih HR belum ada tombol di HP: BENAR.** Backend takeover teruji (`TestHRTakeoverNeedsNote`); FE belum ada tombolnya.
- Seed demo `HR_CABANG` memang sudah punya `command.command.read`, `command.command.approve`, `workflow.hr.decide` (`BE/cmd/seed/roles.go:12-16`) — konsisten dengan Matriks Izin.

Implikasi: untuk target HRD + Kepala Cabang, kerja terbesar di FE (gate permission, pintu masuk menu, layar list pegawai/cuti/lembur), bukan backend.

---

## 5. Rekomendasi prioritas

1. **Perbarui sheet "Belum Dibuat"**: pindahkan 11 item §2 ke "Sudah ada"/turunkan cakupannya (B-033 sisa: notifikasi server; B-018 sisa: tombol Ambil Alih di HP). Catat sumber verifikasi (berkas + baris) di kolom "Catatan Tim".
2. **Gate menu by permission di FE** (Peta Fitur baris 23) — prasyarat rilis ke HR_CABANG; tanpa ini kepala cabang melihat menu yang 403-annya.
3. **Pintu masuk menu ke antrean command + tombol Ambil Alih** — dua pekerjaan FE kecil yang membuka fitur backend yang sudah selesai dan teruji.
4. Baru kerjakan item Pengembangan yang benar-benar belum ada, urut dampak: B-001 (shift lintas hari), B-009 (paksa tutup periode absen), B-034 (pembersih file yatim), B-040 (reaktivasi), B-045/B-046 (massal).

---

## 6. Sheet "Sengaja Tidak Dibuat"

Tidak ada yang perlu dibantah dari sisi kode. Catatan tunggal: butir 4 (THR tidak boleh ditahan karena SP) sesuai PP 36/2021 dan Permenaker 6/2016 — keputusan sah dan tidak bisa dipenuhi tanpa melanggar aturan; biarkan tertutup.
