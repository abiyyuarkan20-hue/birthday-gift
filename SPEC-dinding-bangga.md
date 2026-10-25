# SPEC — Fitur "Dinding Bangga" 🏅

Dokumen ini untuk mengarahkan agent AI yang memodifikasi `index.html` (web ulang tahun Zahwa Davina, tema 8-bit + misi rahasia "Operasi 26/10"). Pengirim: Abiyyu, yang menyebut dirinya **Arkan** di semua teks. Tujuan fitur: membuat Zahwa merasa **dilihat**, bukan hanya disayang. Sebuah dinding pajangan berisi medali pixel, masing-masing mewakili satu hal yang Arkan bangga dari Zahwa selama setahun terakhir (usahanya, keberaniannya, sifat baiknya, hal-hal kecil yang Arkan perhatikan).

Fitur ini **bagian dari finale** (bagian `#after`), bukan layar baru. Itu sengaja, supaya risikonya kecil dan bisa selesai cepat.

---

## 0. Konteks dan aturan umum

### 0.1 Aturan keras
- Jangan menulis ulang dari nol. Jangan mengubah perilaku fitur yang sudah ada (kuis, gacha, boss, kontrak, kue, pet, dialog error, pemindai, amplop, taman bila sudah ada).
- Satu file `index.html`, Tailwind CDN + JS vanilla. Tanpa library baru, tanpa file gambar/audio baru. Semua pixel art lewat SVG (`rect`, `shape-rendering="crispEdges"`), CSS, atau Canvas.
- Mobile-first 360×640, tetap rapi di 320×568, tanpa scroll horizontal.
- Semua teks Indonesia dan semua lewat `CONFIG.dindingBangga`. Tidak ada teks hardcode di HTML/JS selain label teknis.
- Hormati `prefers-reduced-motion` dan tombol mute (`sfxOn`). Semua bunyi lewat `beep()`/`sfx`.
- Tidak ada game over dan tidak ada jalan buntu. **Semua medali bisa dibuka sejak awal**, tidak ada yang terkunci. Satu-satunya yang "terkunci" adalah Piala Utama, dan itu hanya hadiah tambahan.
- `localStorage` selalu dalam `try/catch`; fitur tetap jalan dengan memori sementara bila storage gagal.
- Semua timer/interval/listener global dibersihkan di `cleanup()`.
- Tidak ada kamera, tidak ada data dikirim keluar.
- Desain orisinal: jangan meniru ikon, logo, atau medali dari game atau aplikasi mana pun.
- Setelah tiap tahap, ekstrak skrip dan jalankan `node --check`, lalu pastikan tidak ada error di console.

### 0.2 Cek awal wajib
Baca seluruh file dulu: `CONFIG`, `go()`, `inits.fin`, `revealAfter()`, `cleanup()`, struktur `#after` (kontrak, kue, dan kartu amplop/taman bila ada), modul `kontrak`, `kueManager`, `maskot`, `starRain`, helper `$`, `$$`, `fill`, `rand`, `beep`, `sfx`, `confetti`, `typeText`, `toast`, `emit`, `on`, dan daftar kunci di `?reset=1`.

Pastikan `maskot` terdefinisi dan `.btn-yl` ada di CSS. Jika `hariIni()` sudah ada (dari fitur amplop), pakai untuk fitur `muncul` (lihat 4.4); jika belum, definisikan dengan perilaku sama (tanggal lokal `YYYY-MM-DD`, bisa ditimpa `?tanggal=`).

### 0.3 Peta z-index
HUD 40 · pet dock 44 · maskot 45 · scanline 55 · hujan bintang 59 · confetti 60 · sheet amplop 63 · dexOverlay 65 · pet panel 66 · dialogScrim 67 · os-win 68 · toast 70 · dexBanner 80.
Dinding Bangga **tidak memakai overlay atau modal**. Rincian medali tampil inline di bawah dinding, jadi tidak perlu z-index khusus.

### 0.4 Kunci localStorage
| Kunci | Isi |
|---|---|
| `dinding-bangga-2610` | `{v:1, dilihat:[id], piala:false, pertama:ISO}` |

`?reset=1` yang sudah ada harus ikut menghapus kunci ini.

