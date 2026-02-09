# Arsitektur Keamanan Backend (Go)

Dokumen ini menjelaskan mengapa sistem backend yang dibangun dengan bahasa Go ini memiliki tingkat keamanan yang tinggi dan sangat sulit ditembus oleh peretas (hacker).

---

## 1. Keamanan Berlapis (Defense in Depth)

Aplikasi ini tidak hanya mengandalkan satu pintu keamanan, melainkan menggunakan strategi keamanan berlapis:

### A. Autentikasi JWT & Redis
- **Stateless & Secure**: Menggunakan JSON Web Token (JWT) yang ditandatangani secara digital. Hacker tidak bisa memalsukan token tanpa mengetahui `JWT_SECRET_KEY`.
- **Session Control (Redis)**: Setiap token divalidasi ulang ke database Redis. Jika user logout atau admin memblokir user, token tersebut otomatis tidak bisa digunakan kembali meskipun belum kadaluarsa (Expired).

### B. Otorisasi Berbasis Role (RBAC)
Sistem menggunakan Middleware untuk memastikan user hanya bisa mengakses data yang sesuai dengan haknya.
- **Super Admin**: Akses penuh.
- **Technician/Finance**: Hanya akses ke modul spesifik.
- **Customer**: Hanya bisa melihat data miliknya sendiri.
- *Hacker tidak bisa "melompat" antar akun (IDOR Protection).*

---

## 2. Perlindungan Data Sensitif

### Enkripsi MikroTik (AES-256)
Seperti yang dijelaskan dalam [Cara Kerja Enkripsi.md](file:///home/numbernine/Videos/user/user-service/Cara%20Kerja%20Enkripsi.md), data krusial seperti password MikroTik tidak pernah disimpan dalam bentuk teks biasa. Bahkan jika hacker memiliki akses ke database, mereka hanya akan melihat "sampah digital" tanpa kunci AES yang tersimpan di server.

### Hashing Password (Bcrypt)
Password akun admin/user tidak disimpan di DB. Aplikasi menggunakan **Bcrypt** dengan *cost factor* tinggi. Bcrypt dirancang untuk sangat lambat diproses secara brutal (Brute Force), sehingga memakan waktu bertahun-tahun bagi hacker untuk memecahkan satu password.

---

## 3. Ketahanan Terhadap Serangan Umum (OWASP Top 10)

Backend Go ini telah dilengkapi proteksi terhadap:

-   **SQL Injection**: Menggunakan *ORM / Prepared Statements*. Input user tidak pernah digabungkan langsung ke perintah SQL.
-   **Cross-Site Scripting (XSS)**: Middleware `Secure` di [app.go](file:///home/numbernine/Videos/user/user-service/internal/app/app.go) secara otomatis menambahkan header keamanan (`X-XSS-Protection`, `Content-Security-Policy`).
-   **Brute Force Protection**: Dilengkapi dengan **Rate Limiter** di tingkat API (Redis Based). Jika hacker mencoba ribuan login dalam semenit, IP mereka akan otomatis diblokir sementara.
-   **CORS Protection**: Membatasi domain mana saja yang boleh memanggil API ini, mencegah serangan dari website tidak resmi.

---

## 4. Keunggulan Bahasa Go (Golang)

-   **Compiled Language**: Go dikompilasi menjadi file *binary* tunggal. Tidak ada kode sumber (source code) yang tertinggal di server produksi, berbeda dengan PHP atau Python yang kodenya bisa dibaca jika server ditembus.
-   **Type Safety**: Go adalah bahasa yang *strictly typed*, mencegah banyak bug keamanan yang biasanya disebabkan oleh kesalahan tipe data (seperti *buffer overflow*).
-   **Memory Safety**: Pengelolaan memori yang otomatis (Garbage Collected) mencegah hacker melakukan manipulasi memori server.

---

## Kesimpulan

Sistem ini sangat sulit dibobol karena hacker harus menembus 4 gerbang sekaligus:
1.  **Gerbang Lalu Lintas**: Harus lolos dari Rate Limiter dan CORS.
2.  **Gerbang Akses**: Harus memiliki Kunci JWT dan Session di Redis.
3.  **Gerbang Data**: Harus bisa memecahkan enkripsi AES (membutuhkan Secret Key di server).
4.  **Gerbang Sistem**: Harus bisa menembus proteksi sistem operasi Linux untuk mengambil file `.env`.

> [!IMPORTANT]
> Keamanan terlemah seringkali ada pada **User**. Selalu gunakan password yang kuat dan jangan pernah membagikan file `.env` kepada siapapun.
