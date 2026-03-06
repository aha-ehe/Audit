# Laporan Audit Keamanan Lanjutan (Advanced) Tahap 5

**Target:** `https://elaina-ai-silk.vercel.app`
**Tanggal:** 6 Maret 2026
**Fokus Pengujian:** Penyalahgunaan Kuota API AI (Denial of Wallet / API Exhaustion)

Laporan Tahap 5 ini dikhususkan untuk menjawab pertanyaan krusial mengenai ketahanan aplikasi dari serangan spesifik pada sistem berbasis AI: Bagaimana jika penyerang berusaha menyedot habis kuota (API Key) AI yang Anda miliki dengan mengeksploitasi fitur chat?

## Ringkasan Eksekutif
Berdasarkan simulasi serangan yang menggunakan puluhan "bot CLI" bersamaan yang bernaung di bawah **satu akun terdaftar**, aplikasi Anda terbukti **sangat rentan** terhadap serangan *Denial of Wallet (DoW)*. Server meneruskan setiap pesan dari klien ke penyedia AI tanpa mempedulikan batas wajar penggunaan per menit per pengguna. Hal ini dapat berujung pada kerugian finansial yang signifikan atau pemblokiran API Key AI oleh provider karena *abuse* (penyalahgunaan).

## Temuan: Simulasi Serangan *Denial of Wallet* (DoW)

### 1. Eksploitasi Paralel (Konkuren)
*   **Status:** Sangat Rentan (Tingkat Keparahan: KRITIS)
*   **Skenario Serangan:**
    Kami mendaftarkan sebuah akun baru yang sah pada aplikasi Anda dan mendapatkan satu `sessionId` yang valid.
    Menggunakan *script* bot CLI buatan, kami mengirim **100 *request* pesan panjang (`Pesan panjang... AAAAAAA...`) secara serentak (konkuren)** ke endpoint autentikasi `/api/chat` dalam waktu kurang dari dua detik.
*   **Hasil Observasi Server:**
    *   Server memproses **100/100 (100%)** dari permintaan tersebut.
    *   Tidak ada satu pun permintaan yang ditolak atau direspons dengan HTTP *429 Too Many Requests*.
    *   Server gagal mengenali anomali di mana satu pengguna mencoba melakukan "chatting" 50 kali per detik.
*   **Dampak Potensial ke API Key AI:**
    1.  **Tagihan Membengkak (Financial Drain):** Karena layanan LLM AI mengenakan tarif per 1000 Token, *payload* yang disengaja dikirim sangat panjang oleh penyerang akan dikalikan dengan ribuan permintaan per jam, memakan ratusan hingga ribuan dolar tagihan.
    2.  **Pemblokiran Layanan:** Provider API AI umumnya menerapkan limitasi *Tier-based*. Permintaan *brute-force* ini dapat menyebabkan akun AI Anda diblokir sementara (*Quota Exceeded*) atau bahkan ditutup permanen (*Suspended*), membuat aplikasi *Elaina AI* lumpuh (*Down*) bagi seluruh pengguna organik lainnya.

## Solusi Mitigasi Lapis Ganda (Defense in Depth)

Karena aplikasi Anda sepenuhnya gratis dan dapat didaftarkan oleh siapa saja (seperti ditunjukkan di Tahap 3 bahwa tidak ada *rate-limit* pada halaman Registrasi), menutup satu akun bot tidaklah cukup. Anda harus menerapkan kombinasi perlindungan berikut:

### 1. Pembatasan Laju Level Pengguna (User-Level Rate Limiting)
Implementasikan pembatasan berdasarkan *User ID* atau *Session ID*, bukan hanya sekedar IP.
Misalnya menggunakan pustaka `express-rate-limit`:
*   **Batas Normal:** Maksimal 5 *request* pesan ke `/api/chat` setiap 1 menit per sesi pengguna.
*   Jika melebihi, kembalikan HTTP *429 Too Many Requests* dengan pesan ramah *"Tunggu sebentar sebelum mengirim pesan lagi."*

### 2. Batas Panjang Token (Max Token / Character Length)
Jangan izinkan klien mengirim *payload* *string* sepanjang yang mereka mau. Lakukan pemotongan ( *truncation* / *trimming* ) di sisi server:
*   Pesan maksimum: 500 karakter atau ~100 token.
*   Jika input melebihi 500 karakter, buang sisa karakternya atau tolak dengan HTTP 400 *Bad Request*. Ini mencegah penyerang menyedot biaya tagihan lebih cepat melalui parameter *prompt* berukuran raksasa.

### 3. Sistem Antrean / Pemblokiran Konkurensi
Jangan biarkan satu pengguna mengeksekusi panggilan AI lebih dari satu kali dalam waktu bersamaan.
*   Saat pengguna mengirim `Pesan A`, simpan status *Flag/Lock* (misalnya `isGenerating: true`) di memori (*Redis/Cache/Database*) untuk sesi tersebut.
*   Jika `Pesan B` masuk saat `isGenerating` masih *true* (contoh jika penyerang menggunakan multi-CLI), tolak seketika dengan pesan: *"Tunggu hingga AI selesai menjawab pesan Anda sebelumnya."*

## Kesimpulan Tahap 5
Pertanyaan Anda tentang kemungkinan disedotnya kuota API secara massal melalui CLI (*Client Command Line*) sangat relevan dan tepat sasaran. Aplikasi web Anda saat ini benar-benar tidak terlindungi dari eksploitasi API AI (*Denial of Wallet*). Memprioritaskan penambahan `Rate-Limiter` di *endpoint* `/api/chat` harus menjadi langkah utama Anda sebelum mempublikasikan sistem AI ini secara komersial atau berskala besar.