### 0.5 Prioritas (kalau waktu mepet)
- **P0 (wajib):** dinding + medali + panel rincian + pelacak "dilihat" + penyimpanan + cleanup + uji dasar.
- **P1:** Piala Utama dengan perayaan, tombol sebelumnya/berikutnya, maskot bereaksi, efek ayunan dan kilau.
- **P2 (boleh dilewati):** `muncul` (medali terjadwal), kartu "Simpan sebagai PNG".

---

## 1. Konsep dan perilaku

- Di finale, setelah bagian kue dan sebelum kartu laci amplop (`#ampKartu`; jika tidak ada, sebelum judul `▸ BLOOPERS`), muncul bagian baru **"DINDING BANGGA"**.
- Isinya satu dinding pajangan pixel dengan **sekitar 9 medali** tergantung di pita. Disarankan 9 sampai 12; maksimal 21.
- Tiap medali = satu hal yang Arkan bangga dari Zahwa. Mengetuk medali membuka rincian: judul, kategori, penjelasan Arkan yang diketik per huruf, dan tanda tangan "— Arkan".
- Pelacak: "3/9 DILIHAT". Medali yang belum pernah dibuka berkilau, yang sudah dibuka mendapat tanda ✓.
- Setelah **semua medali dilihat**, **Piala Utama** terbuka dengan perayaan kecil dan pesan penutup dari Arkan.
- Nada teks: Arkan **aktif, hangat, tulus**, bukan pasif dan bukan lebay. Hindari kalimat yang menyiratkan Arkan merepotkan atau selalu dimaafkan. Fokus pada apa yang Arkan **lihat** dan **kagumi** dari Zahwa.

---

## 2. Perubahan alur

- Tidak ada layar baru, tidak ada perubahan pada `order`, `STEPS`, atau tombol SKIP.
- Tambahkan bagian HTML di `#after`, dan panggil `dindingBangga.init()` di `inits.fin` (di samping `kontrak.init(); kueManager.init();`).
- **`init()` harus idempoten**: `inits.fin` berjalan tiap kali masuk finale. Pasang pendengar hanya sekali (pakai penanda), tetapi `render()` boleh dijalankan ulang tiap kali.
- Bagian ini tampil lewat `revealAfter()` seperti bagian lain di `#after` (tidak perlu logika tambahan).

---

## 3. Tampilan

### 3.1 Susunan bagian
1. Judul `▸ DINDING BANGGA` (gaya `chip` seperti judul lain) dan subjudul.
2. Chip periode (`26 OKT 2025 – 26 OKT 2026`) dan penghitung `3/9 DILIHAT`.
3. **Dinding** (`#dbDinding`).
4. **Panel rincian** (`#dbDetail`) di bawah dinding.
5. **Piala Utama** (`#dbPiala`) di bawah panel.

### 3.2 Dinding
- Bingkai `.frame`. Latar dinding pixel dengan CSS saja: papan kayu atau bata dari `repeating-linear-gradient`, warna gelap keunguan senada tema (misalnya dasar `#3b2a5c` dengan garis `rgba(0,0,0,.25)`). Tanpa gambar.
- Grid **3 kolom** (`repeat(3, minmax(0,1fr))`, gap sekitar 8px). Di lebar 320px tiap sel sekitar 90px, tetap ≥ 44px untuk area ketuk.
- Tiap baris medali diberi **rak** (garis papan tebal di bawahnya) agar terasa seperti pajangan. Di bagian atas tiap medali ada **paku** pixel kecil tempat pita digantung.
- Tiap medali adalah `<button>` berisi: SVG medali, dan label judul pendek di bawahnya (VT323 **≥ 18px**, maksimal 2 baris). Jumlah medali tidak harus kelipatan 3.
- **Ayunan**: medali bergoyang pelan (rotasi ±2°, `steps(4)`, penundaan berbeda per medali). Tidak ada saat reduced-motion.
- **Belum dilihat**: titik kilau (✨ atau bintang pixel) berkedip di pojok medali. Saat reduced-motion, kilau statis. **Sudah dilihat**: kilau hilang, muncul tanda ✓ kecil. Status tidak boleh hanya lewat warna; selalu ada penanda bentuk.

