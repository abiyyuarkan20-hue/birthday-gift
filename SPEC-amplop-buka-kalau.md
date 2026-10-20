# SPEC — Fitur "Amplop Buka Kalau…" ✉️

Dokumen ini untuk mengarahkan agent AI yang memodifikasi `index.html` (web ulang tahun Zahwa Davina, tema 8-bit + misi rahasia "Operasi 26/10"). Pengirim: Abiyyu, yang menyebut dirinya **Arkan** di semua teks. Tujuan fitur ini: membuat web **tetap dibuka berbulan-bulan setelah 26 Oktober 2026**.

---

## 0. Konteks dan aturan umum

### 0.1 Aturan keras (sama dengan fitur sebelumnya)
- Jangan menulis ulang dari nol. Jangan mengubah perilaku fitur yang sudah ada.
- Satu file `index.html`, Tailwind CDN + JS vanilla. Tanpa library baru, tanpa file gambar/audio baru. Semua pixel art lewat SVG (`rect`, `shape-rendering="crispEdges"`), CSS, atau Canvas.
- Mobile-first 360×640. Harus tetap rapi di 320×568. Tidak boleh ada scroll horizontal.
- Semua teks Indonesia dan semua lewat `CONFIG.amplop`. Tidak ada teks hardcode di HTML/JS, kecuali label teknis.
- Hormati `prefers-reduced-motion` dan tombol mute (`sfxOn`).
- Tidak ada game over dan tidak ada jalan buntu: setiap layar punya tombol kembali.
- `localStorage` selalu dalam `try/catch`, dan fitur tetap jalan (dengan memori sementara) kalau storage gagal.
- Semua timer/interval/listener global dibersihkan di `cleanup()`.
- Tidak ada kamera, tidak ada data yang dikirim keluar.
- Desain orisinal: jangan meniru logo, ikon, suara, atau karakter yang sudah ada.
- Setelah tiap tahap, ekstrak skrip dan jalankan `node --check`, lalu pastikan tidak ada error di console.

### 0.2 Cek awal wajib sebelum mulai (pra-syarat)
Baca seluruh file dulu: `CONFIG`, `go()`, `order`, `inits`, `levelClear()`, `cleanup()`, helper `$`, `$$`, `fill`, `rand`, `beep`, `sfx`, `confetti`, `typeText`, `pixelHeart`, `toast`, fungsi templating `{namaPanggilan}`/`{pengirim}`, logika layar `lock`, pemutar pesan suara di `#after`, dan modul `maskot`, `pet`, `kueManager`, `kontrak`.

Pastikan hal berikut sebelum menambah fitur (versi lampiran sebelumnya memilikinya):
1. `maskot` **harus terdefinisi** (objek dengan `show/hide/say/setMood`). Kalau belum, `go()` melempar `ReferenceError: maskot is not defined` di setiap pindah layar. Perbaiki lebih dulu.
2. Kelas `.btn-yl` harus ada di CSS.
3. `#petPanel` memakai `inset-x: 0` yang bukan properti CSS valid. Ganti dengan `left:0; right:0; margin-inline:auto; max-width:36rem`.

### 0.3 Peta z-index yang sudah ada (jangan bentrok)
HUD 40 · pet dock 44 · maskot 45 · scanline 55 · hujan bintang 59 · confetti 60 · dexOverlay 65 · pet scrim 65 · pet panel 66 · dialogScrim 67 · os-win 68 · toast 70 · dexBanner 80.
Fitur ini memakai: **layar amplop** (di dalam sistem `.screen`, tanpa z-index khusus) dan **sheet surat 63** (di bawah pet panel dan dex agar tidak menimpa fitur lain, di atas confetti dan maskot).

### 0.4 Kunci localStorage
| Kunci | Isi |
|---|---|
| `amplop-2610` | `{v:1, dibuka:{id:{n,terakhir,idx}}, favorit:[id], kunjungan:n, pertama:ISO}` |
| `misi-selesai-2610` | `"1"` setelah finale pernah tercapai |

`?reset=1` yang sudah ada harus ikut menghapus kedua kunci ini.

---

## 1. Konsep

Satu layar baru **`amplop`** berisi **laci amplop**. Tiap amplop bertuliskan "Buka kalau…" dan berisi pesan pendek dari Arkan untuk situasi tertentu (sedih, kangen, kesal, tidak bisa tidur, lapar, dll). Ada tiga jenis amplop:

