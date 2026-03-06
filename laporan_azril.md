# Laporan Audit Keamanan Website

**Target:** `https://azril-ai.vercel.app/`
**Tanggal:** 6 Maret 2026

## Ringkasan Eksekutif
Berdasarkan hasil analisis dan penetrasi pengujian pada `https://azril-ai.vercel.app/`, arsitektur aplikasi ini sangat berbeda dengan web sebelumnya. Aplikasi ini tidak memiliki *backend* (API Server) sama sekali. Aplikasi sepenuhnya berjalan di *Client-Side* (browser) menggunakan file HTML dan `script.js`, yang secara langsung menghubungi layanan AI pihak ketiga (Groq API).

Pendekatan *Client-Side* ini memunculkan satu **celah keamanan sangat kritis (CRITICAL)** yang dapat menyebabkan kerugian finansial atau hilangnya akses layanan AI Anda.

---

## Temuan dan Hasil Analisis

### 1. Hardcoded API Key (Tingkat: KRITIS)
*   **Status:** Sangat Rentan / Telah Dieksploitasi dalam Simulasi
*   **Analisis:**
    Kunci rahasia API Groq Anda (`gsk_R1Vl1xoDivT1...[REDACTED]...`) tertulis secara "telanjang" (hardcoded) di dalam file `script.js` baris pertama.
    Setiap orang yang mengunjungi web Anda bisa membuka *View Page Source* atau menekan F12 (*Developer Tools*) dan langsung melihat kunci tersebut.
*   **Dampak Potensial (Denial of Wallet):**
    Penyerang tidak perlu menyerang web Anda. Mereka cukup menyalin API Key tersebut dan menggunakannya di komputer/server mereka sendiri untuk membuat aplikasi AI, *spam*, atau *botnet*. Hal ini akan menyedot habis kuota *request* harian/bulanan akun Groq Anda (Groq membatasi token/hari, misalnya 12.000 token per menit untuk tier tertentu). Akun Anda bisa terkena tagihan raksasa (jika berbayar) atau diblokir permanen (jika gratis).
*   **Rekomendasi Utama:**
    **SEGERA CABUT (REVOKE) API KEY INI DARI DASHBOARD GROQ ANDA.**
    Jangan pernah meletakkan rahasia (API Key, Database Password, dll) di kode *frontend* (JS/HTML). Anda **wajib** membuat *backend* sederhana (misal menggunakan Node.js/Vercel Serverless Functions) sebagai jembatan. *Frontend* akan memanggil *backend* Anda, lalu *backend* Anda lah yang menyimpan API Key di `.env` dan meneruskannya ke Groq secara tertutup.

### 2. Cross-Site Scripting (XSS) & Prompt Injection
*   **Status:** Tangguh (Secara Fungsional)
*   **Analisis:**
    Kami menguji *Prompt Injection* dengan menyuruh AI membuang instruksi `system` (sebagai AZRIL AI) dan mencetak *payload* `<script>alert(1)</script>`. AI merespons sesuai perintah, tetapi karena Anda menggunakan pustaka `marked.js` dengan fungsi *renderer* khusus:
    `.replace(/</g, "&lt;").replace(/>/g, "&gt;")`
    Maka eksekusi skrip berbahaya di dalam blok kode berhasil dinetralkan.
*   **Kesimpulan:** Aplikasi aman dari *Stored/Reflected XSS* ketika menampilkan jawaban AI berformat blok kode, asalkan *renderer* ini juga memfilter output teks di luar blok kode (*paragraph/list*).

### 3. Clickjacking (UI Redressing)
*   **Status:** Rentan
*   **Analisis:**
    Server secara eksplisit merespons dengan Header `Access-Control-Allow-Origin: *` dan absennya perlindungan `X-Frame-Options` atau `Content-Security-Policy: frame-ancestors`.
*   **Dampak:** Web `azril-ai.vercel.app` dapat disematkan (di-*embed*) melalui tag `<iframe>` di situs web mana pun (termasuk situs peretas) tanpa izin.
*   **Rekomendasi:** Tambahkan file `vercel.json` di *root* proyek Anda untuk mengonfigurasi `X-Frame-Options` menjadi `DENY` atau `SAMEORIGIN`.

### 4. Denial of Service (Aplikasi Server)
*   **Status:** Tidak Berlaku (N/A)
*   **Analisis:** Karena web Anda hanya merupakan situs statis (*static site* HTML/JS) yang dipangku oleh CDN Vercel, ia sangat kebal terhadap serangan beban lalu lintas (DDoS). Semua beban komputasi AI ditanggung oleh API Groq, bukan oleh server Vercel Anda.

---

## Kesimpulan Akhir
Desain web `azril-ai.vercel.app` saat ini lebih menyerupai purwarupa (*prototype*). Praktik meletakkan Kunci API secara langsung di `script.js` adalah **kesalahan paling fatal dalam pengembangan web**.

Langkah pertama yang mutlak harus Anda lakukan hari ini juga adalah:
1. Masuk ke akun [Groq Console](https://console.groq.com/keys).
2. Hapus kunci API `gsk_R1Vl1xo...`.
3. Buat kunci baru dan simpan hanya di server *backend* (API Endpoint), jangan ditampakkan lagi ke *Client-Side*.