### 3.3 Sprite medali (SVG orisinal)
`viewBox="0 0 16 20"`, ukuran tampil sekitar 56×70px, dibuat dari `rect`.
- **Pita** (y 0–7): dua ekor, kiri `x4..6` warna pita 1 dan kanan `x9..11` warna pita 2, dengan takik V di bawah.
- **Cakram** (y 8–18) dengan lebar baris `[4, 6, 8, 10, 10, 10, 10, 10, 8, 6, 4]`, tiap baris terpusat di x=8 (`x = 8 - lebar/2`). Gambar dulu versi hitam (outline) yang lebih besar satu piksel ke semua arah (y 7–19, lebar +2), lalu cakram berwarna di atasnya.
- **Kilap**: dua piksel putih di kiri atas cakram.
- **Ikon**: emoji dari `CONFIG` ditaruh sebagai teks, ditumpuk di tengah cakram (sekitar piksel (8, 13.5)).
- Warna cakram dan pita ditentukan oleh **kategori** (lihat skema).
- SVG `aria-hidden="true"`; informasi untuk pembaca layar ada di `aria-label` tombol.

### 3.4 Panel rincian (inline, tanpa modal)
- Wadah `#dbDetail`, tersembunyi sampai ada medali yang dibuka. Bingkai `.frame` atau kertas krem `.paper`.
- Isi: medali besar (sekitar 96px), chip kategori, judul (VT323 besar), teks penjelasan, baris "— Arkan", dan nomor `MEDALI 3 DARI 9`.
- Teks penjelasan diketik lewat `typeText` (pola sama dengan sheet surat amplop): ketuk di mana saja di panel untuk langsung menampilkan semuanya. Saat reduced-motion, langsung tampil penuh.
- Untuk pembaca layar: teks utuh diisi sekaligus ke wilayah `sr-only` dengan `aria-live="polite"`, sedangkan teks yang sedang diketik diberi `aria-hidden="true"` sampai selesai.
- Tombol: `◀ SEBELUMNYA`, `BERIKUTNYA ▶` (bergulir melingkar di antara medali yang tampil dan menandainya "dilihat"), dan `TUTUP`.
- Saat dibuka, panel ber-`scrollIntoView({block: "nearest"})` (halus, kecuali reduced-motion). Medali yang sedang dibuka diberi kontur kuning di dinding.
- Bila ada `kisah` (cerita spesifik, lihat skema), tampilkan sebagai paragraf kedua dengan label kecil `teksUI.labelKisah` ("YANG ARKAN INGAT").

### 3.5 Piala Utama
- Sprite piala pixel emas orisinal (cangkir + dua gagang + dasar), `viewBox` sekitar 16×16.
- **Terkunci**: siluet abu gelap dengan "?" dan teks `teksUI.pialaTerkunci` ("Lihat semua medali untuk membuka piala, {n} lagi").
- **Terbuka**: piala berwarna emas dengan kilau, judul `piala.judul`, dan pesan penutup `piala.pesan` (ditampilkan penuh, tidak perlu diketik).
- Perayaan saat terbuka (sekali): setelah teks medali terakhir selesai diketik, tunda ±800ms, lalu `sfx.win()`, `confetti(80)`, `starRain.burst({jumlah: 60})` bila tersedia, `toast`, `maskot.say`, dan gulir ke piala. Saat reduced-motion, tanpa gulir halus dan tanpa efek bergerak, tetapi teks tetap tampil.
- Piala yang sudah pernah terbuka tetap terbuka saat finale diulang, tanpa perayaan ulang.

### 3.6 Kartu PNG (P2, opsional)
Tombol `SIMPAN PIALA 📥` yang membuat gambar PNG lewat Canvas 2D (judul piala, nama Zahwa, daftar judul medali) dengan pola yang sama seperti `simpanKartuLokal`. Tanpa data keluar. Lewati jika waktu mepet.

---

## 4. Logika

### 4.1 Penyimpanan
`dbStore` (pola sama dengan `ampStore`): `load()` dengan validasi (`v===1`, `dilihat` berupa larik) dan jatuh ke state baru bila rusak; `save()` dalam `try/catch`; memori cadangan bila storage gagal.

### 4.2 Membuka medali (`buka(id)`)
1. Cari medali di daftar tampil. Jika tidak ada, abaikan.
2. Tambahkan `id` ke `dilihat` bila belum ada, simpan, perbarui tanda ✓ dan penghitung, dan emit `bangga:lihat {id, n, total}`.
3. Tampilkan panel dan ketik teks (dengan guard `tok` agar ketikan lama tidak menimpa).
4. `sfx.coin()` dan `maskot.say` (acak dari `maskot.lihat`).
5. Jika semua medali tampil sudah ada di `dilihat` dan `piala` belum terbuka: set `piala=true`, simpan, emit `bangga:semua {}` (sekali saja), lalu jadwalkan perayaan (3.5) setelah ketikan selesai.

