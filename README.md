## Laporan Penyelesaian Challenge *JavaScript Attack* pada DVWA

**Tujuan:**
Menyelesaikan tantangan *JavaScript Attack* pada DVWA untuk memahami bagaimana token/tokenisasi klien dibuat dari input pengguna (frase) di sisi klien, serta bagaimana memanipulasi/meniru proses itu untuk menyelesaikan challenge pada berbagai level keamanan (Low, Medium, High, Impossible).

### 1. Deskripsi Singkat

Challenge *JavaScript Attack* mensyaratkan mengubah sebuah *phrase* menjadi nilai token yang valid menggunakan rangkaian fungsi JavaScript di sisi klien. Token ini kemudian dikirimkan bersama form untuk menyelesaikan challenge. Pada setiap level (Low → Medium → High → Impossible) mekanisme pembentukan token berbeda—mulai dari operasi sederhana (ROT13 + MD5) hingga rangkaian hashing berlapis (SHA-256) dan penggabungan string—yang harus dipahami dan direplikasi agar token valid dihasilkan.

### 2. Tahapan Pengujian & Observasi

| Level Keamanan |                                                                                                                                                             Mekanisme Tokenisasi (Client-side) | Observasi / Hasil                                                                                                                                                                                                                                                                                |
| -------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------: | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **Low**        |                                                                                                                                       Token dibuat dari *phrase* → di-ROT13 → lalu di-hash MD5 | Mudah direplikasi: jalankan fungsi di console atau hitung offline → masukkan token ke elemen form atau kirim lewat Burp Suite → berhasil menyelesaikan challenge                                                                                                                                 |
| **Medium**     |                                                  Token dibentuk oleh penggabungan string: ada *concatenation* antara parameter internal dan nilai *phrase*, lalu diproses oleh fungsi tertentu | Token masih bisa direkayasa setelah mengikuti proses penggabungan; meniru urutan fungsi di console / Burp Suite menghasilkan token valid → challenge tuntas                                                                                                                                      |
| **High**       | Tokenisasi berlapis: beberapa fungsi dipanggil berurutan (token part 1 → doSomething → token part 2 → hashing SHA-256 → token part 3) dan penggabungan string tambahan (mis. append `zz`/`xx`) | Lebih kompleks tapi masih dapat diotomasi: cukup jalankan fungsi gabungan yang sesuai (atau token part yang dipanggil saat submit) untuk mendapatkan token akhir → berhasil jika proses ditiru dengan benar; lewat Burp Suite perlu memanggil function yang setara untuk mendapatkan token valid |
| **Impossible** |                                Tidak ada challenge — pesan menunjukkan bahwa input klien tidak bisa dipercaya dan tidak ada mekanisme yang mustahil untuk mencegah manipulasi klien sepenuhnya | Tidak tersedia exploit/solusi karena level ini menegaskan prinsip keamanan: jangan percaya input dari client; pendekatan server-side diperlukan                                                                                                                                                  |

### 3. Contoh Langkah Teknis (ringkasan)

* **Low:** `phrase` → ROT13 → MD5 → token. Jalankan di console: peroleh token → submit.
* **Medium:** Gabungkan nilai internal + `phrase` + string tambahan → panggil fungsi → token.
* **High:** Jalankan urutan fungsi (token part 1 → proses → token part 2 → SHA-256 → token part 3) atau panggil fungsi final yang dieksekusi saat tombol submit untuk mendapatkan token akhir.
* Untuk manipulasi lewat Burp Suite: rekam request, modifikasi body/header dengan token yang dihasilkan secara manual atau lewat script, lalu kirim ulang.

### 4. Teknik Mitigasi & Pelajaran Keamanan

* **Jangan mengandalkan keamanan di sisi klien.** Semua logika penting (validasi, verifikasi token) harus dilakukan di server.
* **Server-side verification:** Token yang dihasilkan di sisi klien harus diverifikasi ulang oleh server (mis. cek signature/nonce yang dibuat server-side).
* **Gunakan nonce/CSRF token yang bersifat server-rendered** dan jangan bergantung pada algoritma tokenisasi yang dapat dibalik di client.
* **Obfuscation bukanlah security:** Teknik yang hanya menyulitkan pembacaan JS (minify/obfuscate) tidak mencegah attacker yang bisa mengeksekusi/men-debug skrip di browser.
* **Prinsip:** “Never trust user input” — validasi & autorisasi final harus di server.

### 5. Kesimpulan

* Challenge *JavaScript Attack* mengilustrasikan bahwa proses tokenisasi di sisi klien dapat dianalisis dan direplikasi untuk menyelesaikan challenge bila tidak ada verifikasi server-side.
* Semakin kompleks urutan fungsi dan hashing di sisi klien, semakin sulit namun tetap mungkin untuk direplikasi jika attacker memiliki akses ke kode klien.
* Pencegahan efektif mensyaratkan verifikasi server-side (mis. server-generated nonce, HMAC dengan secret server, atau validasi backend).