| Jenis | Perilaku | Fungsi untuk retensi |
|---|---|---|
| **Biasa** | Selalu bisa dibuka kapan pun | Alasan membuka web saat butuh |
| **Tersegel** (`bukaMulai`) | Terlihat tapi terkunci sampai tanggal tertentu, dengan hitung mundur | Alasan kembali di tanggal tertentu |
| **Terjadwal** (`muncul`) | Tidak tampil sama sekali sampai tanggalnya, lalu muncul dengan lencana BARU | Konten baru tanpa mengubah kode, cukup upload ulang `index.html` |

Ditambah satu slot khusus **Amplop Tanggal 3** (bulanan, tiap tanggal jadian) yang terbuka setiap tanggal 3 dengan pesan berbeda tiap bulan.

Prinsip nada teks: Arkan **aktif dan proaktif** (mengajak, berjanji, datang duluan), bukan pasif ("Arkan nggak pinter ngomong", "makasih udah sabar sama Arkan"). Tidak lebay.

---

## 2. Perubahan alur layar

### 2.1 Layar `amplop` berada di luar urutan linear
- Tambahkan `<section class="screen" id="amplop">` dan `inits.amplop`.
- **Jangan** masukkan `amplop` ke array `order` (urutan misi) dan **jangan** masukkan ke `STEPS`.
- Di `go()`: pada baris yang mengatur `skipT`, kecualikan `amplop` (tombol SKIP tidak boleh muncul di layar ini). HUD tetap tersembunyi (sudah otomatis karena bukan level/fin).
- Simpan asal masuk di variabel modul `amplopAsal` (`"fin"` atau `"lock"`), dan tombol **KEMBALI** memanggil `go(amplopAsal || "lock")`.
- Parameter uji: `?lompat=amplop` harus bekerja (cek mekanisme `?lompat` yang ada, sesuaikan whitelist kalau ada).

### 2.2 Titik masuk
1. **Finale**: di dalam `#after`, tambahkan kartu baru **"LACI AMPLOP ✉️"** dengan teks ajakan dan tombol `data-go="amplop"` (atau `go('amplop')` dengan `amplopAsal="fin"`). Letakkan setelah kontrak/kue, sebelum penutup. Kartu itu muncul lewat `revealAfter()` seperti bagian lain, bukan di luar alur.
2. **Kunjungan ulang**: saat `finale:start` terpicu, simpan `misi-selesai-2610="1"`. Setiap kali halaman dibuka dan flag ada, layar `lock` menampilkan tombol tambahan **"✉️ LACI AMPLOP"** di bawah tombol utama (tidak mengganti tombol kode yang ada, jadi ia tetap bisa memutar ulang misi). Ini jalur utama saat dia membuka web berbulan-bulan kemudian.
3. Maskot menyapa kunjungan ulang: `maskot.say(CONFIG.amplop.maskot.balik)` sekali per sesi di layar `lock`.

Catatan: jangan mengubah logika kunci kode di `lock`. Baca dulu cara kerjanya, lalu tambahkan tombol tanpa menyentuhnya.

---

## 3. Tampilan

### 3.1 Layar laci
- Header: judul `CONFIG.amplop.judul` (Press Start 2P) + penghitung `3/10 TERBUKA` (VT323).
- Grid **2 kolom** di 360px (gap 12px), tiap sel minimal **44×44px** area ketuk, praktisnya sekitar 150×120px.
- Tiap sel adalah `<button>` berisi: ikon amplop pixel (SVG), label "Buka kalau…" dalam **VT323 ≥ 18px** (Press Start 2P terlalu kecil untuk label panjang), dan emoji mood.
- Di bawah grid: tombol **KEMBALI**, dan baris kecil `kunjungan ke-N`.
- Daftar amplop diurutkan: yang BARU dulu → biasa → tersegel → sudah dibuka yang tidak favorit. Favorit ❤️ diberi sorotan.
- Layar boleh scroll vertikal. Beri `padding-bottom` agar tidak tertutup pet dock dan maskot.

### 3.2 Pixel art amplop (orisinal)
- Grid 16×12. Badan amplop warna `warna` per amplop (hex dari CONFIG), kertas krem `var(--cr)`, outline hitam 1px grid.
- Tiga keadaan (swap string SVG, bukan animasi berat):
  - **Tertutup**: flap turun, ada segel hati kecil `pixelHeart`.
  - **Terbuka**: flap naik, ada surat mengintip, tanda ✓ kecil.
  - **Tersegel-waktu**: gembok pixel + pita menyilang, warna abu-abu, label tanggal.
- Lencana **BARU** kecil di pojok untuk amplop yang belum pernah dibuka dan baru muncul (hanya jika `dibuka[id]` belum ada).
- Jangan menyalin gaya ikon surat/amplop dari emoji atau logo aplikasi mana pun. Gambar sendiri dari `rect`.