### 4.3 Sebelumnya/Berikutnya
Bergulir melingkar di daftar tampil menurut urutan di `CONFIG`. Memanggil `buka()` yang sama, jadi penandaan "dilihat" dan penyimpanan konsisten.

### 4.4 Medali terjadwal (P2)
Medali boleh memiliki `muncul: "YYYY-MM-DD"`. Medali dengan `muncul` di masa depan **tidak dirender dan tidak dihitung** di penghitung. Setelah tanggalnya tiba, ia tampil dengan kilau BARU. Piala yang sudah terbuka tidak ikut terkunci lagi. Perbandingan tanggal lewat `hariIni()`.

### 4.5 Templating
Pakai `fill()` untuk `{namaPanggilan}` dan `{namaAsli}`. Placeholder lain tanpa nilai harus jatuh ke teks kosong, **jangan tampil mentah** sebagai `{…}` (buat `dbFill` kecil bila perlu). Teks default sengaja ditulis tanpa placeholder agar aman dipakai apa adanya.

### 4.6 Peristiwa
| Event | Data | Pemicu |
|---|---|---|
| `bangga:lihat` | `{id, n, total}` | Medali dibuka |
| `bangga:semua` | `{}` | Semua medali sudah dilihat (sekali) |

Jangan mengubah event yang sudah ada. Boleh menambah pendengar baru.

---

## 5. Aksesibilitas
- Tiap medali adalah `<button>` dengan `aria-label`, misalnya "Medali Pejuang Kuliah, kategori Usaha, belum dilihat". Setelah dibuka: "…, sudah dilihat".
- Wadah dinding `role="group"` dengan `aria-label` dari `CONFIG`. Panel rincian `role="region"`, dengan wilayah `aria-live="polite"` untuk teks utuh.
- Fokus terlihat (outline kuning, sama seperti fitur amplop). Fokus **tidak dipindahkan** otomatis ke panel; fokus tetap di medali yang ditekan.
- Kontras: teks panel hitam di atas krem, atau krem di atas ungu gelap; ukuran ≥ 18px VT323. Warna kategori hanya aksen, kategori selalu disebut dengan kata.
- `prefers-reduced-motion`: lihat 3.2, 3.4, 3.5.

---

## 6. Pembersihan (`cleanup()`)
Tambahkan `if (typeof dindingBangga !== 'undefined') dindingBangga.cleanup();` di `cleanup()` (pola yang sama dengan `amplopManager`). Hentikan: ketikan `typeText` yang sedang jalan (picu `pointerdown` sintetis pada elemen teks, seperti pola amplop), timeout perayaan piala, dan timeout animasi. Tidak ada interval global.

---

## 7. Skema CONFIG (isi teks siap pakai)

Teks default **sengaja umum dan hangat** agar aman jika tidak sempat diganti, tetapi paling bagus kalau kamu mengisi `kisah` dengan cerita nyata (lihat bagian 11).

