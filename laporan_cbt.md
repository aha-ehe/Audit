# Laporan Audit Keamanan Website CBT Sekolah

**Target:** `https://cbtschool-smpn2-demak-220226.vercel.app`
**Tanggal:** 6 Maret 2026

## Ringkasan Eksekutif
Berdasarkan hasil analisis, aplikasi *Computer Based Test (CBT)* ini adalah *Single Page Application* (React/Vite) yang terhubung langsung secara *Serverless* ke database Supabase. Terdapat sejumlah kelemahan sangat kritis (KRITIS) pada konfigurasi basis data (*Supabase Row Level Security*) yang mengakibatkan seluruh sistem CBT ini dapat diakali/dicurangi (*Cheat*) oleh siswa biasa maupun peretas luar.

---

## Temuan Utama

### 1. Kebocoran Kunci Jawaban (Cheat Vulnerability - KRITIS)
*   **Analisis Eksploitasi:**
    Dalam aplikasi berbasis ujian/CBT, aturan emasnya adalah *kunci jawaban tidak boleh dikirim ke perangkat peserta (browser) sampai ujian selesai atau ditutup*.
    Namun, pada saat siswa membuka halaman soal, API *backend* Supabase mengirimkan struktur data yang utuh. Contoh bocoran dari tabel `questions`:
    ```json
    {
      "question": "Ayat Q.S. At- Taubah /9: 122 mengajarkan...",
      "options": ["A", "B", "C", "D"],
      "correct_answer_index": 2,
      "answer_key": {"index": 2}
    }
    ```
*   **Dampak:** Siswa yang memiliki sedikit pengetahuan teknis (*Developer Tools / Network Tab*) atau yang menginstal ekstensi curang khusus di browsernya dapat melihat semua jawaban yang benar sebelum memilih dan menekan *Submit*, memastikan nilai mereka selalu 100 secara otomatis.
*   **Rekomendasi:** Hapus (jangan tampilkan di *select statement* API) kolom `correct_answer_index` dan `answer_key` untuk peran (*role*) `anon` atau `authenticated student`. Validasi benar/salah harus murni terjadi di sisi server setelah peserta mengirim (`POST`) jawaban mereka.

### 2. Bypass Autentikasi dan Pencurian Data Siswa/Admin (Auth Bypass - KRITIS)
*   **Analisis Eksploitasi:**
    Konfigurasi keamanan level baris (Row Level Security / RLS) pada proyek Supabase (`ytlizvulzbnubvdlhtpz`) saat ini gagal diaktifkan dengan benar atau dibuat terlalu melonggar (Permissive).
    Memanfaatkan kunci publik `anon` API, kami melakukan panggilan REST API sederhana dan berhasil membaca isi seluruh tabel `users`.
*   **Data yang Bocor (Contoh):**
    ```json
    [
      {
        "username": "15735@smpn2demak.sch.id",
        "full_name": "ALVITO RASENDRIYA ADHA KURNIAWAN",
        "qr_login_password": "15735"
      },
      {
        "username": "admin@cbtschool.com",
        "full_name": "Administrator Utama",
        "qr_login_password": "[REDACTED]"
      }
    ]
    ```
*   **Dampak:**
    1. Siapapun dari internet bisa mengunduh basis data siswa lengkap (NISN, Nama, dll).
    2. Siapapun bisa mencuri kata sandi (*qr_login_password*) dan *login* mengambil alih akun siswa lain atau bahkan *Administrator Utama*.
*   **Rekomendasi:** Wajib mengaktifkan *Row Level Security (RLS)* di semua tabel Supabase. Tabel `users` hanya boleh dibaca dan ditulis oleh Administrator atau hanya baris data milik *user* itu sendiri saja yang boleh dibaca oleh siswa yang sedang login. Blokir akses baca untuk *role anon*.

### 3. Clickjacking (UI Redressing - SEDANG)
*   **Analisis Eksploitasi:**
    Website memuat Header `Access-Control-Allow-Origin: *` dan absennya perlindungan anti-*iframe* seperti `X-Frame-Options` atau `Content-Security-Policy: frame-ancestors`.
*   **Dampak:** Halaman ujian dapat disematkan di situs jahat lain untuk menipu pengguna awam.
*   **Rekomendasi:** Terapkan Header *X-Frame-Options: SAMEORIGIN* di `vercel.json`.

---

## Kesimpulan Akhir
Dalam wujudnya saat ini, aplikasi `cbtschool-smpn2-demak` **belum layak pakai (Not Ready for Production) untuk ujian sekolah yang sah**. Konfigurasi Supabase RLS yang terbuka dan tereksposnya *Kunci Jawaban* di API merupakan kelemahan bawaan arsitektur (*design flaw*) yang menggugurkan keabsahan nilai hasil ujian siswa. Pengembang wajib merevisi seluruh kebijakan RLS Supabase sebelum melaksanakan ujian sungguhan.
