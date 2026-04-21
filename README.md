# Reflection

## Commit 1 Reflection Notes

Pada tahap ini, saya mempelajari bagaimana server single-threaded di Rust menerima dan memproses request dari browser menggunakan `TcpListener` dan `TcpStream`.

Program dimulai dengan membuat listener pada alamat `127.0.0.1:7878` menggunakan `TcpListener::bind`. Listener ini berfungsi untuk “mendengarkan” setiap koneksi yang masuk ke server. Ketika ada request dari browser, method `incoming()` akan menghasilkan stream yang merepresentasikan koneksi tersebut. Setiap stream kemudian di-unwrap untuk memastikan tidak terjadi error, lalu diteruskan ke fungsi `handle_connection`.

Fungsi `handle_connection` bertugas membaca data yang dikirim oleh client (browser). Di dalamnya, `BufReader` digunakan untuk membaca stream per baris secara efisien. Method `.lines()` mengubah input menjadi iterator berupa baris-baris teks. Setiap baris kemudian di-unwrap untuk diambil nilai string-nya.

Penggunaan `.take_while(|line| !line.is_empty())` bertujuan untuk membaca hanya bagian header dari HTTP request. Hal ini karena dalam protokol HTTP, header diakhiri dengan baris kosong, sehingga proses pembacaan dihentikan ketika menemui baris kosong tersebut.

Hasil pembacaan disimpan dalam bentuk `Vec<String>` bernama `http_request`, yang kemudian dicetak ke console menggunakan `println!`. Dari output ini, saya dapat melihat struktur request HTTP yang dikirim oleh browser, seperti method (GET), path (/), serta berbagai header seperti Host, User-Agent, dan lain-lain.

Dari percobaan ini, saya memahami beberapa hal:

- Browser berkomunikasi dengan server menggunakan protokol HTTP dalam bentuk teks.
- Server menerima request sebagai aliran data (stream) yang perlu diproses secara manual.
- Header HTTP memiliki format baris per baris dan diakhiri dengan baris kosong.
- Rust menyediakan tools seperti BufReader untuk mempermudah pembacaan data dari stream.

Selain itu, saya juga mempelajari bagaimana browser dapat mengirim beberapa request secara otomatis (retry), sehingga server bisa mencetak beberapa output “Request” meskipun hanya satu kali akses dilakukan.

## Commit 2 Reflection Notes

![Commit 2 screen capture](/assets/images/commit2.png)

Pada tahap ini, saya mempelajari bagaimana server Rust mengirimkan response HTTP berupa halaman HTML ke browser. Sekarang, fungsi `handle_connection` tidak hanya membaca request, tetapi juga mengirim response dengan status line "HTTP/1.1 200 OK" dan header seperti Content-Length. File `hello.html` dibaca menggunakan `fs::read_to_string`, lalu isinya dimasukkan ke dalam response yang dikirim ke client. Saya juga memahami bahwa format response HTTP harus sesuai standar, termasuk pemisahan header dan body menggunakan \r\n\r\n. Hasilnya, browser akhirnya dapat menampilkan halaman HTML sederhana yang dikirim oleh server Rust.

## Commit 3 Reflection Notes

Sebelumnya, fungsi `handle_connection` membaca seluruh HTTP request headers ke dalam sebuah Vec, tetapi tidak pernah benar-benar memeriksa isinya. Akibatnya, server selalu merespons dengan `hello.html` dan status 200 OK untuk setiap request. Ini perilaku yang salah secara semantik karena HTTP mendefinisikan status code berbeda untuk situasi berbeda (200 untuk sukses, 404 untuk tidak ditemukan, dan lain sebagainya).

### Refactoring yang dilakukan
- Pertama, saya mengubah cara membaca request, alih-alih mengumpulkan semua header ke Vec, saya hanya perlu ambil baris pertama saja (request_line) karena baris itulah yang memuat method dan path (GET / HTTP/1.1).
- Kedua, saya menambahkan if-else untuk memeriksa nilai `request_line`:
```rust
let (status_line, filename) = if request_line == "GET / HTTP/1.1" {
    ("HTTP/1.1 200 OK", "hello.html")
} else {
    ("HTTP/1.1 404 NOT FOUND", "404.html")
};
```
Refactoring ini membuat alur kode terasa lebih masuk akal karena ada pemisahan antara membaca request dan menentukan response. Hasilnya, server jadi terlihat lebih “nyata” karena bisa memiliki respons yang berbeda tergantung permintaan dari browser. Hal ini sesuai dengan bagaimana HTTP mendefinisikan status code berbeda untuk situasi yang berbeda.

![Commit 3 screen capture](/assets/images/commit3.png)

## Commit 4 Reflection Notes

Pada tahap ini, saya mencoba mensimulasikan kondisi ketika server memproses request yang lambat dengan menambahkan sleep selama 10 detik pada endpoint `/sleep`. Saya melakukan pengujian dengan membuka endpoint `/sleep` terlebih dahulu, kemudian membuka root request endpoint `/`. Saat keduanya diuji, terlihat bahwa request root endpoint ikut tertunda karena harus menunggu endpoint `/sleep` selesai. Hal ini terjadi karena server masih berjalan secara single-threaded, sehingga hanya bisa memproses satu request dalam satu waktu. Akibatnya, request berikutnya harus menunggu sampai proses sebelumnya selesai. Dari sini saya memahami bahwa pendekatan ini tidak efisien untuk menangani banyak user sekaligus. Simulasi ini menunjukkan pentingnya concurrency atau multi-threading agar server bisa menangani banyak request secara bersamaan.