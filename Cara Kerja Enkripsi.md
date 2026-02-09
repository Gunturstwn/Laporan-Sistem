# Cara Kerja Enkripsi MikroTik

Dokumen ini menjelaskan bagaimana aplikasi melindungi informasi sensitif (Host, Username, Password, Port) MikroTik di database namun tetap bisa melakukan login secara otomatis.

## Konsep Utama: Enkripsi Dua Arah (Symmetric Encryption)

Berbeda dengan password user yang menggunakan *Hashing* (satu arah, tidak bisa dikembalikan), kredensial MikroTik menggunakan **Enkripsi AES (Advanced Encryption Standard)**. Ini adalah enkripsi dua arah, artinya data yang sudah dikunci (enkripsi) bisa dibuka kembali (dekripsi) menggunakan **Kunci Rahasia (Secret Key)** yang sama.

---

## Alur Kerja (Workflow)

### 1. Penyimpanan (Encryption)
Saat Anda menambah atau memperbarui data MikroTik melalui dashboard:
1.  Aplikasi menerima input teks biasa (misal: `password123`).
2.  Aplikasi memanggil fungsi `Encrypt(text)` di [conv.go](file:///home/numbernine/Videos/user/user-service/utils/conv/conv.go).
3.  Teks dienkripsi menggunakan algoritma **AES-CFB** dengan `APP_AES_SECRET_KEY` dari file `.env`.
4.  Hasil enkripsi berupa kode acak (Base64) disimpan ke database (misal: `uX8v2B...`).

### 2. Penggunaan / Login (Decryption)
Saat aplikasi perlu melakukan perintah ke MikroTik (misal: cek trafik atau tambah user):
1.  Aplikasi mengambil data terenkripsi dari database.
2.  Aplikasi memanggil fungsi `Decrypt(encoded)` di [conv.go](file:///home/numbernine/Videos/user/user-service/utils/conv/conv.go).
3.  Dengan menggunakan **Secret Key** yang sama, kode acak tadi diubah kembali menjadi teks asli (`password123`).
4.  Teks asli inilah yang dikirimkan ke MikroTik untuk proses login melalui API RouterOS.

---

## Mengapa Ini Aman?

1.  **Perlindungan Database**: Jika seseorang berhasil mencuri atau melihat database Anda, mereka tidak akan bisa melihat IP atau password MikroTik Anda. Mereka hanya akan melihat deretan karakter acak yang tidak berguna tanpa "Kunci Rahasia".
2.  **Kunci Terpisah**: Kunci rahasia disimpan di file `.env` server, bukan di database. Ini menambah lapisan keamanan (*Separation of Concerns*).
3.  **IV (Initialization Vector)**: Sistem menggunakan *random IV* untuk setiap enkripsi. Artinya, meskipun Anda memiliki dua MikroTik dengan password yang sama, hasil enkripsi di database akan terlihat **berbeda**, sehingga lebih sulit ditebak oleh hacker.

---

## Di Mana Pengaturan Kuncinya?

Kunci enkripsi diatur melalui variabel lingkungan di file `.env`:
```env
APP_AES_SECRET_KEY=kunci_rahasia_32_karakter_disini
```

> [!CAUTION]
> **JANGAN PERNAH** mengganti `APP_AES_SECRET_KEY` jika sudah ada data MikroTik di database. Jika kunci diganti, aplikasi tidak akan bisa mendekripsi data lama, dan login ke MikroTik akan gagal (Error: *cipher: message authentication failed* atau data korup).

---

## Ringkasan Teknis
- **Algoritma**: AES (Advanced Encryption Standard).
- **Mode**: CFB (Cipher Feedback).
- **Encoding**: Base64 (agar aman disimpan sebagai teks di DB).
- **Implementasi**: [utils/conv/conv.go](file:///home/numbernine/Videos/user/user-service/utils/conv/conv.go).
