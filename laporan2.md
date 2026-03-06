# Laporan Audit Keamanan Lanjutan (Advanced) Tahap 2

**Target:** `https://elaina-ai-silk.vercel.app`
**Tanggal:** 6 Maret 2026

Laporan ini merupakan kelanjutan dari pengujian eksploitasi dan membahas vektor serangan tingkat tinggi (*advanced attack vectors*) pada aplikasi yang berbasis Node.js/Express.

## Ringkasan Eksekutif
Pengujian Tahap 2 berfokus pada kerentanan eksotis dan logika tingkat aplikasi seperti SSRF, entropi token sesi (*Session Hijacking*), miskonfigurasi CORS, serta ketahanan aplikasi terhadap *Denial of Service* (DoS) ringan melalui input JSON raksasa.

## Temuan dan Hasil Analisis

### 1. Entropi Token Sesi (Kerentanan: KRITIS)
*   **Status:** Sangat Rentan terhadap Pembajakan Sesi (*Session Hijacking*)
*   **Analisis:**
    Aplikasi membangkitkan (*generate*) token sesi (`sessionId`) dengan menggunakan fungsi bawaan Javascript:
    `"session_" + Math.random().toString(36).substring(2) + "_" + Date.now()`

    Karena *Math.random()* **bukanlah** Pembangkit Bilangan Acak Semu Kriptografis Aman (*Cryptographically Secure Pseudo-Random Number Generator / CSPRNG*), *seed* yang digunakan V8 engine dapat diprediksi oleh penyerang jika mereka bisa mengumpulkan cukup banyak sampel token. Selain itu, dengan struktur *timestamp* di akhir token (`Date.now()`), penyerang bisa menebak waktu login korban dan mencoba membajak sesinya.
*   **Rekomendasi:** Berhenti menggunakan `Math.random()` untuk keperluan otentikasi. Gunakan pustaka kriptografi bawaan Node.js, contohnya `crypto.randomBytes(32).toString('hex')`, atau beralih sepenuhnya ke standar keamanan seperti JWT (JSON Web Tokens).

### 2. Miskonfigurasi CORS (Cross-Origin Resource Sharing)
*   **Status:** Konfigurasi Longgar (*Misconfiguration*)
*   **Analisis:**
    Server memantulkan (merefleksikan) header `Origin` mana pun kembali sebagai `Access-Control-Allow-Origin: <origin-penyerang>` dan disertai dengan `Access-Control-Allow-Credentials: true`.
    Secara teori, ini adalah mimpi buruk keamanan (*Critical CORS Misconfiguration*). Namun, karena arsitektur aplikasi Anda mengharuskan token dikirim via header `Authorization: Bearer <token>` dan bukan dikirim otomatis oleh browser melalui *Cookie* (seperti `Set-Cookie`), eksploitasi secara praktis cukup sulit karena penyerang tidak bisa memaksa browser korban untuk "otomatis" melampirkan token tersebut melalui web *phishing* / situs pihak ketiga.
*   **Rekomendasi:** Alih-alih merespons dengan header `Origin` yang sama (`*` atau direfleksikan), buatlah *whitelist* array di Express.js: `['https://elaina-ai-silk.vercel.app', 'http://localhost:3000']`.

### 3. Server-Side Request Forgery (SSRF)
*   **Status:** Tidak Rentan
*   **Analisis:**
    Endpoint `/api/chat/multimedia` menerima URL pada atribut `fileUrl` yang mungkin diunduh oleh backend. Kami mencoba melakukan *bypass* untuk mengakses IP internal `127.0.0.1`, layanan AWS Metadata `169.254.169.254`, dan file lokal `file:///etc/passwd`. Semuanya gagal dan memicu status *HTTP 500* `{"error":"Gagal memproses pesan multimedia"}`. Backend tidak membiarkan informasi internal bocor ke luar.
*   **Rekomendasi:** Terapkan pengecekan ketat (seperti library `is-ssrf-safe`) jika aplikasi ke depannya benar-benar melakukan `fetch()` terhadap URL yang disubmit pengguna.

### 4. Denial of Service (DoS) Parsing JSON
*   **Status:** Tahan / Aman
*   **Analisis:**
    Kami mengirim objek JSON bersarang (*nested object* lebih dari 1.000 level) dan JSON string dengan lebih dari 100.000 karakter ke `/api/auth/login`. Aplikasi Express/Vercel (kemungkinan melalui `body-parser` default) dengan sigap menolaknya dan mengembalikan status HTTP *400 Bad Request* atau merespons tanpa mengalami *crash* dan tanpa membuat kehabisan memori.
*   **Rekomendasi:** Tetapkan limit ukuran request yang tegas pada middleware JSON Anda (`express.json({ limit: '10kb' })`) untuk menjaga ketersediaan tetap prima.

## Kesimpulan
Pengujian tahap lanjut berhasil menyorot satu celah yang cukup fatal yaitu cara aplikasi membuat *Session Token* dengan `Math.random()`. Jika penyerang bertekad kuat, mereka bisa mengambil alih akun pengguna yang sedang aktif dengan menebak *timestamp* dan PRNG state-nya. Selain masalah kriptografi tersebut dan perlunya memperbaiki aturan CORS, pertahanan infrastruktur (DoS & SSRF) dari aplikasi ini dapat diandalkan karena beroperasi di atas kerangka *serverless* modern (Vercel).