### 3.3 Sheet surat (saat amplop diketuk)
Gunakan `role="dialog"` `aria-modal="true"`, z-index 63, scrim gelap di belakang. Urutan animasi (total ≤ 1,4 detik):
1. Amplop membesar ke tengah (transform scale).
2. Flap terbuka (ganti SVG ke keadaan terbuka, atau rotasi sederhana).
3. Surat naik, lalu teks pesan diketik lewat `typeText`. **Ketuk di mana saja untuk langsung menampilkan seluruh teks.**
4. Setelah teks selesai: kotak **"MISI KECIL"** (saran) muncul, lalu tombol.

Tombol di sheet:
- **TUTUP** (selalu ada, juga Esc dan ketuk scrim).
- **BACA LAGI** (hanya jika amplop punya >1 varian pesan: tampilkan varian berikutnya).
- **❤️ SIMPAN** (toggle favorit).
- **PUTAR SUARA 🔊** (hanya jika `suara:true`: memutar pesan suara yang sudah ada di `#after`; baca dulu elemen audionya dan **gunakan kembali**, jangan membuat audio baru).
- **KABARI ARKAN 💬** (hanya jika `CONFIG.amplop.kabariUrl` tidak kosong: tautan biasa `target="_blank" rel="noopener"`. Ini navigasi yang ia tekan sendiri, situs tidak mengirim data apa pun. Default kosong, jadi tombol tersembunyi).

Mode **malam** untuk amplop `tidur`: latar sheet gelap dengan beberapa bintang pixel CSS statis (tanpa canvas baru, tanpa animasi saat reduced-motion).

### 3.4 Amplop tersegel
Tap saat terkunci: amplop bergetar singkat (dilewati saat reduced-motion), bunyi rendah pendek (mute-aware), toast "Masih disegel sampai {tanggal}. Sabar ya, {panggilan} 💗". Di sel tersegel tampil hitung mundur **"X HARI LAGI"**. Kalau tinggal < 1 hari tampil "BESOK" atau "HARI INI JAM {jam}" sesuai data. Hitung mundur hanya diperbarui saat layar terbuka (satu `setInterval` 60 detik, **dibersihkan di `cleanup()`** dan saat pindah layar).
- Saat tanggal tiba, amplop otomatis terbuka segelnya: ganti SVG ke tertutup, lencana BARU, `sfx.coin`, `maskot.say(CONFIG.amplop.maskot.segelTerbuka)`.

---

## 4. Logika data dan waktu

### 4.1 Tanggal
- "Hari ini" = tanggal **lokal** perangkat dalam format `YYYY-MM-DD`. Bandingkan sebagai string, bukan `Date` dengan jam, supaya tidak meleset karena zona waktu.
- Parameter uji `?tanggal=YYYY-MM-DD` menggantikan "hari ini" (hanya untuk pengujian, cukup 1 fungsi `hariIni()`).
- Semua perbandingan lewat satu fungsi, mis. `hariIni()`.

### 4.2 Aturan tampil per amplop
```
tampil  = !a.muncul     || hariIni() >= a.muncul
terkunci = a.bukaMulai  && hariIni() <  a.bukaMulai
```
Amplop `muncul` di masa depan **tidak dirender** (dan tidak dihitung di penghitung X/N).

### 4.3 Pemilihan varian pesan
- Buka pertama kali selalu varian ke-0.
- Selanjutnya: varian yang paling lama tidak ditampilkan (rotasi `idx = (idx+1) % pesan.length`). "BACA LAGI" juga memajukan `idx`.
- Simpan `n` (jumlah buka), `terakhir` (ISO), `idx`.

### 4.4 Amplop Tanggal 3 (bulanan)
- Slot spesial dengan `id: "tanggal3"`, selalu tampil, di barisan atas.
- **Terbuka** jika hari ini tanggal ke-`CONFIG.amplop.bulanan.tanggal` (3). Di hari lain tersegel dan hitung mundur menuju tanggal 3 berikutnya.
- `bulanKe` = jumlah bulan sejak 3 November 2024: `(tahun-2024)*12 + (bulan-11)`. Pada 3 November 2026 bernilai 24.
- Pesan: `pool[(bulanKe-1) % pool.length]`, dengan `{bulanKe}` dan `{tahunKe}` (= `Math.floor(bulanKe/12)`) disubstitusi. Kalau `bulanKe` kelipatan 12, pakai pesan `tahunan` (jika ada) sebagai gantinya.
- Boleh dibuka berkali-kali di hari yang sama. `kunci "dibuka"` mencatat tiap bulan agar BARU tampil lagi bulan berikutnya (simpan `terakhirBulan: "YYYY-MM"`).

