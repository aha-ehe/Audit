# Laporan Audit Keamanan Lanjutan (Advanced) Tahap 4

**Target:** `https://elaina-ai-silk.vercel.app`
**Tanggal:** 6 Maret 2026

Laporan Tahap 4 merupakan puncak pengujian keamanan yang mengulik kelemahan pada lapisan *Edge Caching*, arsitektur autentikasi ganda, interaksi basis sistem file, dan kelemahan *Client-Side UI*.

## Ringkasan Eksekutif
Secara arsitektur, *backend serverless* Vercel melindungi aplikasi dari eksploitasi berbasis *file system* atau pencurian *cache* dinamis. Namun di sisi pengguna (*client-side*), aplikasi sangat rentan diculik atau disematkan ke dalam web berbahaya (*Clickjacking*) karena absennya header perlindungan yang esensial.

## Temuan dan Observasi Utama

### 1. Clickjacking / UI Redressing (Tingkat: TINGGI)
*   **Status:** Sangat Rentan
*   **Analisis Eksploitasi:**
    Kami telah membuat situs web jebakan (*attacker server*) yang menyematkan halaman utama `https://elaina-ai-silk.vercel.app` ke dalam tag `<iframe>` yang dibuat tembus pandang (*transparent*).
    Situs target berhasil dimuat ke dalam *iframe* penyerang secara mulus tanpa pemblokiran browser. Hal ini membuktikan tidak adanya implementasi Header Keamanan `X-Frame-Options: DENY/SAMEORIGIN` atau `Content-Security-Policy: frame-ancestors`.
*   **Dampak Potensial:**
    Penyerang dapat mengelabui korban untuk login atau memencet tombol penting di aplikasi `Elaina AI` saat pengguna sebenarnya mengira mereka sedang menekan tombol "Dapatkan Uang Gratis" di situs web peretas.
*   **Rekomendasi Utama:** Konfigurasikan Vercel (`vercel.json`) atau middleware Express (`helmet.js`) untuk senantiasa menyertakan Header `X-Frame-Options: DENY` (atau `SAMEORIGIN`) serta `Content-Security-Policy`.

### 2. Session Fixation (Fiksasi Sesi)
*   **Status:** Tangguh / Aman
*   **Analisis Eksploitasi:**
    Dalam eksperimen ini, kami (sebagai penyerang) berusaha memaksakan sesi spesifik yang kami buat sendiri (`session_evil_123`) saat melakukan login dengan akun sah. Tujuannya adalah agar jika token ini diumpankan kepada korban, peretas bisa membajak sesi tersebut.
    **Hasilnya:** Sistem *backend* dengan sigap membuang/mengabaikan token paksaan tersebut dan membangkitkan (*generate*) token `sessionId` baru yang unik. Ini artinya kerangka logika *Session Management* (Manajemen Sesi) ketika Login telah dikerjakan dengan sangat baik (mencegah eksploitasi fiksasi).

### 3. Web Cache Poisoning & Deception (Vercel Edge)
*   **Status:** Tangguh / Aman
*   **Analisis Eksploitasi:**
    Aplikasi *serverless* acap kali rentan karena CDN secara agresif melakukan *caching* statis terhadap halaman berakhiran `.css` atau `.jpg`. Kami memanipulasi URL dinamis menjadi `/api/auth/status?style.css` agar Vercel CDN menyangka itu adalah *stylesheet* dan menyimpan respons status rahasia tersebut (Cache Deception).
    **Hasilnya:** Mekanisme *router* memisahkan URL dengan rapi. Vercel merespons permintaan dinamis (`/api/...`) dengan Header `x-vercel-cache: BYPASS` atau `MISS`, menolak secara cerdas permohonan untuk men-*cache* respon API, meski ekstensinya dimanipulasi.

### 4. Path Traversal & Local File Inclusion (LFI)
*   **Status:** Tangguh / Aman
*   **Analisis Eksploitasi:**
    Kami mencoba membocorkan struktur kode *backend* dengan cara mengunggah dan meminta rute `../../../../etc/passwd` maupun versi *URL-encoded* via parameter `fileUrl` pada API chat. Permintaan ini digagalkan oleh sistem *Exception/Error Handling* yang tertutup dengan status konstan *HTTP 500 Gagal memproses pesan*.

## Kesimpulan Tahap 4
Aplikasi berhasil mempertahankan integritas di balik *Edge Server* (Vercel) dari berbagai gempuran berbasis URL (seperti Poisoning, SSRF, & Traversal) serta mempertahankan rotasi sesi yang aman saat otentikasi. Sayangnya, lapisan *Frontend* tertinggal di belakang tanpa pertahanan *anti-iframe* dasar. Mengatasi celah *Clickjacking* (serta *Brute Force* dan Entropi Sesi dari laporan sebelumnya) akan mendorong *website* ini masuk ke tingkat keamanan setara "Aplikasi Produksi Perbankan / Enterprise".
