# Dokumentasi Connection Pooling MikroTik

Dokumen ini menjelaskan implementasi dan cara kerja **Connection Pooling** untuk MikroTik dalam project ini.

## Apa itu Connection Pooling?

Connection pooling adalah teknik untuk memelihara sekumpulan koneksi database atau API yang tetap terbuka (open), sehingga koneksi tersebut dapat digunakan kembali (reused) untuk permintaan berikutnya. Hal ini jauh lebih efisien daripada membuka dan menutup koneksi baru setiap kali ada permintaan hit endpoint.

## Implementasi dalam Project

Sistem ini diimplementasikan melalui dua komponen utama:
1.  **`MikrotikPool`**: Pengelola utama koneksi yang menggunakan pola *Singleton*.
2.  **`NewMikrotikConnection`**: Fungsi *entry point* yang digunakan oleh servis lain untuk mendapatkan koneksi.

### Lokasi File:
-   [mikrotik_pool.go](file:///home/numbernine/Videos/user/user-service/config/mikrotik_pool.go) (Logika Inti Pool)
-   [mikrotik.go](file:///home/numbernine/Videos/user/user-service/config/mikrotik.go) (Wrapper & Retry Logic)

---

## Fitur Utama

### 1. Singleton Pattern
Sistem hanya menjalankan satu instance pool untuk seluruh aplikasi. Ini memastikan semua servis berbagi "wadah" koneksi yang sama.
```go
func GetPool() *MikrotikPool {
    poolOnce.Do(func() {
        globalPool = &MikrotikPool{ ... }
    })
    return globalPool
}
```

### 2. Get or Create Logic
Setiap kali servis membutuhkan koneksi, pool akan mengecek apakah koneksi untuk MikroTik ID tersebut sudah ada:
-   **Jika Ada**: Mengecek kesehatan koneksi. Jika sehat, langsung dikembalikan.
-   **Jika Tidak Ada / Rusak**: Melakukan login ulang dan menyimpannya ke dalam pool.

### 3. Health Monitoring & Recovery
Aplikasi tidak sekadar mengambil koneksi yang ada, tapi juga melakukan validasi:
-   **`isConnectionAlive`**: Menjalankan perintah ringan (`/system/identity/print`) untuk memastikan socket masih terbuka.
-   **Retry Logic**: Jika sebuah perintah gagal karena "broken pipe" atau "invalid credentials", system akan menghapus koneksi tersebut dari pool dan mencoba koneksi ulang **secara otomatis**.

### 4. Background Cleanup
Untuk mencegah kebocoran resource atau terlalu banyak koneksi menggantung di MikroTik, terdapat *goroutine* yang berjalan setiap 5 menit untuk menutup koneksi yang tidak digunakan (idle) selama lebih dari **10 menit**.

---

## Alur Penggunaan dalam Kode

Servis (seperti `PPPoEService`) tidak perlu tahu cara melakukan login. Cukup panggil:

```go
// Mendapatkan koneksi dari pool
conn, err := config.NewMikrotikConnection(mikrotikDevice)
if err != nil {
    return err
}

// Jalankan perintah (Reuse koneksi yang sudah ada)
reply, err := conn.Run("/ppp/secret/print")
```

---

## Keuntungan Sistem Ini

1.  **Performa Cepat**: Menghilangkan latency "handshake" TCP dan proses autentikasi MikroTik yang memakan waktu (rata-rata menghemat 100ms - 500ms per request).
2.  **Stabilitas MikroTik**: Mencegah MikroTik kelebihan beban akibat terlalu banyak proses *login/logout* dalam waktu singkat.
3.  **Resilience**: Jika MikroTik restart, aplikasi akan mendeteksi koneksi mati dan menyambung kembali secara transparan tanpa perlu restart aplikasi Go.

---

## Apakah Cocok untuk MikroTik Resource Kecil (RAM/CPU Rendah)?

**JAWABANNYA: SANGAT COCOK & DISARANKAN.**

Banyak yang mengira menjaga koneksi tetap terbuka akan membebani MikroTik, namun faktanya justru sebaliknya. Berikut alasannya:

### 1. Mengurangi Beban CPU (CPU Overhead)
Proses login (autentikasi) dan pembuatan socket baru adalah proses yang cukup berat bagi CPU MikroTik ber-resource kecil. Jika aplikasi Anda hit API setiap beberapa detik tanpa pooling, CPU MikroTik akan sering melonjak (*spike*) hanya untuk memproses handshake login. Dengan **Pooling**, CPU hanya bekerja berat sekali di awal.

### 2. Efisiensi RAM MikroTik
Koneksi API yang *idle* (diam) hanya memakan resource RAM yang sangat kecil (beberapa KB saja). MikroTik hAP lite dengan RAM 32MB pun sanggup menangani beberapa koneksi API yang terbuka secara persisten tanpa gangguan performa.

### 3. Mencegah Log Spamming
Setiap kali ada login baru, MikroTik akan mencatatnya ke Log. Jika tanpa pooling, Log MikroTik Anda akan penuh dengan pesan "user logged in" dan "user logged out", yang secara tidak langsung juga memakan resource disk/RAM untuk penyimpanan log.

### 4. Batasan yang Perlu Diperhatikan
Meskipun sangat disarankan, pastikan hal berikut:
-   **Max Sessions**: Secara default, MikroTik membatasi jumlah user API yang bisa login bersamaan. Karena aplikasi Anda menggunakan *pool* yang dikontrol dari satu server, ini justru mengamankan agar jumlah koneksi tidak melebihi batas (limit).
-   **Idle Timeout**: Pengaturan 10 menit dalam kode Anda sudah ideal. Jika MikroTik sangat kecil, Anda bisa memperpendek menjadi 5 menit jika dirasa perlu, namun 10 menit adalah angka yang aman.

---
> [!TIP]
> Untuk MikroTik dengan resource kecil, pooling adalah **solusi terbaik** untuk menjaga stabilitas perangkat agar tidak sering mengalami *hang* akibat lonjakan CPU saat proses autentikasi berulang.