### 4.5 Templating
Pakai fungsi templating yang sudah ada untuk `{namaPanggilan}` dan `{pengirim}`. Tambahkan variabel baru lewat `CONFIG.amplop.variabel`:
`{makananFavorit}`, `{bulanKe}`, `{tahunKe}`, `{tanggal}` (tanggal buka dalam format "3 November 2026"). Placeholder yang tidak punya nilai harus **jatuh ke teks netral, jangan tampil mentah** sebagai `{...}`.

---

## 5. Event bus
| Event | Data | Pemicu |
|---|---|---|
| `amplop:open` | `{id, n}` | Sheet dibuka |
| `amplop:segel` | `{id}` | Tap amplop tersegel |
| `amplop:favorit` | `{id, on}` | Toggle simpan |
| `amplop:semua` | `{}` | Semua amplop biasa sudah dibuka minimal sekali (sekali saja) |

Reaksi: maskot menyapa tiap buka (`CONFIG.amplop.maskot.buka`), dan saat `amplop:semua` ada confetti kecil + `maskot.say(semua)`. Reaksi pet opsional, hanya jika modul `pet` sudah punya API aman untuk itu (baca dulu, jangan paksa).

---

## 6. Aksesibilitas
- Tiap amplop adalah `<button>` dengan `aria-label`, mis. "Buka kalau lagi sedih, belum dibuka" / "…, tersegel sampai 3 November 2026".
- Sheet: fokus pindah ke sheet saat dibuka, **focus trap**, Esc menutup, fokus kembali ke amplop pemicu.
- Teks pesan diberi `aria-live="polite"` setelah selesai diketik, atau tampil utuh langsung untuk pembaca layar.
- Kontras: teks pesan hitam di atas kertas krem. Warna amplop hanya aksen.
- `prefers-reduced-motion`: tanpa zoom/getar/bintang bergerak; keadaan akhir langsung tampil; `typeText` ditampilkan utuh (tanpa mengetik).

---

## 7. Pembersihan (`cleanup()`)
Tambahkan penutupan untuk: interval hitung mundur, timeout animasi sheet, interval/timeout `typeText` yang sedang jalan, listener `keydown`/`resize` global yang dipasang saat sheet terbuka, dan tutup sheet (kembalikan `overflow` body). Cek `typeof amplopManager !== 'undefined'` seperti pola `errorDialog`/`pet`.

---

## 8. Skema CONFIG (isi teks sudah ditulis, siap pakai)

Placeholder `{makananFavorit}` dll. diisi dari `variabel`. Edit isi sesuka hati, tapi **pertahankan nada proaktif**.