## Laporan Pengujian Open HTTP Redirect pada DVWA

**Tujuan:**
Menguji kerentanan *Open HTTP Redirect* pada aplikasi DVWA pada beberapa level keamanan (Low, Medium, High, Impossible) untuk memahami vektor eksploitasi, teknik bypass, dan mekanisme mitigasi yang efektif.

### 1. Deskripsi Singkat

Open HTTP Redirect terjadi ketika aplikasi menerima URL tujuan dari input pengguna (mis. parameter `redirect`) dan langsung mengirimkan respons `Location` tanpa validasi memadai. Hal ini memungkinkan penyerang membuat link yang tampak berasal dari domain tepercaya tetapi mengarahkan korban ke situs phishing/malware. Pengujian ini dilakukan dengan merekam request (Burp Suite), memodifikasi parameter redirect, dan mengamati perilaku tiap level keamanan DVWA.

### 2. Tahapan Pengujian & Observasi

| Level Keamanan |                                                                                            Mekanisme yang Diuji | Observasi / Hasil                                                                                                                                           |
| -------------- | --------------------------------------------------------------------------------------------------------------: | ----------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Low**        |                                      Parameter `redirect` dipakai langsung untuk header `Location` tanpa filter | Redirect ke URL arbitrer berhasil — response `302` dan `Location` mengarahkan ke target eksternal yang diset → rentan                                       |
| **Medium**     |                                 Filter sederhana: tolak jika parameter mengandung substring `http` atau `https` | Menyulitkan input langsung berawalan `http(s)://` tetapi masih dapat dibypass (mis. hilangkan `http`, gunakan relative URL, atau encoding) → masih berisiko |
| **High**       |                        Validasi berbasis pola: `redirect` harus mengandung substring tertentu (mis. `info.php`) | Membatasi target ke path internal tertentu → bypass lebih sulit; namun teknik encoding atau menyisipkan `info.php` dalam parameter bisa diuji               |
| **Impossible** | Nilai `redirect` harus numerik dan hanya menerima nilai terdaftar (mis. `1,2,9`) yang dimapping ke target fixed | Mapping id → target menutup vektor redirect arbitrary → aman dalam skenario pengujian ini                                                                   |

**Catatan pengujian:**

* Semua pengujian direkam dengan Burp Suite (Proxy → Repeater) untuk memodifikasi parameter `redirect` dan mengamati header `Location`.
* Pada level Low, penggunaan halaman PHP sederhana sebagai target mempermudah verifikasi behavior (klik Follow redirect atau cek `Location` header).
* Teknik bypass yang diuji: menghapus `http(s)`, URL-encoding, penggunaan relative paths, dan memasukkan substring yang diperbolehkan.

### 3. Langkah Teknis (ringkas)

1. **Siapkan target redirect** (mis. file `target.php`) untuk menerima parameter/menampung redirect.
2. **Rekam request** ke endpoint Open Redirect DVWA dengan Burp Suite proxy.
3. **Kirim ke Repeater** dan modifikasi parameter `redirect` menjadi URL eksternal / local untuk menguji perilaku.
4. **Periksa respons**: status `302` dan header `Location` mengonfirmasi redirect; gunakan tombol *Follow redirection* untuk verifikasi.
5. **Uji bypass** sesuai level: hilangkan `http(s)`, lakukan URL-encoding, atau tambahkan substring yang dibatasi (mis. `info.php`) untuk menguji filter level Medium/High.
6. **Catat hasil per level** dan dokumentasikan payload yang berhasil/tersangkali terblokir.

### 4. Teknik Mitigasi yang Direkomendasikan

* **Whitelist atau mapping server-side**: jangan menerima URL mentah dari user; gunakan mapping `id → target` (seperti pada level Impossible).
* **Validasi ketat**: hanya izinkan domain/internal paths yang sudah dideklarasikan.
* **Gunakan relative redirects yang tervalidasi**: jika perlu redirect, gunakan nama rute/internal token, bukan URL lengkap dari input user.
* **URL canonicalization & encoding checks**: normalisasi dan dekode parameter sebelum validasi untuk mencegah bypass berbasis encoding.
* **Telemetry & rate-limiting**: catat redirect yang gagal/berulang untuk mendeteksi abuse dan blokir pola mencurigakan.
* **Minimal disclosure**: hindari memberikan pesan error yang mengungkapkan target atau parameter internal.

### 5. Kesimpulan