```js
dindingBangga: {
  aktif: true,
  judul: "DINDING BANGGA",
  subjudul: "Hal-hal yang Arkan bangga dari kamu setahun ini. Ketuk tiap medali.",
  periode: "26 OKT 2025 – 26 OKT 2026",
  kategori: {
    usaha:      { label: "USAHA",              disc: "#ffd93d", pita1: "#ff6fae", pita2: "#b79ced" },
    berani:     { label: "KEBERANIAN",         disc: "#ff7b54", pita1: "#ffd93d", pita2: "#fff4e0" },
    hati:       { label: "SIFAT BAIK",         disc: "#ff6fae", pita1: "#6ee7b7", pita2: "#b79ced" },
    kecil:      { label: "KECIL TAPI BERARTI", disc: "#6ee7b7", pita1: "#ffd93d", pita2: "#ff6fae" }
  },
  medali: [
    { id: "kuliah", kategori: "usaha", ikon: "📚", judul: "Pejuang Kuliah",
      teks: "Arkan lihat kamu belajar, begadang, dan tetap lanjut walau capek. Nggak semua orang sanggup menjalani proses kayak kamu. Arkan bangga sama setiap usaha, termasuk yang nggak kelihatan orang lain.",
      kisah: "" },   // ← ISI dengan satu cerita nyata, mis. momen tugas/ujian yang Arkan ingat
    { id: "mencoba", kategori: "berani", ikon: "🚀", judul: "Berani Mencoba",
      teks: "Kamu berani mencoba hal baru walau takut salah. Keberanian itu bukan nggak takut, tapi tetap melangkah. Arkan bangga lihat kamu melangkah, dan Arkan ada di sisimu tiap langkahnya.",
      kisah: "" },
    { id: "hatibaik", kategori: "hati", ikon: "💗", judul: "Hati yang Baik",
      teks: "Cara kamu peduli sama orang lain, bahkan di hari kamu sendiri lagi capek. Arkan perhatiin, dan itu salah satu hal yang paling Arkan kagumi dari kamu.",
      kisah: "" },
    { id: "bangkit", kategori: "berani", ikon: "🌱", judul: "Bangkit Lagi",
      teks: "Waktu hari lagi berat, kamu tetap bangkit pelan-pelan. Nggak harus cepat. Arkan bangga kamu nggak menyerah, dan Arkan mau jadi orang yang bikin bangkitnya terasa lebih ringan.",
      kisah: "" },
    { id: "konsisten", kategori: "usaha", ikon: "⭐", judul: "Rajin dan Konsisten",
      teks: "Kamu konsisten mengerjakan hal yang penting buat kamu, bahkan saat nggak ada yang menyuruh. Arkan lihat, dan Arkan salut.",
      kisah: "" },
    { id: "sabar", kategori: "hati", ikon: "🕊️", judul: "Sabar dan Tenang",
      teks: "Kamu sabar menghadapi banyak hal. Arkan kagum, dan Arkan mau jadi orang yang bikin kamu nggak perlu sabar sendirian.",
      kisah: "" },
    { id: "halkecil", kategori: "kecil", ikon: "🌼", judul: "Ingat Hal Kecil",
      teks: "Kamu sering ingat hal-hal kecil, dan itu bikin orang di sekitarmu merasa dianggap. Arkan merasakannya, dan Arkan senang diperhatikan sama kamu.",
      kisah: "" },
    { id: "tawa", kategori: "kecil", ikon: "😆", judul: "Tawa yang Menular",
      teks: "Tawa kamu bikin suasana jadi ringan. Arkan senang bisa jadi alasan kamu ketawa, dan Arkan bakal terus cari cara bikin kamu ketawa.",
      kisah: "" },
    { id: "apaadanya", kategori: "hati", ikon: "👑", judul: "Jadi Diri Sendiri",
      teks: "Kamu apa adanya sudah cukup. Arkan bangga kamu berani jadi diri sendiri, di depan siapa pun.",
      kisah: "" }
    /* Medali terjadwal (opsional, P2):
    { id: "bonus1", kategori: "hati", ikon: "🎁", judul: "Medali Bonus", muncul: "2026-12-03", teks: "...", kisah: "" } */
  ],
  piala: {
    judul: "PIALA UTAMA: {namaAsli}",
    pesan: "Medali-medali ini kelihatan kecil kalau dihitung satu per satu. Tapi kalau dijumlahkan, itu kamu. Arkan bangga banget sama Zahwa yang sekarang, dan Arkan nggak sabar lihat Zahwa yang akan datang. Selamat ulang tahun ke-21. Arkan selalu ada di barisan paling depan yang menyemangati kamu. 💗"
  },
  teksUI: {
    penghitung: "{n}/{total} DILIHAT",
    tutup: "TUTUP", sebelumnya: "◀ SEBELUMNYA", berikutnya: "BERIKUTNYA ▶",
    nomor: "MEDALI {n} DARI {total}",
    labelKisah: "YANG ARKAN INGAT",
    ttd: "— Arkan",
    pialaTerkunci: "Lihat semua medali untuk membuka piala ({n} lagi).",
    pialaTerbuka: "PIALA TERBUKA! 🏆",
    toastPiala: "Semua medali sudah dilihat. Piala utama terbuka! 🏆",
    simpanPiala: "SIMPAN PIALA 📥",
    dindingLabel: "Dinding medali kebanggaan",
    ariaBelum: "belum dilihat", ariaSudah: "sudah dilihat",
    ariaMedali: "Medali {judul}, kategori {kategori}, {status}"
  },
  maskot: {
    lihat: ["Wah, medali yang bagus!", "Arkan beneran bangga lho.", "Baca pelan-pelan ya."],
    piala: "Semua medali terbuka! Kamu hebat banget!"
  }
}
```