```js
amplop: {
  aktif: true,
  judul: "LACI AMPLOP",
  subjudul: "Buka kapan pun kamu butuh. Satu amplop, satu pesan dari Arkan.",
  tombolFinale: "BUKA LACI AMPLOP ✉️",
  tombolLock: "✉️ LACI AMPLOP",
  kartuFinale: {
    judul: "LACI AMPLOP ✉️",
    teks: "Misinya selesai, tapi Arkan masih nyimpen beberapa amplop buat kamu. Buka waktu kamu butuh, bukan sekarang semuanya ya. Ada yang baru kebuka di tanggal tertentu."
  },
  kabariUrl: "",                 // contoh: "https://wa.me/62xxxxxxxxxx"; kosong = tombol tidak muncul
  variabel: { makananFavorit: "makanan favoritmu" },   // ← GANTI, mis. "ayam KFC"
  bulanan: {
    tanggal: 3, mulai: "2024-11-03",
    judul: "Buka tiap tanggal 3", warna: "#ff6fae", ikon: "💞",
    pesan: [
      "Hari ini genap {bulanKe} bulan sejak 3 November 2024. Arkan nggak lupa, dan Arkan masih sama bersyukurnya kayak bulan pertama. Bulan ini Arkan mau lebih baik lagi buat kamu.",
      "Bulan ke-{bulanKe}. Pelan-pelan, tapi pasti. Arkan senang bisa jalan bareng kamu. Apa pun yang kamu hadapi bulan ini, Arkan siap bantu dari depan, bukan dari belakang.",
      "Tanggal 3 lagi! Satu bulan lagi udah kita lewati, {namaPanggilan}. Terima kasih udah bikin bulan-bulan Arkan lebih berwarna. Ayo rayain kecil-kecilan, Arkan yang atur.",
      "{bulanKe} bulan bareng kamu. Arkan nggak butuh alasan besar buat bersyukur. Cukup lihat kamu masih di sini. Bulan ini, Arkan janji bakal lebih peka.",
      "Selamat tanggal 3, sayang. Kalau ada satu hal yang Arkan pelajari dari {bulanKe} bulan ini: kamu itu layak diperjuangkan, setiap bulan.",
      "Hei {namaPanggilan}, hari ini istimewa lagi. {bulanKe} bulan! Arkan masih pengin dengar cerita-ceritamu, masih pengin bikin kamu ketawa. Kita lanjut ya."
    ],
    tahunan: "TAHUN KE-{tahunKe}! {bulanKe} bulan, dan Arkan masih memilih kamu, setiap hari. Terima kasih sudah jadi rumah. Tahun berikutnya, Arkan janji lebih baik lagi. Selamat anniversary, {namaPanggilan}. 💗"
  },
  amplop: [
    { id: "sedih", judul: "Buka kalau lagi sedih", ikon: "💙", warna: "#8ecae6",
      pesan: [
        "Hei, {namaPanggilan}. Kalau kamu buka ini, berarti lagi ada yang berat di hati. Kamu nggak harus langsung baik-baik aja, dan nggak harus jelasin apa-apa dulu. Pelan-pelan aja. Kapan pun kamu siap cerita, Arkan dengerin sampai habis. Sedihmu hari ini nggak mengurangi betapa berharganya kamu.",
        "Sedih itu boleh. Nangis juga boleh. Yang jangan cuma satu: nyimpen semuanya sendirian. Kirim aja ke Arkan \"lagi sedih\", nggak perlu kata-kata bagus. Habis itu Arkan yang urus sisanya, entah telepon, datang, atau bawain makanan."
      ],
      misi: "Tarik napas 4 detik, tahan 4 detik, buang pelan 6 detik. Ulangi 3 kali. Terus minum air putih.", suara: false },

    { id: "kangen", judul: "Buka kalau lagi kangen", ikon: "💗", warna: "#ff6fae",
      pesan: [
        "Kangen itu tanda kita punya sesuatu yang berharga. Hari ini kamu kangen, dan percaya deh, Arkan juga. Mungkin lagi sibuk, mungkin lagi jauh, tapi kamu nggak pernah hilang dari pikiran Arkan. Kirim pesan sekarang, nggak usah nunggu ada alasan. Arkan senang banget dikabarin.",
        "Kalau lagi kangen, putar lagi pesan suaranya, atau lihat lagi foto-foto di misi kemarin. Habis itu, Arkan yang telepon duluan ya kalau bisa. Kangenmu pasti dibalas."
      ],
      misi: "Putar pesan suara, lalu kirim satu stiker apa aja ke Arkan.", suara: true },

    { id: "kesal", judul: "Buka kalau lagi kesal", ikon: "🔥", warna: "#ff7b54",
      pesan: [
        "Kalau kamu kesal, entah sama Arkan atau sama dunia, kamu boleh kesal. Tapi sebelum bales chat pakai huruf kapital semua, baca ini dulu ya. Kalau kesalnya gara-gara Arkan, kemungkinan besar Arkan memang salah atau kurang peka. Bilang aja langsung. Arkan janji dengerin tanpa membela diri dulu. Kalau karena hal lain, Arkan siap jadi tempat kamu ngomel sampai puas.",
        "Mode ngambek terdeteksi. 🚨 Misi: tarik napas dulu sebelum bilang \"terserah\". Arkan nggak akan kabur, dan nggak akan ngambek balik. Kita beresin bareng, ya."
      ],
      misi: "Tulis semua kekesalanmu di catatan HP (nggak usah dikirim). Setelah lega, kabari Arkan, kita bahas pelan-pelan.", suara: false },

    { id: "tidur", judul: "Buka kalau nggak bisa tidur", ikon: "🌙", warna: "#6c4ab6", malam: true,
      pesan: [
        "Belum bisa tidur ya? Pikiran yang berisik itu nggak perlu diselesaikan malam ini. Besok masalahnya masih ada, tapi kamu bakal lebih kuat ngadepinnya. Tugasmu malam ini cuma satu: istirahat. Semoga mimpinya indah, dan Arkan ikut jagain dari sini.",
        "Udah jam segini dan kamu masih melek. Coba taruh HP sebentar, redupkan lampu, dan ingat: kamu sudah berbuat cukup hari ini. Besok pagi, Arkan yang ucapin selamat pagi duluan."
      ],
      misi: "Napas 4-7-8 tiga kali. Kalau masih belum bisa, hitung mundur dari 50 pelan-pelan.", suara: false },

    { id: "lapar", judul: "Buka kalau lagi lapar", ikon: "🍗", warna: "#ffd93d",
      pesan: [
        "Perut bunyi? Itu alarm paling jujur. Jangan ditunda ya. Makan dulu, {makananFavorit} kalau bisa. Kapan-kapan Arkan yang traktir, catat aja di daftar utang yang nggak akan pernah ditagih. Habis makan, mood kamu pasti naik 80%.",
        "Sebagai agen yang baik, kamu wajib makan teratur. Ini perintah dari Arkan. 😤 Kalau Arkan ada di sana, udah Arkan beliin dari tadi."
      ],
      misi: "Pesan {makananFavorit} sekarang. Foto dulu, kirim ke Arkan sebagai bukti misi selesai.", suara: false },

    { id: "senang", judul: "Buka kalau lagi senang", ikon: "🌈", warna: "#6ee7b7",
      pesan: [ "Lagi senang? Mantap! Rayain dulu, jangan buru-buru mikirin yang lain. Cerita ke Arkan ya, sekecil apa pun. Kebahagiaanmu itu kebahagiaan Arkan juga." ],
      misi: "Kirim satu kabar bahagiamu ke Arkan, sekarang juga.", suara: false },

    { id: "capek", judul: "Buka kalau lagi capek", ikon: "🔋", warna: "#9bc1ff",
      pesan: [ "Capek? Kamu udah berusaha keras, dan itu kelihatan. Istirahat bukan berarti menyerah. Kasih badanmu waktu. Kalau ada yang bisa Arkan bantu, bilang aja, atau Arkan yang tanya langsung." ],
      misi: "Rebahan 10 menit tanpa HP. Setelah itu minum dan makan sesuatu.", suara: false },

    { id: "nggakcukup", judul: "Buka kalau ngerasa nggak cukup", ikon: "✨", warna: "#ffb3d9",
      pesan: [ "Dengerin baik-baik: kamu cukup. Kamu pintar, kuat, dan lebih berharga dari yang kamu kira. Kalau suara di kepalamu bilang sebaliknya, itu suara yang salah. Kalau kamu minta, Arkan bisa kasih seribu alasan, dan Arkan nggak akan kehabisan." ],
      misi: "Tulis 3 hal yang berhasil kamu lakuin minggu ini, sekecil apa pun.", suara: false },

    { id: "ragu", judul: "Buka kalau ragu Arkan sayang kamu", ikon: "🧸", warna: "#f4a261",
      pesan: [ "Kalau kamu lagi ragu: Arkan sayang kamu. Hari ini, kemarin, dan besok. Kalau Arkan lagi sibuk atau telat bales, itu bukan karena sayangnya berkurang. Tanyain aja langsung, Arkan jawab jujur dan Arkan yang akan memperbaiki kalau ada yang kurang." ],
      misi: "Kirim ke Arkan: \"Hari ini aku butuh diyakinin.\" Arkan bakal langsung jawab.", suara: false },

    { id: "bingung", judul: "Buka kalau lagi bingung", ikon: "🧭", warna: "#a8dadc",
      pesan: [ "Bingung itu wajar. Coba tulis dua pilihannya, terus tanya diri sendiri: yang mana yang bikin lega? Kalau butuh teman mikir, Arkan siap duduk bareng sampai ketemu jawabannya." ],
      misi: "Tulis pro dan kontra tiap pilihan. Habis itu, ceritain ke Arkan.", suara: false },

    /* Tersegel: terlihat, terkunci sampai tanggal */
    { id: "anniv2", judul: "Buka kalau anniversary ke-2", ikon: "💍", warna: "#ffd93d", bukaMulai: "2026-11-03",
      pesan: [ "Dua tahun, {namaPanggilan}. Dua tahun sejak 3 November 2024. Banyak hal berubah, tapi satu hal nggak pernah: Arkan masih memilih kamu, setiap hari. Terima kasih sudah jadi bagian dari dua tahun ini. Tahun ketiga, Arkan janji lebih baik lagi. Selamat anniversary. 💗" ],
      misi: "Ajak Arkan makan malam. Kali ini Arkan yang pesan tempat.", suara: false },

    { id: "ultah22", judul: "Buka kalau kamu umur 22", ikon: "🎂", warna: "#ff6fae", bukaMulai: "2027-10-26",
      pesan: [ "Selamat ulang tahun ke-22, {namaPanggilan}! Setahun lagi sudah terlewati. Arkan bangga lihat kamu tumbuh, dan Arkan masih di sini, masih sayang, dan masih punya banyak rencana buat kita. Semoga tahun ini lebih baik dari yang sebelumnya." ],
      misi: "Tulis satu doa buat dirimu sendiri tahun ini.", suara: false }

    /* Contoh terjadwal (tidak tampil sebelum tanggalnya):
    { id: "natal26", muncul: "2026-12-24", judul: "Buka kalau malam Natal/Tahun Baru", ikon: "🎁", warna: "#6ee7b7", pesan: ["..."], misi: "..." }
    */
  ],
  maskot: {
    balik: "Selamat datang kembali, Agen! Ada amplop baru?",
    buka: ["Wah, surat dari Arkan!", "Dibaca pelan-pelan ya."],
    segelTerbuka: "Ada amplop yang baru bisa dibuka!",
    semua: "Semua amplop sudah kamu buka! Arkan pasti senang."
  },
  teksUI: { tutup: "TUTUP", bacaLagi: "BACA LAGI", simpan: "❤️ SIMPAN", tersimpan: "❤️ TERSIMPAN", putarSuara: "PUTAR SUARA 🔊", kabari: "KABARI ARKAN 💬", baru: "BARU", kembali: "← KEMBALI", misi: "MISI KECIL", hariLagi: "{n} HARI LAGI", besok: "BESOK", hariIni: "HARI INI", disegel: "Masih disegel sampai {tanggal}. Sabar ya, {namaPanggilan} 💗", terbuka: "{n}/{total} TERBUKA" }
}
```

