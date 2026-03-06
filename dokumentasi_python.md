# Panduan Lengkap Menggunakan API Elaina AI dengan Python

Dokumen ini berisi panduan teknis langkah demi langkah untuk berinteraksi dengan *backend* aplikasi Anda (`https://elaina-ai-silk.vercel.app`) secara langsung menggunakan bahasa pemrograman Python (tanpa melalui *browser*).

Kita akan menggunakan pustaka `requests` yang sangat populer di Python. Jika belum terinstal, Anda bisa menginstalnya dengan menjalankan perintah:
```bash
pip install requests
```

---

## 1. Cara Mendaftar Pengguna Baru (Register)

Pendaftaran pengguna dilakukan dengan mengirimkan *request* `POST` ke endpoint `/api/auth/register`. Anda perlu mengirimkan data dalam format JSON berisi `username` dan `password`.

### Kode Python: Registrasi
```python
import requests
import json

# URL endpoint pendaftaran
register_url = "https://elaina-ai-silk.vercel.app/api/auth/register"

# Data pengguna baru yang ingin didaftarkan
payload = {
    "username": "user_python_001",  # Ganti dengan username yang diinginkan
    "password": "password_rahasia123" # Minimal 6 karakter
}

# Header untuk memberitahu server bahwa kita mengirim format JSON
headers = {
    "Content-Type": "application/json"
}

print("Sedang mencoba mendaftar...")
# Mengirim POST request
response = requests.post(register_url, json=payload, headers=headers)

# Menampilkan hasil
if response.status_code == 200:
    data = response.json()
    if data.get("success"):
        print("Registrasi Berhasil!")
        print("Username:", data.get("username"))
        print("Session ID:", data.get("sessionId"))

        # Simpan Session ID ini! Kita akan membutuhkannya untuk Chat.
        session_id = data.get("sessionId")
    else:
        print("Gagal:", data)
else:
    print(f"Error HTTP {response.status_code}: {response.text}")
```

**Penjelasan:**
*   Aplikasi Anda menuntut `username` unik dan `password` minimal 6 karakter.
*   Jika berhasil, server mengembalikan objek JSON yang salah satunya memuat kunci `"sessionId"`. **Token ini ibarat "Kunci Masuk"** untuk setiap interaksi selanjutnya.

---

## 2. Cara Mengirim Pesan (Chat) ke AI

Setelah mendapatkan `sessionId` (baik dari hasil pendaftaran di atas, maupun dari hasil login di `/api/auth/login`), Anda bisa mulai bercakap-cakap dengan AI.

Anda harus menyertakan token sesi tersebut di dalam parameter **Header `Authorization`** dengan format `Bearer <TOKEN_ANDA>`.

### Kode Python: Chatting dengan AI
```python
import requests

# URL endpoint chat
chat_url = "https://elaina-ai-silk.vercel.app/api/chat"

# Session ID yang didapat dari Register atau Login
# (Ganti nilai ini dengan Session ID asli Anda)
session_token = "session_xyz123_1700000000000"

# Pesan yang ingin dikirimkan ke AI
payload = {
    "message": "Halo Elaina, ceritakan dong fakta unik tentang kucing!"
}

# Menyertakan Token Sesi ke dalam Header Authorization
headers = {
    "Content-Type": "application/json",
    "Authorization": f"Bearer {session_token}"
}

print("Mengirim pesan ke AI, mohon tunggu...")
# Mengirim POST request
response = requests.post(chat_url, json=payload, headers=headers)

# Menangani hasil balasan
if response.status_code == 200:
    data = response.json()
    # Biasanya server membalas dengan JSON yang mengandung balasan AI
    print("\n--- Balasan Elaina AI ---")
    # Struktur respons bergantung pada desain backend Anda.
    # Misalnya jika responsnya adalah {"reply": "Kucing bisa..."}
    print(data)
    print("-------------------------")
elif response.status_code == 400 or response.status_code == 401:
    print("Gagal: Sesi tidak valid atau Anda belum login.")
    print("Detail:", response.json())
else:
    print(f"Server Error {response.status_code}: {response.text}")
```

### Kesalahan Umum (Troubleshooting):
1. **`{"error":"Silakan login terlebih dahulu"}`**: Ini terjadi jika token `sessionId` Anda salah, sudah *expired* (kedaluwarsa), atau Anda lupa menambahkan header `Authorization: Bearer <TOKEN>`.
2. **`{"error":"Gagal terhubung ke AI service"}`**: Ini berarti kredensial *Python* Anda sudah benar dan diizinkan masuk oleh Vercel Anda, tetapi *backend* Node.js Anda sedang mengalami masalah saat meneruskan pesan ke layanan AI asli (seperti OpenAI/Gemini/dll). Periksa ketersediaan kuota API Key AI Anda di sisi *backend*.

---

Dengan menggunakan kedua *script* Python di atas, Anda bebas menciptakan *Command Line Interface (CLI)* khusus AI, *bot Discord/Telegram*, atau mengintegrasikan Elaina AI ke dalam aplikasi desktop otomatis secara independen di luar *browser* Chrome atau Safari!