> Catatan untuk agent: `kategori` pada tiap medali merujuk ke kunci di `CONFIG.dindingBangga.kategori`; bila kunci tidak ditemukan, jatuh ke kategori pertama. Jika `kisah` kosong, jangan render label dan paragrafnya. Semua string melewati `fill()`/`dbFill`.

---

## 8. Kasus tepi (wajib ditangani)
- `localStorage` rusak/penuh/dinonaktifkan atau JSON rusak: mulai dari state baru di memori; tidak crash.
- `medali` kosong atau semuanya `muncul` di masa depan: sembunyikan seluruh bagian (judul dan dinding) tanpa error.
- Jumlah medali bukan kelipatan 3, atau hanya 1 medali: tampilan tetap rapi; tombol sebelumnya/berikutnya tersembunyi bila hanya satu.
- Ketuk medali berkali-kali dengan cepat: hanya satu ketikan berjalan (guard `tok`), tidak ada teks tumpang tindih.
- Pindah layar atau finale diulang saat mengetik: `cleanup()` menghentikan ketikan, tidak ada timer yatim, dan `init()` idempoten (tidak menggandakan pendengar).
- Medali ditambah setelah piala terbuka (lewat `muncul`): piala tetap terbuka, medali baru bertanda BARU, penghitung memakai total terbaru.
- Layar 320×568: tidak ada scroll horizontal, label tidak terpotong (maksimal 2 baris), area ketuk ≥ 44px.
- `aktif: false`: sembunyikan seluruh bagian.
- Emoji tidak terender di perangkat tertentu: tombol tetap terbaca lewat judul dan `aria-label`.

---

## 9. Urutan kerja (jalankan `node --check` setelah tiap tahap)
1. **Pra-syarat 0.2** dan garis dasar: jalankan uji fitur yang sudah ada.
2. **CONFIG + penyimpanan** (P0): `CONFIG.dindingBangga`, `dbStore`, kunci ke `?reset=1`.
3. **Dinding dan sprite medali** (P0): HTML bagian di `#after`, `render()`, sprite SVG, grid 3 kolom, rak, paku, tanda dilihat.
4. **Panel rincian** (P0): `buka()`, `typeText` dengan ketuk-untuk-lewati, `aria-live`, penghitung, penyimpanan.
5. **Sebelumnya/Berikutnya, ayunan, kilau, maskot** (P1).
6. **Piala Utama dan perayaan** (P1), event `bangga:semua`.
7. **Aksesibilitas, reduced-motion, mute, `cleanup()`**, panggilan di `inits.fin`.
8. **P2** bila waktu ada: `muncul`, kartu PNG.
9. **Uji penuh** (bagian 10), lalu serahkan.

---

## 10. Uji penerimaan (Playwright Chromium, 360×640 dan 320×568)
Catatan: CDN Tailwind bisa tidak terjangkau di sandbox. Buat salinan uji dengan CSS Tailwind statis, **bukan** untuk dikirim.

1. `?reset=1&lompat=fin`, lalu panggil `revealAfter()`: bagian Dinding Bangga tampil dengan **9 medali**, tanpa scroll horizontal, tanpa error. Penghitung `0/9 DILIHAT`, piala terkunci dengan teks "9 lagi".
2. Ketuk medali pertama: panel muncul, teks mulai diketik; ketuk di panel → teks langsung penuh, tanda ✓ muncul, penghitung `1/9`. Teks panel tidak mengandung `{`.
3. Ketuk medali yang sama berkali-kali: penghitung tetap `1/9`, tidak ada ketikan ganda.
4. Tombol `BERIKUTNYA ▶` dan `◀ SEBELUMNYA` bergulir melingkar dan menandai medali; setelah medali terakhir, berikutnya kembali ke yang pertama.
5. Buka semua 9 medali: piala terbuka (setelah ketikan terakhir selesai), event `bangga:semua` terpicu **tepat sekali**, pesan piala tampil, confetti berjalan.
6. Muat ulang (tanpa `?reset=1`): `dilihat` bertahan (`9/9`), piala tetap terbuka tanpa perayaan ulang. Dengan `?reset=1`: kembali ke `0/9`.
7. Emulasi `prefers-reduced-motion: reduce`: tanpa ayunan/gulir halus/efek bergerak; teks panel langsung penuh; fungsi tetap benar.
8. Mute aktif: tidak ada bunyi (`beep` melewati `sfxOn`).
9. Simulasikan `localStorage` melempar error: fitur tetap jalan (memori); tanpa error.
10. Ulangi finale (pindah ke `lock`, kembali ke `fin`) dan pindah layar saat teks diketik: tidak ada timer yatim, pendengar tidak ganda (satu ketukan → satu ketikan), tanpa error.
11. Keyboard: Tab menjangkau semua medali, Enter/Spasi membuka, `aria-label` benar ("…, sudah dilihat" setelah dibuka).
12. (P2) Medali dengan `muncul: "2099-01-01"` tidak dirender dan tidak dihitung; dengan tanggal lampau tampil BARU.
13. Fitur lama tidak berubah: kuis, gacha, boss, kontrak, kue, pet, dialog error, pemindai, serta amplop dan taman bila sudah ada.