> Catatan untuk agent: `masing-masing "pesan"` boleh berisi satu atau lebih varian. `warna` hanya aksen. Semua string lewat templating.

---

## 9. Kasus tepi (wajib ditangani)
- `localStorage` rusak/penuh/dinonaktifkan: fitur jalan dengan memori sementara; tidak melempar error.
- JSON `amplop-2610` rusak: abaikan dan mulai ulang, jangan crash.
- Perangkat dengan jam dimajukan/dimundurkan: tidak masalah, tampilan hanya mengikuti `hariIni()`. Tidak perlu anti-curang.
- Ketuk cepat berkali-kali pada satu amplop: hanya satu sheet terbuka (guard `sheetAktif`).
- Pindah layar saat sheet terbuka: `cleanup()` menutup sheet dan membersihkan semua timer.
- Pesan sangat panjang: area surat `overflow-y:auto` dengan `max-height` sekitar 55dvh; gunakan `dvh` (bukan `vh`) untuk iOS Safari.
- Layar 320×568: grid tetap 2 kolom, label boleh 2 baris, tidak ada teks terpotong.
- Tidak ada amplop yang tampil (semua `muncul` di masa depan): tampilkan teks ramah dan tombol kembali.
- Pesan suara belum ada atau audio gagal dimuat: tombol PUTAR SUARA disembunyikan, tidak error.
- `amplop:semua` hanya terpicu sekali seumur hidup (simpan flag).

