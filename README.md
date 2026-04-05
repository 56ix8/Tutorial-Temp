# Tutorial Setup TempMail Open API

Tutorial ini fokus buat nyambungin domain lu ke Cloudflare dan nge-link backend (Worker) ke frontend (index.html). Asumsinya lu udah punya Worker dan D1 Database yang udah jalan.

## 1. Sambungin Nameservers & Routing Domain
Langkah ini buat mastiin email yang dikirim ke domain lu masuk ke Worker.

1. Buka tempat lu beli domain, terus ubah pengaturan Nameserver-nya ke Nameserver Cloudflare lu.
2. Tunggu beberapa menit sampe status domain lu di dashboard Cloudflare berubah jadi Active.
3. Kalo udah aktif, buka dashboard Cloudflare, klik domain lu, terus masuk ke menu Email > Email Routing.
4. Pastiin Email Routing udah nyala. Terus pindah ke tab Routing rules.
5. Scroll ke bawah cari Catch-all address, klik Edit.
6. Set Action ke "Send to a Worker" dan pilih nama Worker TempMail lu di kolom Destination. Klik Save.

## 2. Sambungin Backend ke index.html
Langkah ini buat nyambungin layar UI web lu sama mesin Worker-nya.

1. Buka file `index.html` di browser. Nanti bakal otomatis muncul form setup (SYS.INIT).
2. Di kolom Endpoint.URL, masukin link Worker Cloudflare lu. 
   (Contoh: https://nama-worker.username.workers.dev) 
   *Catatan: Pastiin ga ada tanda garis miring (/) di ujung belakang URL-nya.*
3. Di kolom Valid.Domains, masukin nama domain lu. Kalo ada banyak domain, pisahin pake koma.
   (Contoh: dealegon.com, domain-lain.net)
4. Klik tombol EXECUTE.

Udah beres. UI lu sekarang udah nyambung ke backend dan webnya udah siap dipake buat nerima email.
