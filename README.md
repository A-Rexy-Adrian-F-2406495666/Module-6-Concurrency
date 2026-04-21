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

## Commit 5 Reflection Notes


Sebelumnya, setiap request diproses secara berurutan sehingga satu request yang lambat bisa menghambat request lainnya. Pada tahap ini, saya mengubah server dari single-threaded menjadi multithreaded menggunakan ThreadPool. ThreadPool adalah kumpulan thread yang sudah dibuat di awal (pre-spawned) dan siap menerima pekerjaan. Alih-alih menggunakan `thread::spawn` per request, kita bisa membuat sejumlah thread tetap (misal 4) selama program berjalan.

### Komponen utama pada ThreadPool:

1. Channel sebagai antrian job
    ```rust
    let (sender, receiver) = mpsc::channel();
    ```
    Channel ini dipakai sebagai “jalur komunikasi” antara thread utama dan worker. Thread utama berperan sebagai pengirim job (producer), sementara worker akan mengambil dan menjalankan job tersebut (consumer). Jadi, alurnya mirip antrean: job dikirim, lalu worker ambil satu per satu untuk diproses.

2. `Arc<Mutex<Receiver>>` untuk berbagi receiver

    Karena receiver tidak bisa di-clone begitu saja, kita perlu membungkusnya dengan `Arc` supaya bisa dimiliki bersama oleh banyak thread. Lalu ditambah `Mutex` agar aksesnya tetap aman. Hal ini karena hanya satu worker yang bisa mengambil job pada satu waktu. Ini penting untuk menghindari race condition atau bentrok antar thread.

3. Graceful shutdown via `Drop`

    Saat `ThreadPool` dihentikan (di-drop), channel akan otomatis ditutup karena `sender` sudah tidak ada. Worker yang sedang menunggu job di `recv()` akan menerima error, lalu keluar dari loop dengan sendirinya. Dengan cara ini, semua thread bisa berhenti dengan rapi tanpa ada yang "tersangkut" di belakang layar.

## Commit Bonus Reflection Notes

Pada tahap ini, saya mencoba mengganti fungsi `new` dengan `build` agar proses pembuatan `ThreadPool` menjadi lebih aman. Sebelumnya, kesalahan seperti ukuran thread 0 langsung menyebabkan panic, sekarang error tersebut dikembalikan dalam bentuk `Result` sehingga bisa ditangani dengan lebih baik.

Perbedaan utama antara new dan build ada di cara mereka menangani kondisi error, terutama saat input tidak valid.

- Return type

    `new` langsung mengembalikan `ThreadPool`, sedangkan `build` mengembalikan `Result<ThreadPool, PoolCreationError>`. Artinya, build mengakui sejak awal bahwa proses pembuatan `ThreadPool` bisa gagal, jadi hasilnya dibungkus dalam `Result`.
- Saat input tidak valid

    Pada `new`, jika input seperti ukuran thread = 0, program akan langsung `panic!` atau berhenti. Sementara pada build, kondisi ini tidak langsung menghentikan program, tetapi mengembalikan `Err(...) `yang bisa ditangani.
- Siapa yang menangani error

    Pada `new`, error ditangani oleh runtime (program langsung crash). Sedangkan pada `build`, error diserahkan ke pemanggil (caller), jadi kita bisa memutuskan sendiri mau diapakan error tersebut.

### Kenapa `new` menggunakan `assert!` dianggap kurang ideal
Konvensi Rust menyatakan bahwa `new` seharusnya tidak gagal jika dipanggil, diasumsikan inputnya valid. Dengan `assert!(size > 0)`, program langsung panik saat runtime tanpa memberi kesempatan kepada caller untuk bereaksi.  Hal ini cocok untuk program yang inputnya dijamin valid, tetapi kurang tepat untuk validasi input dari luar.

### Kenapa `build` lebih ekspresif

`build` mengembalikan `Result`, yang berarti kemungkinan gagalnya dikodekan di dalam type system itu sendiri. Compiler memaksa caller untuk menangani kedua kemungkinan. Hal ini sesuai dengan filosofi Rust: _make impossible states unrepresentable dan errors are values_. Caller bisa memilih `unwrap()`, `unwrap_or_else`, `?`, atau pattern matching sesuai kebutuhannya.