---

## 10. Urutan kerja (jalankan `node --check` setelah tiap tahap)
1. **Pra-syarat 0.2**: pastikan `maskot` terdefinisi, `.btn-yl` ada, `#petPanel` diperbaiki.
2. **CONFIG + penyimpanan**: tambahkan `CONFIG.amplop`, modul `amplopStore` (baca/tulis `amplop-2610`, try/catch, memori cadangan), tambahkan kunci ke `?reset=1`, fungsi `hariIni()` dengan `?tanggal=`.
3. **Layar laci**: HTML `#amplop`, `inits.amplop`, grid, SVG amplop 3 keadaan, urutan tampil, penghitung, tombol kembali; kecualikan dari `order`/skip timer.
4. **Sheet surat**: animasi, `typeText` + ketuk-untuk-lewati, varian pesan, misi kecil, tombol, focus trap, mode malam.
5. **Tersegel + terjadwal + Tanggal 3**: aturan 4.2 dan 4.4, hitung mundur 60 detik, pembukaan segel otomatis.
6. **Titik masuk**: kartu di `#after` (via `revealAfter()`), flag `misi-selesai-2610`, tombol di `lock` untuk kunjungan ulang, sapaan maskot.
7. **Aksesibilitas, reduced-motion, mute, cleanup()**.
8. **Uji penuh** (bagian 11), lalu serahkan.

---

## 11. Uji penerimaan (Playwright Chromium, 360×640 dan 320×568)
Catatan: CDN Tailwind bisa tidak terjangkau di sandbox. Buat salinan uji dengan CSS Tailwind statis, **bukan** untuk dikirim.