---

## 11. Bagian CONFIG yang perlu kamu isi sendiri (ini yang membuatnya menyentuh)
- **`kisah` di setiap medali**: satu cerita nyata yang spesifik, 1 sampai 2 kalimat, misalnya momen tertentu waktu dia belajar, berani mencoba sesuatu, atau bersikap baik. Ini bagian terpenting. Pujian spesifik jauh lebih berkesan daripada pujian umum.
- **`judul` dan `ikon` medali**: ganti dengan nama yang hanya kalian berdua yang paham, kalau ada.
- **Jumlah medali**: tambah atau kurangi sesuai cerita nyata (9 sampai 12 ideal; maksimal 21).
- **`piala.pesan`**: tambahkan satu kalimat penutup yang paling tulus dari kamu.
- **`periode`**: sesuaikan bila ingin rentang waktu lain.

Aturan nada saat mengedit: tulis apa yang Arkan **lihat dan kagumi** dari Zahwa. Hindari kalimat yang membuat Arkan terdengar pasif atau merepotkan.

---

## 12. Prompt siap tempel untuk agent

> Kamu akan memodifikasi file `index.html` yang sudah ada (web ulang tahun bertema 8-bit + misi rahasia, Tailwind CDN + JS vanilla, satu file). Jangan menulis ulang dari nol dan jangan mengubah perilaku fitur yang sudah ada. Baca seluruh file dulu, terutama `CONFIG`, `go()`, `inits.fin`, `revealAfter()`, `cleanup()`, struktur `#after`, modul `kontrak`/`kueManager`/`maskot`/`starRain`, daftar kunci di `?reset=1`, dan helper `$`, `$$`, `fill`, `rand`, `beep`, `sfx`, `confetti`, `typeText`, `toast`, `emit`, `on`. Tambahkan fitur "Dinding Bangga" sesuai dokumen `SPEC-dinding-bangga.md` yang terlampir, sebagai bagian di dalam `#after` (BUKAN layar baru), ditaruh setelah bagian kue dan sebelum kartu amplop/judul BLOOPERS. Ikuti persis bagian 0 (aturan umum dan cek pra-syarat), perilaku dan tampilan (bagian 1 sampai 3), logika (bagian 4), skema `CONFIG` dan isi teks yang sudah ditulis (bagian 7), dan urutan kerja (bagian 9, kerjakan P0 lebih dulu, lalu P1, P2 hanya jika waktu ada). `init()` harus idempoten karena `inits.fin` berjalan tiap masuk finale. Semua medali harus bisa dibuka sejak awal; tidak ada jalan buntu. Aturan keras: mobile-first (360×640, juga 320×568), tanpa library atau file gambar/audio baru (pixel art lewat SVG/CSS/Canvas), semua teks Indonesia dan lewat `CONFIG`, hormati `prefers-reduced-motion` dan tombol mute, `localStorage` selalu dalam `try/catch`, semua timer dibersihkan di `cleanup()`, tidak ada kamera atau data yang dikirim keluar, semua desain orisinal. Nada teks: Arkan proaktif, hangat, dan tulus, bukan pasif atau lebay. Setelah tiap tahap, ekstrak skrip dan jalankan `node --check`, lalu pastikan tidak ada error di console, dan jalankan uji penerimaan bagian 10. Output: `index.html` yang sudah diperbarui, ringkasan singkat perubahan, dan daftar bagian `CONFIG` yang perlu disesuaikan pengguna.