* Open Redirect adalah vektor serangan sederhana namun berisiko tinggi bila input user langsung menentukan `Location` header.
* Filter berbasis blacklist (mis. larang `http`) mengurangi serangan langsung tetapi rentan terhadap bypass; whitelist/mapping server-side adalah solusi yang paling efektif.
* Level `Impossible` (mapping numerik ke target fixed) menunjukkan pendekatan aman yang praktis untuk mencegah redirect arbitrary.






## Laporan Pengujian Authorization Bypass pada DVWA

**Tujuan:**
Menguji kerentanan *Authorization Bypass* pada aplikasi DVWA di berbagai level keamanan (Low, Medium, High, Impossible) untuk memahami bagaimana pengecekan otorisasi yang lemah dapat dimanfaatkan, teknik pengujian, dan mitigasi efektif.

### 1. Deskripsi Singkat

Authorization bypass terjadi ketika aplikasi gagal memeriksa hak akses/role dengan benar sehingga pengguna biasa (non-admin) dapat mengakses fungsi atau data yang seharusnya hanya untuk admin. Pengujian dilakukan dengan dua akun (admin dan user biasa) dan manipulasi request (langsung URL / cookies / replay request) untuk mencoba mengakses endpoint terbatas.

### 2. Tahapan Pengujian & Observasi

| Level Keamanan |                                                                                               Mekanisme yang Diuji | Observasi / Hasil                                                                                                                                                      |
| -------------- | -----------------------------------------------------------------------------------------------------------------: | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Low**        |                             Tidak ada pemeriksaan otorisasi pada endpoint — akses dikendalikan hanya lewat menu UI | Akses langsung ke URL admin oleh akun non-admin berhasil → rentan terhadap authorization bypass                                                                        |
| **Medium**     |                           Beberapa endpoint dilindungi di UI, namun beberapa endpoint masih dapat diakses langsung | Akses ke beberapa URL ditolak, tetapi ada endpoint lain yang masih dapat diakses oleh user biasa → risiko parsial                                                      |
| **High**       | Proteksi lebih ketat pada banyak endpoint; beberapa request yang hanya bisa lewat POST masih harus diuji via proxy | Sebagian besar endpoint tidak bisa diakses langsung; namun dengan swap cookies / replay request ditemukan 1 endpoint yang masih bisa diakses → masih ada celah praktis |
| **Impossible** |                                                       Validasi otorisasi ketat di server-side untuk semua endpoint | Semua percobaan akses dengan akun non-admin ditolak → tidak ditemukan celah authorization bypass dalam skenario pengujian ini                                          |

**Catatan pengujian:**

* Pengujian menggunakan Burp Suite (Proxy → Repeater) untuk merekam request, menukar cookie/session, dan replay request dengan parameter/headers yang dimodifikasi.
* Untuk endpoint berbasis POST, request dikirim ulang lewat Repeater setelah mengganti cookie/session user untuk menguji apakah server memeriksa role atau hanya mengandalkan UI.

### 3. Langkah Teknis (ringkas)

1. **Siapkan dua akun**: satu admin, satu user biasa (mis. `Gordon`).
2. **Rekam request** saat admin mengakses fitur terbatas (mis. update user).
3. **Logout** dari admin → login sebagai user biasa.
4. **Coba akses langsung** URL fitur admin via browser; jika tidak muncul di menu, akses langsung URL.
5. **Rekam request user biasa** dengan Burp; tukar cookie/session pada request yang direkam admin dengan cookie user biasa atau sebaliknya → kirim ulang (Repeater).
6. **Amati respons**: 200/redirect berarti akses diberikan; 403/401 berarti ditolak. Catat endpoint yang masih rentan.

### 4. Teknik Mitigasi yang Direkomendasikan

* **Server-side authorization checks:** Pastikan setiap endpoint memvalidasi role/permission di server, bukan hanya menyembunyikan link di UI.
* **Return 403 Forbidden** untuk akses yang tidak berhak (jangan hanya sembunyikan link di frontend).
* **Session binding & integrity:** Pastikan session/cookie terikat pada user dan tidak mudah di-replay; validasi session id dan user id pada server.
* **Least privilege:** Terapkan prinsip hak minimal — hanya user dengan role yang tepat diberi akses dan hak tersebut dideklarasikan eksplisit.
* **Audit & logging:** Log upaya akses tidak sah dan deteksi pola replay atau cookie swapping.
* **Input validation & anti-CSRF:** Untuk aksi sensitif (POST/PUT), gunakan token CSRF dan cek origin/referer bila relevan.

### 5. Kesimpulan

* Authorization bypass sering muncul bukan karena bug UI tetapi karena kurangnya pengecekan otorisasi di server-side.
* Teknik pengujian efektif: akses langsung URL, rekam & replay request lewat proxy, dan swap session/cookie untuk memverifikasi apakah server memeriksa role.
* Level `Impossible` pada DVWA memperlihatkan praktik yang benar — menolak semua akses tidak sah (mis. 403) dan memeriksa hak di backend.