1. `?reset=1&lompat=amplop`: grid tampil, 0 error console, **tidak ada scroll horizontal**.
2. Ketuk amplop "sedih": sheet terbuka, teks diketik, ketuk lagi → teks langsung penuh, misi kecil muncul, Esc menutup, fokus kembali ke amplop.
3. Buka amplop yang sama 3 kali: varian berotasi, `n=3`, penghitung "X/N TERBUKA" naik hanya sekali per amplop.
4. ❤️ SIMPAN lalu muat ulang: favorit bertahan, urutan menaruhnya di atas.
5. Tanggal asli hari ini: `anniv2` tersegel dengan hitung mundur benar. `?tanggal=2026-11-03`: terbuka, lencana BARU, `tanggal3` terbuka dengan `bulanKe=24` dan memakai pesan tahunan.
6. `?tanggal=2026-11-04`: `tanggal3` tersegel, hitung mundur "29 HARI LAGI" (menuju 3 Des).
7. Ubah satu amplop jadi `muncul: "2099-01-01"`: tidak dirender dan tidak dihitung.
8. Emulasi `prefers-reduced-motion: reduce`: tanpa animasi, teks langsung penuh.
9. Mute aktif: tidak ada bunyi.
10. Simulasikan `localStorage` melempar error: fitur tetap jalan.
11. Pindah layar saat sheet terbuka dan saat hitung mundur jalan: tidak ada timer yatim (`cleanup()` bersih), tidak ada error.
12. Alur penuh: `?reset=1`, selesaikan sampai finale → kartu LACI AMPLOP muncul → buka → kembali ke finale. Lalu muat ulang: tombol "✉️ LACI AMPLOP" muncul di `lock`, tombol kode lama tetap berfungsi.
13. Fitur lama tidak berubah: kuis, gacha, boss, kontrak, kue, pet, dialog error, pemindai tetap jalan.

---

## 12. Di luar lingkup
Tidak ada backend, notifikasi push, akun, atau pengiriman data keluar. Konten baru ditambah dengan mengedit `CONFIG.amplop.amplop` (pakai `muncul` untuk menjadwalkan) lalu meng-upload ulang `index.html`.

---

## 13. Bagian CONFIG yang perlu kamu sesuaikan
- `variabel.makananFavorit` — makanan favorit Zahwa.
- `kabariUrl` — tautan WhatsApp (opsional). Kosongkan jika tidak mau tombolnya.
- Isi `pesan` tiap amplop — sentuhan pribadi yang spesifik (satu kenangan nyata) jauh lebih menyentuh dari teks umum, terutama untuk `kangen`, `kesal`, `anniv2`.
- `misi` tiap amplop — pastikan Arkan benar-benar sanggup memenuhinya (mis. "Arkan yang pesan tempat").
- `bukaMulai` dan `muncul` — jadwalkan konten baru untuk tanggal penting (anniversary, Natal/Tahun Baru, awal semester).
- `bulanan.pool` — tambah sampai 12 pesan supaya tidak berulang tiap tahun.

---

## 14. Prompt siap tempel untuk agent

> Kamu akan memodifikasi file `index.html` yang sudah ada (web ulang tahun bertema 8-bit + misi rahasia, Tailwind CDN + JS vanilla, satu file). Jangan menulis ulang dari nol dan jangan mengubah perilaku fitur yang sudah ada. Baca seluruh file dulu, terutama `CONFIG`, `go()`, `order`, `inits`, `levelClear()`, `cleanup()`, `revealAfter()`, logika layar `lock`, pemutar pesan suara di `#after`, modul `maskot`/`pet`/`kontrak`/`kueManager`, dan helper `$`, `$$`, `fill`, `rand`, `beep`, `sfx`, `confetti`, `typeText`, `pixelHeart`, `toast`, serta fungsi templating `{namaPanggilan}`/`{pengirim}`. Tambahkan fitur "Amplop Buka Kalau…" sesuai dokumen `SPEC-amplop-buka-kalau.md` yang terlampir. Ikuti persis bagian 0 (aturan umum dan cek pra-syarat), alur layar (bagian 2), skema `CONFIG` dan isi teks yang sudah ditulis (bagian 8), dan urutan kerja (bagian 10). Layar `amplop` TIDAK boleh masuk ke `order`/`STEPS` dan TIDAK boleh memunculkan tombol SKIP. Aturan keras: mobile-first (360×640, juga 320×568), tanpa library atau file gambar/audio baru (pixel art lewat SVG/CSS/Canvas), semua teks Indonesia dan lewat `CONFIG`, hormati `prefers-reduced-motion` dan tombol mute, tidak ada game over atau jalan buntu, `localStorage` selalu dalam `try/catch`, semua timer dibersihkan di `cleanup()`, tidak ada kamera atau data yang dikirim keluar, semua desain orisinal. Nada teks: Arkan proaktif dan hangat, bukan pasif atau lebay. Setelah tiap tahap, ekstrak skrip dan jalankan `node --check`, lalu pastikan tidak ada error di console, dan jalankan uji penerimaan bagian 11. Output: `index.html` yang sudah diperbarui, ringkasan singkat perubahan, dan daftar bagian `CONFIG` yang perlu disesuaikan pengguna.
