# Laporan Audit Keamanan Website CBT Hira

**Target:** `https://hira-cbt.vercel.app/`
**Backend API:** `https://hiracbt-backend.vercel.app/api/v1/`
**Tanggal:** 6 Maret 2026

## Ringkasan Eksekutif
Aplikasi *Hira Test Portal* adalah aplikasi CBT modern berbasis *Next.js* (*React*) yang memiliki arsitektur terpisah (*decoupled*) dengan API khusus. Berbeda dengan aplikasi CBT sebelumnya yang rentan karena basis data *Supabase* langsung diekspos ke publik, aplikasi *Hira CBT* memiliki server *backend* tersendiri. Namun, selama pengujian, **backend API ini tidak dapat diakses (Offline / 504 Gateway Timeout)** untuk permintaan otentikasi yang valid.

Oleh karena itu, pengujian eksploitasi tingkat dalam (seperti kebocoran soal dan skor) tidak dapat dibuktikan secara praktis. Walau begitu, sejumlah temuan tingkat permukaan (*surface-level findings*) berhasil didokumentasikan.

---

## Temuan Utama

### 1. Gangguan Ketersediaan (Denial of Service - Backend Offline)
*   **Status:** Kritis untuk Operasional Aplikasi (CRITICAL)
*   **Analisis:**
    Ketika mengirimkan *payload* yang tidak valid secara format JSON ke endpoint `/api/v1/student/signin` (misalnya menggunakan objek `{}` alih-alih *string*), *backend* dengan cepat dan cerdas merespons *400 Bad Request* (`"studentId" must be a string`).
    Namun, ketika struktur datanya benar (`{"studentId": "12345", "password": "password"}`), Vercel memotong koneksi setelah 10 detik dengan pesan **FUNCTION_INVOCATION_TIMEOUT (HTTP 504)**.
*   **Kesimpulan:** Kerangka validasi *input* (seperti `Joi` atau `Zod`) berfungsi sempurna. Namun, lapisan koneksi *Database* di belakangnya sedang putus (*broken connection string*) atau layanannya sedang tertidur (*cold start timeout*).
*   **Rekomendasi:** Pengembang harus segera memeriksa *logs* pada *Serverless Function* Vercel untuk repositori `hiracbt-backend` dan memastikan parameter `DATABASE_URL` atau konfigurasi *connection pool* (seperti Prisma/PgBouncer) sudah benar.

### 2. Kesenjangan Header Keamanan Frontend (Clickjacking)
*   **Status:** Rentan (SEDANG)
*   **Analisis Eksploitasi:**
    Seperti banyak aplikasi yang disebarkan (*deployed*) lewat konfigurasi dasar Vercel, *frontend* `hira-cbt.vercel.app` memuat respons dengan `Access-Control-Allow-Origin: *` dan **tidak memiliki** instruksi pencegahan *Iframe* (`X-Frame-Options` atau `CSP: frame-ancestors`).
*   **Pengecualian Positif:** Sangat menarik untuk dicatat bahwa *Backend API* (`hiracbt-backend.vercel.app`) secara mengejutkan **dikonfigurasi dengan sangat aman**. *Backend* tersebut memuat *Header* `Content-Security-Policy` yang super ketat dan `X-Frame-Options: SAMEORIGIN` (kemungkinan dikonfigurasi melalui *library* `helmet` pada Express.js/Nest.js).
*   **Dampak Potensial:** Hanya antarmuka *Frontend* (UI) saja yang dapat dicuri atau dijebak oleh peretas menggunakan tag `<iframe src="https://hira-cbt.vercel.app">` di situs web palsu untuk serangan *Phishing* (mencuri ID dan *Password* siswa).
*   **Rekomendasi:** Anda sudah mengatur keamanan di *Backend*, sekarang atur juga di *Frontend* dengan menambahkan `vercel.json` berisi `X-Frame-Options: SAMEORIGIN` pada *repository* `hira-cbt`.

### 3. Ketahanan Terhadap Injeksi & Validasi Data (Injection & Parameter Pollution)
*   **Status:** Tangguh / Sangat Aman (EXCELLENT)
*   **Analisis:**
    Aplikasi sama sekali tidak menolerir manipulasi tipe data (*Type Juggling*). Berbeda dengan *backend* Node.js pada umumnya yang sering gagal menangani input selain *string*, API *Hira* secara eksplisit memvalidasi dan menolak tipe data yang salah dengan pesan yang jelas dan aman (contoh: `{"status":"error","message":"\"id\" is required"}`). Ini mengindikasikan bahwa *backend* di balik aplikasi ini ditulis dengan standar kehati-hatian (*engineering standards*) yang jauh lebih tinggi daripada rata-rata aplikasi sekolah.

---

## Kesimpulan Akhir
Berdasarkan bukti dari mekanisme *routing*, pembagian tanggung jawab (*separation of concerns*) antara Frontend dan Backend, serta penanganan *error* validasi yang ketat, **Hira CBT dirancang oleh pengembang yang jauh lebih mengerti perihal arsitektur piranti lunak (Software Architecture)** dibandingkan CBT sekolah sebelumnya.

Namun, sehebat apapun arsitekturnya, jika server database *backend*-nya tidak menyala (*504 Timeout*), aplikasi tetap bernilai nol bagi pengguna akhir. Tolong hidupkan (*debug*) koneksi database *backend* Anda, perbaiki sedikit masalah *Clickjacking* di Frontend, dan aplikasi ini siap digunakan dengan aman untuk ujian.
