# Frontend Arsitektur — dots-hcm-ui

**Versi Dokumen:** 1.0 · **Tanggal:** 2026-11-02

Rujukan: [[Architecture]] · [[Rules]] · [[DOTS HCM Project Index]]
**Sumber kebenaran:** `dots-hcm-ui/CLAUDE.md` — baca `../dots-hcm/CLAUDE.md` juga.

---

## 1. Bentuk

Next.js 15 App Router · React 19 · Tailwind 4 · TanStack Query 5 · pnpm.
Design system `@dots/ui` + `@dots/utils` dari `../dots-platform-ui`
(`link:../../dots-platform-ui/packages/*`). `@dots/auth` **tidak** dipakai —
sesi dikelola sendiri di `src/shared/lib/session.ts`.

```
src/
├── app/                    route (App Router)
│   ├── login/              login: Kode BPR + username/email + password
│   ├── change-password/
│   └── (app)/              semua halaman dalam shell utama
│       ├── dashboard/      tile ringkasan
│       ├── employees/      record adalah menu: detail + aksi per pegawai
│       ├── attendance/     shifts, schedules, days, imports, periods, corrections
│       ├── leave/          types, requests, overtime, holidays
│       ├── performance/    periods, templates, unit-kpis, bands, appraisals
│       ├── training/       competencies, courses, sessions, obligations, compliance
│       ├── payroll/        settings, components, rates, salary-structures, runs, payslips
│       ├── reports/        datasets/{key}, per-area (attendance, leave, payroll, …)
│       ├── inbox/[id]/     kotak masuk persetujuan (workflow)
│       ├── me/             ESS pegawai: leave/new, overtime/new, payslips, appraisals, delegations
│       ├── commands/       antrean maker-checker
│       └── settings/       users, roles, code-lists, custom-fields, approval-policies, workflow-policies
├── features/<module>/      logika fitur: api client, hooks TanStack Query, komponen feature
└── shared/
    ├── lib/                api.ts, session.ts, nav.ts
    └── components/         komponen lokal yang BELUM ada di @dots/ui
```

**Prinsip IA CBS: record adalah menu.** Aksi seperti mutasi, promosi,
perpanjang kontrak, nonaktifkan muncul di halaman pegawai, bukan di sidebar.

---

## 2. Konvensi

- **Teks UI bahasa Indonesia; identifier, path, komentar bahasa Inggris.**
- Hanya komponen `@dots/ui`. Komponen lokal di `src/shared/components` hanya untuk yang belum ada di sana. `@dots/ui` di dots-platform-ui **tidak punya** Toaster, Checkbox, NumberInput — pakai `ToasterProvider`/`useToast` dan `Toggle` lokal.
- Unduhan file lewat `hcmDownload`, bukan `hcm` (respons file huma = struct output dengan `Body []byte` + header Content-Type/Content-Disposition).
- `registry.Column/Filter` ber-tag json snake_case — `src/features/reports/datasets.ts` bergantung pada nama itu.
- `NEXT_PUBLIC_API_BASE_URL` default `http://localhost:8096`.

---

## 3. Selesai berarti

```bash
pnpm vitest run && pnpm typecheck && pnpm lint && pnpm build
```

Empat-empatnya hijau sebelum tugas dinyatakan selesai.

---

## 4. Referensi

[[DOTS HCM Project Index]] · [[Architecture]] · [[Rules]] · [[Backlog]] · [[Backend Arsitektur]] · [[Mobile Arsitektur]]
