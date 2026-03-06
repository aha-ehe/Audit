# Laporan Audit Keamanan Lanjutan (Advanced) Tahap 3

**Target:** `https://elaina-ai-silk.vercel.app`
**Tanggal:** 6 Maret 2026

Laporan Tahap 3 berfokus pada dinamika interaksi tingkat HTTP dan kelakuan *backend* pada kondisi *edge-case* (kasus batas) yang sering kali disalahgunakan untuk memperoleh pijakan awal eksploitasi, seperti *Race Conditions*, percobaan *Brute-Force*, dan manipulasi parameter serta tipe data konten.

## Temuan dan Observasi Utama

### 1. Ketiadaan Limitasi Laju (Rate-Limiting) pada Otentikasi
*   **Status:** Rentan (Kerentanan Terdokumentasi)
*   **Analisis Eksploitasi:**
    Sebuah *script* serang telah mengirimkan **50 permintaan login beruntun** secara intensif dan berkelanjutan (hit per detik tinggi) ke `/api/auth/login`.
    Semua (100%) permintaan tersebut ditangkap dan diproses oleh server, menghasilkan status "Username tidak ditemukan".
*   **Dampak Potensial:**
    Aplikasi sama sekali tidak memiliki perisai terhadap serangan *Brute-Force* (memukul *password* secara acak) dan *Credential Stuffing* (memasukkan daftar *password* hasil kebocoran pihak ketiga). Karena batas panjang *password* minimum hanya 6 karakter, *botnet* dapat menembus akun pengguna manapun hanya dengan menebak variasi *password* secara iteratif tanpa takut diblokir oleh CAPTCHA atau diblokir alamat IP-nya.
*   **Rekomendasi Utama:** Gunakan `express-rate-limit` khusus untuk endpoint login dan register, dan pastikan konfigurasi Vercel mendeteksi/memblokir IP klien.

### 2. Pengujian Kondisi Balapan (Race Conditions)
*   **Status:** Tangguh / Aman
*   **Analisis Eksploitasi:**
    Kerentanan *Race Condition* (*Time of Check to Time of Use* / TOCTOU) terjadi apabila dua *request* tiba pada milidetik yang sama, dan sistem gagal melakukan penguncian (*database lock*) saat mengecek *"Apakah username X sudah terdaftar?"*.
    Kami mengirim 10 permintaan registrasi *concurrent* (bersamaan secara absolut) dengan payload JSON username yang sama.
    **Hasilnya:** Hanya 1 permintaan yang sukses (HTTP 200), sementara 9 permintaan lainnya diblokir seketika dengan pesan "Username sudah digunakan" (HTTP 400).
*   **Kesimpulan:**
    Sistem penyimpanan/database di *backend* (kemungkinan MongoDB/Mongoose atau Prisma) telah mengimplementasikan konstrain keunikan (*unique constraints*) yang kuat pada *field username*, menahan segala bentuk *Race Conditions*.

### 3. Eksploitasi Manipulasi Parameter dan Type-Juggling
*   **Status:** Aman dari Eksploitasi Pintu Belakang (*Bypass*)
*   **Analisis Eksploitasi:**
    Kami menginjeksi input anomali seperti struktur Array (`{"username": ["admin"]}`) dan Nilai Nol (`{"username": null}`) untuk menggoyahkan fungsi `length` atau pencarian string (misal bypass autentikasi array seperti kerentanan PHP *Type Juggling*).
    Aplikasi ini ditulis dalam Node.js (JavaScript), yang secara keliru akan mentolerir beberapa tipe konversi yang buruk. Namun, endpoint menangani dan mem-validasi logika ini secara rapi. Permintaan secara aman dikembalikan dengan respons galat biasa tanpa menyetujui *login*.
*   **Rekomendasi:** Terapkan standar skema ketat (seperti JSON Schema/Zod) agar server di awal sudah menolak input selain *String*.

### 4. Manipulasi Metode dan Kebocoran Stack Trace
*   **Status:** Tertangani Sebagian (*Partial Mitigation*)
*   **Analisis Eksploitasi:**
    Sebagian aplikasi Node.js/Express sering membocorkan struktur folder server (*path disclosure*) atau kerangka bahasa (kode di baris 10, dsb) manakala ia *"crash"* dikarenakan format konten yang aneh.
    Saat mengirim *payload* valid JSON namun dengan identitas HTTP Header `Content-Type: text/plain`, aplikasi gagal mem- *parsing* konten. Namun aplikasi tidak panik dengan cara yang tidak aman; ia memberikan pesan generik *HTTP 500 Terjadi kesalahan server* tanpa menyingkapkan detail teknis apa pun (seperti *stack trace* error Express default di lingkungan pengembangan/ *development*).
*   **Kesimpulan:**
    Variabel lingkungan (Environment Variable) aplikasi di Vercel tampaknya sudah di- *set* sebagai "Production" atau setidaknya penanganan galat (*error handler*) akhir telah menyamarkan error internal, ini merupakan pertahanan yang bagus.

## Kesimpulan Akhir Tahap 3
Sistem menunjukkan level pertahanan *engineering* yang cukup matang dan stabil dalam mengelola logika transaksi, penguncian ganda, serta pengalihan *error stack*. Ironisnya, karena aplikasi melangkah maju tanpa hambatan *(rate-limiter)*, ini membuat pintu gerbang utama (halaman login) rentan didobrak habis-habisan (Brute Force). Menutup celah Brute Force dan Token Entropi yang ditemukan di Tahap 2 sudah cukup untuk membuat aplikasi web ini secara fungsional sangat aman (*Highly Secure*).
