# Reflection

## Commit 1: Reflection Notes

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

## Commit 2: Screen Capture

![Commit 2 screen capture](/assets/images/commit2.png)
