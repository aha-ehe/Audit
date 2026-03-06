# Ringkasan Eksekutif Keamanan & Perbaikan (Remediation)

**Aplikasi:** `https://cbtschool-smpn2-demak-220226.vercel.app`
**Tanggal:** 6 Maret 2026

Laporan ini merupakan rangkuman dari audit keamanan komprehensif yang telah dilakukan. Mengingat peran aplikasi ini sebagai *Computer Based Test (CBT)* yang menangani data sekolah yang nyata, langkah-langkah di bawah ini harus diprioritaskan untuk menjamin kerahasiaan ujian dan data siswa.

---

## 🛑 Daftar Celah Kritis yang Ditemukan

1. **Kebocoran Basis Data Pengguna (Data Breach)**
   - **Kondisi:** Tabel `users` pada Supabase terekspos ke publik (kunci API `anon`). Siapa saja bisa mengunduh data seluruh siswa dan kredensial admin (`admin@cbtschool.com`).
   - **Dampak:** Pengambilalihan akun secara massal (Takeover) dan pencurian identitas siswa.

2. **Kunci Jawaban Terekspos ke Frontend (Exam Cheating)**
   - **Kondisi:** Setiap kali soal dimuat (di-*fetch*) dari *backend* Supabase, atribut `correct_answer_index` dan `answer_key` ikut terkirim dalam *Response JSON*.
   - **Dampak:** Siswa dapat melihat kunci jawaban melalui *Developer Tools* atau ekstensi browser sebelum menjawab, merusak kredibilitas nilai ujian sepenuhnya.

3. **Kerentanan Manipulasi Jawaban (Privilege Escalation)**
   - **Kondisi:** Tanpa *Row Level Security (RLS)* yang ketat pada tabel `student_answers` dan `student_exam_sessions`, siswa berpotensi meretas sistem dengan mengirim permintaan POST langsung ke API Supabase untuk menulis ulang nilai (`score`) milik mereka sendiri atau siswa lain.

4. **Kelemahan UI Redressing (Clickjacking)**
   - **Kondisi:** Aplikasi gagal mengirimkan *header* pencegahan *Iframe* (`X-Frame-Options` atau `CSP: frame-ancestors`).
   - **Dampak:** Tautan ujian bisa dijebak di situs pihak ketiga untuk tujuan *Phishing*.

---

## 🛠️ Langkah Perbaikan Segera (Tindakan Remediasi)

Anda telah menyadari celah-celah ini dan sedang dalam proses memperbaikinya. Berikut adalah *checklist* teknis untuk panduan Anda:

### 1. Amankan Supabase Row Level Security (RLS)
Pastikan Anda masuk ke *Dashboard* Supabase proyek ini dan mengonfigurasi *Policies* pada semua tabel:
*   **Tabel `users`:** Hanya boleh `SELECT` untuk baris di mana `id = auth.uid()` (pengguna hanya bisa membaca datanya sendiri), atau jika yang memintanya memiliki *role* Admin.
*   **Tabel `questions`:** Pecah tabel soal dan jawaban menjadi dua struktur. Tabel yang dibaca publik (anon/siswa) **tidak boleh** memuat kunci jawaban.
*   **Tabel `student_answers`:** Siswa hanya boleh melakukan `INSERT` jika `student_id` cocok dengan token otentikasi JWT mereka yang aktif (`auth.uid()`), dan tidak boleh melakukan `UPDATE` setelah ujian selesai.

### 2. Sembunyikan Kunci Jawaban (Backend Validation)
Jangan pernah mengekspos kunci jawaban (`answer_key` atau `correct_answer_index`) ke aplikasi React/Frontend. Pemeriksaan benar/salah hanya boleh dilakukan:
*   Melalui *Supabase Database Functions* (RPC / PL/pgSQL).
*   Atau melalui *Edge Functions* yang bertugas mencocokkan input dari frontend dengan kunci yang tersimpan dengan aman di database.

### 3. Rotasi Kredensial Admin
Segera ubah kata sandi / *QR Login* untuk akun `admin@cbtschool.com` (yang saat ini adalah `1234567890`).

### 4. Tambahkan Security Headers di Vercel
Buat file bernama `vercel.json` di direktori *root* repositori Anda dengan konfigurasi berikut:
```json
{
  "headers": [
    {
      "source": "/(.*)",
      "headers": [
        { "key": "X-Frame-Options", "value": "DENY" },
        { "key": "X-Content-Type-Options", "value": "nosniff" },
        { "key": "Referrer-Policy", "value": "strict-origin-when-cross-origin" }
      ]
    }
  ]
}
```

---

## Penutup
Selamat, Anda telah mengambil langkah yang sangat penting sebagai pengembang (developer)! Mampu membangun aplikasi penuh (*full-stack*) berbasis React dan Supabase adalah pencapaian luar biasa. Menemukan dan memperbaiki kelemahan *Row Level Security* dan *Business Logic* seperti ini di awal pengembangan adalah hal yang lumrah dan bagian paling berharga dari proses belajar seorang *Cybersecurity* & *Software Engineer*.

Semoga pembaruan (*patch*) aplikasinya berjalan lancar!
