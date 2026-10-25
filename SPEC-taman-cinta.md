# SPEC — Fitur "Taman Cinta" 🌻

Dokumen ini untuk mengarahkan agent AI yang memodifikasi `index.html` (web ulang tahun Zahwa Davina, tema 8-bit + misi rahasia "Operasi 26/10"). Pengirim: Abiyyu, yang menyebut dirinya **Arkan** di semua teks. Tujuan fitur: memberi alasan **dibuka setiap hari** setelah 26 Oktober 2026, lewat kebun pixel yang tumbuh seiring bulan jadian dan harus disiram sekali sehari, **tanpa pernah menghukum** kalau dia lupa.

Fitur ini dibangun di atas versi `index.html` yang sudah memuat fitur **Amplop "Buka kalau…"** (`amplopManager`, `ampStore`, `hariIni()`, dst.). Pola kodenya sengaja mengikuti fitur itu.

---

## 0. Konteks dan aturan umum

### 0.1 Aturan keras
- Jangan menulis ulang dari nol. Jangan mengubah perilaku fitur yang sudah ada (kuis, gacha, boss, kontrak, kue, pet, dialog error, pemindai, amplop).
- Satu file `index.html`, Tailwind CDN + JS vanilla. Tanpa library baru, tanpa file gambar/audio baru. Semua pixel art lewat SVG (`rect`, `shape-rendering="crispEdges"`), CSS, atau Canvas.
- Mobile-first 360×640, tetap rapi di 320×568, tanpa scroll horizontal.
- Semua teks Indonesia dan semua lewat `CONFIG.taman`. Tidak ada teks hardcode di HTML/JS selain label teknis.
- Hormati `prefers-reduced-motion` dan tombol mute (`sfxOn`). Semua bunyi lewat `beep()`/`sfx`.
- Tidak ada game over, tidak ada jalan buntu, **tidak ada bunga yang mati**. Setiap layar punya tombol kembali.
- `localStorage` selalu dalam `try/catch`; fitur tetap jalan dengan memori sementara bila storage gagal.
- Semua timer/interval/listener global dibersihkan di `cleanup()`.
- Tidak ada kamera, tidak ada data dikirim keluar, tidak ada notifikasi push.
- Desain orisinal: jangan meniru ikon, logo, sprite, atau karakter yang sudah ada.
- Setelah tiap tahap, ekstrak skrip dan jalankan `node --check`, lalu pastikan tidak ada error di console.

### 0.2 Cek awal wajib
Baca seluruh file dulu: `CONFIG`, `go()`, `order`, `STEPS`, `inits`, `cleanup()`, `revealAfter()`, layar `lock` (termasuk `#btnLockAmplop` dan `tampilTombolLock`), `amplopManager` (terutama `kembali()`, `masuk()`, `cleanup()`), `ampStore`, helper `$`, `$$`, `rand`, `beep`, `sfx`, `confetti`, `toast`, `maskot`, `sky`, dan pembaca parameter `uParams`.

Pastikan hal berikut ada. Kalau tidak ada, buat dengan perilaku yang sama persis:
1. `hariIni()` (tanggal lokal `YYYY-MM-DD`, bisa ditimpa `?tanggal=`), `parseYMD()`, `selisihHari(a,b)` (hari dari `a` ke `b`), `fmtTanggal()`, `BULAN_ID`, `ampPad()`, `ampRM()` (reduced-motion).
2. `maskot` terdefinisi dengan `say()`, `show()`, `hide()`.
3. `.btn-yl` ada di CSS dan `#petPanel` tidak memakai `inset-x`.

### 0.3 Peta z-index (jangan bentrok)
HUD 40 · pet dock 44 · maskot 45 · scanline 55 · hujan bintang 59 · confetti 60 · sheet amplop 63 · dexOverlay 65 · pet scrim 65 · pet panel 66 · dialogScrim 67 · os-win 68 · toast 70 · dexBanner 80.
Taman Cinta **tidak memakai overlay/modal**. Info bunga tampil inline, dan kupu-kupu berada di dalam wadah kebun (`position:absolute`), jadi tidak perlu z-index khusus.

### 0.4 Kunci localStorage
| Kunci | Isi |
|---|---|
| `taman-2610` | `{v:1, awal, kesehatan, terakhirSiram, beruntun, rekor, totalSiram, tahap:{}, intro, lencana:[], pertama, kunjungan}` |

`?reset=1` yang sudah ada harus ikut menghapus kunci ini. Fitur ini membaca `misi-selesai-2610` (sudah ada dari fitur amplop) untuk menampilkan tombol di layar kunci.

---

## 1. Konsep dan aturan permainan

### 1.1 Intinya
- Kebun pixel berisi **satu bunga untuk setiap bulan jadian yang sudah selesai**. Titik hitung: **3 November 2024**, jadi bunga baru ditanam setiap **tanggal 3**.
- Dia **menyiram sekali sehari**. Penyiraman membuat bunga yang masih tumbuh maju satu tahap dan menjaga kebun tetap segar.
- Kalau tidak dibuka, kebun **layu pelan-pelan** sampai batas bawah, **tidak pernah mati**, dan **pulih** hanya dalam satu atau dua kali siram.
- Nada: hangat dan lucu. Layu itu "tidur siang", bukan "mati". Tidak ada teks yang menyalahkan.

### 1.2 Rumus waktu (semua lewat `hariIni()`, bandingkan sebagai string `YYYY-MM-DD`)

Konstanta dari `CONFIG.taman`: `mulai: "2024-11-03"` → `y0=2024, m0=11, T=3`.

```
bulanSelesai(h) = ((y - y0) * 12 + (m - m0)) - (d < T ? 1 : 0)      // minimal 0
tanggalBulan(N) = y = y0 + floor((m0 - 1 + N) / 12)
                  m = ((m0 - 1 + N) % 12) + 1
                  d = T                                              // tanggal "bulan ke-N"
```
Contoh: 21 Okt 2026 → 23 · 2 Nov 2026 → 23 · 3 Nov 2026 → 24 · 3 Des 2026 → 25. Bunga ke-N berlabel `fmtTanggal(tanggalBulan(N))`, misalnya N=1 → "3 Desember 2024", N=24 → "3 November 2026".

### 1.3 Kesehatan kebun (satu nilai untuk seluruh kebun, 0–100)
Dihitung **malas** saat membuka layar (tanpa timer penghitung), bukan lewat interval.

```
hari   = max(0, selisihHari(terakhirSiram, hariIni()))   // tanggal mundur → 0
lewat  = max(0, hari - 1)                                // hari penuh yang terlewat
kesehatanKini = max(lantai, kesehatan - lewat * turunPerHari)
```
Nilai bawaan: `turunPerHari: 10`, `naikSiram: 40`, `lantai: 25`.
- Tanpa `terakhirSiram` (belum pernah siram): `kesehatanKini = 100`.
- Menyiram: `kesehatan = min(100, kesehatanKini + naikSiram)`, `terakhirSiram = hariIni()`.
- **Tidak bisa menyiram dua kali dalam sehari** (`terakhirSiram === hariIni()`): tombol berganti jadi status "sudah disiram", bukan dihapus.
- Jam dimundurkan (`terakhirSiram` di masa depan): perlakukan seperti "sudah disiram hari ini". Jangan crash, tidak perlu anti-curang.

Tingkat tampilan:
| `kesehatanKini` | Tingkat | Tampilan |
|---|---|---|
| ≥ 80 | `segar` | Warna penuh, kelopak tegak, kilau kecil |
| 55–79 | `lesu` | Kepala bunga miring dan turun 1 piksel, warna agak pudar |
| 35–54 | `layu` | Kepala turun 2 piksel, 1–2 kelopak rontok, daun menunduk |
| < 35 | `tidur` | Kepala menguncup tertutup, warna abu-kehijauan, "z" pixel kecil |

Contoh hitungan (turun 10, naik 40, lantai 25): lupa 7 hari penuh → 30 (`tidur`) · siram sekali → 70 (`lesu`) · siram lagi esoknya → 100 (`segar`).

### 1.4 Tahap tumbuh bunga
Tiap bunga punya tahap: **0 benih → 1 tunas → 2 kuncup → 3 mekar**.
- Bunga yang sudah ada saat **kunjungan pertama** langsung `mekar` (tahap 3). Itu sejarah kalian.
- Bunga yang **ditanam setelah kunjungan pertama** mulai di tahap 0, lalu **maju satu tahap setiap kali dia menyiram** (jadi mekar setelah 3 kali siram). Bunga menunggu dengan sabar kalau dia absen dan tidak pernah mundur tahap.
- Simpan `awal` = `bulanSelesai` saat kunjungan pertama. Tahap bunga N: `N <= awal ? 3 : (tahap[N] ?? 0)`.
- Kalau beberapa bulan terlewat, semua bunga baru muncul sekaligus sebagai benih.
- Saat sebuah bunga mencapai tahap 3: emit `taman:mekar {bulanKe}`, animasi mekar, bunyi, dan teks khusus.
- Setiap bulan kelipatan 12 (12, 24, 36, …) adalah **Bunga Emas** (jenis khusus, lihat 3.2).

### 1.5 Beruntun (streak) tanpa hukuman
- `beruntun` naik 1 jika menyiram dan `hari === 1` (kemarin juga siram). Jika `hari >= 2` atau belum pernah: `beruntun = 1`.
- `rekor = max(rekor, beruntun)` dan tidak pernah turun.
- Tampilan "beruntun" saat membuka layar: jika `hari >= 2` tampilkan `0` dengan teks ajakan "Mulai lagi hari ini!", bukan teks "streak putus".
- Lencana capaian (`CONFIG.taman.lencana`): bawaan di 3, 7, 14, 30, 50, 100 hari beruntun. Muncul sekali per nilai (simpan di `lencana`), dengan confetti kecil dan pesan.

---

## 2. Perubahan alur layar

### 2.1 Layar `taman` di luar urutan linear
- Tambahkan `<section class="screen" id="taman">` dan `inits.taman`.
- **Jangan** masukkan `taman` ke `order` atau `STEPS`.
- Refactor kecil yang menjaga perilaku lama: ganti pengecualian satuan di `go()` (`id !== "amplop"`) dan di pembaca `?lompat` (`uLompat === 'amplop'`) dengan satu daftar, misalnya `const LAYAR_LUAR = ['amplop','taman']`. Tombol SKIP tidak boleh muncul di `taman`.
- Peta langit: tambahkan `taman: "pagi"` ke `CONFIG.langit.peta` sebagai nilai cadangan, lalu **timpa** di `masuk()` memakai jam nyata (lihat 3.6).
- `?lompat=taman` harus bekerja.

### 2.2 Titik masuk
1. **Finale**: di `#after`, tambahkan kartu **"TAMAN CINTA 🌻"** tepat di bawah kartu Laci Amplop (tombol `#tmBtnFinale`).
2. **Layar kunci (kunjungan ulang)**: bungkus tombol laci dan tombol taman dalam satu baris `grid grid-cols-2 gap-3` (id wadah `#lockPintas`, ditaruh di posisi yang sama dengan `#btnLockAmplop` sekarang, yaitu di atas kotak kode). Tambahkan `#btnLockTaman` dengan syarat tampil sama (`CONFIG.taman.aktif && ampStore.selesai()`). Kalau hari ini belum disiram, tambahkan penanda `CONFIG.taman.penandaBelumSiram` (bawaan " 💧") di akhir teks tombolnya. Itu pemancing kembali tiap hari. Perbarui saat `screen:enter` `lock`.
3. **Dari laci amplop**: tombol kecil "🌻 KE TAMAN" di layar `amplop`. Dari taman ada "✉️ LACI AMPLOP" bila `typeof amplopManager !== 'undefined'`.
4. Asal masuk disimpan di `tamanAsal` (`"fin" | "lock" | "amplop"`). Tombol **KEMBALI**:
   - `fin`: kembali tanpa menjalankan ulang `inits.fin`. Ekstrak blok "kembali ke finale" dari `amplopManager.kembali()` menjadi fungsi bersama `kembaliKeFin(idKartu)` dengan perilaku persis sama, pakai dari kedua fitur, lalu pastikan uji amplop tetap lolos.
   - `lock`: `go("lock")`.
   - `amplop`: `go("amplop")`.

Jangan ubah logika kunci kode di `lock`.

---

## 3. Tampilan

### 3.1 Susunan layar
1. **Header**: judul `CONFIG.taman.judul` (Press Start 2P), subjudul (VT323 besar).
2. **Baris status** (tiga "chip", VT323 ≥ 18px): `🌻 23 BUNGA` · `🔥 5 HARI` (beruntun) · `💧 BELUM DISIRAM` / `💧 SUDAH DISIRAM ✓`. Status tidak boleh hanya dibedakan lewat warna; selalu ada teks.
3. **Spanduk khusus** (opsional, sesuai kondisi): hari tanggal 3 ("bunga baru ditanam"), kembali setelah absen ("tamannya kangen kamu"), atau Bunga Emas baru.
4. **Kebun** (`#tmKebun`, bingkai `.frame`): grid petak. Latar tanah pixel di bawah tiap bunga. Ada kupu-kupu dekoratif.
5. **Papan nama** (`#tmInfo`, `aria-live="polite"`): info bunga yang terakhir diketuk, di bawah kebun.
6. **Tombol utama besar**: `💧 SIRAM TAMAN`. Setelah disiram: berubah jadi tulisan status "SUDAH DISIRAM HARI INI ✓" dengan teks kecil ajakan kembali besok.
7. **Kata hari ini** (muncul setelah menyiram, lihat 3.5) dan **deretan lencana**.
8. Tombol: `✉️ LACI AMPLOP` (jika ada) dan `← KEMBALI`.
9. `padding-bottom` ±5rem agar tidak tertutup pet dock dan maskot.

### 3.2 Petak dan sprite bunga
- Grid CSS `repeat(auto-fill, minmax(44px, 1fr))`, `gap: 4px`. Tiap petak `<button>` **minimal 44×44px** (cocok di 360px dan 320px). Tambahkan **satu petak "tanah kosong" bergaris putus** untuk bulan berikutnya (berlabel tanggal tanamnya).
- Kebun boleh bertumbuh ke bawah tanpa batas jumlah; layar boleh scroll vertikal.
- Sprite SVG, grid piksel **12 lebar × 14 tinggi**, dibuat dari `rect` (`crispEdges`), dengan lapisan: gundukan tanah (coklat), tangkai, daun, kepala bunga. Warna kepala/pusat/tangkai diambil dari `CONFIG.taman.jenis`.
- Satu fungsi pembuat sprite: `spriteBunga(jenisIdx, tahap, tingkat)`.
  - Tahap 0: gundukan dengan satu biji. Tahap 1: tunas dua daun. Tahap 2: kuncup tertutup berwarna jenis. Tahap 3: kepala penuh.
  - Tingkat `lesu`/`layu`/`tidur` mengubah: posisi kepala (turun 1–2 piksel), rontok kelopak, warna dicampur ke abu-hijau (fungsi `campurWarna(hex, abu, faktor)`), dan `tidur` menggambar kepala sebagai kuncup tertutup ditambah "z" pixel. Variasi kecil per bunga (kemiringan 0 atau 1 piksel) ditentukan dari indeks, **deterministik**, bukan acak.
- **Bentuk kepala 7×7** (`P`=kelopak, `D`=kelopak gelap, `C`=pusat, `.`=kosong). Contoh yang disarankan; bentuk lain digambar dengan konvensi yang sama:
```
aster/daisy       matahari
...P...           .D.P.D.
.P.P.P.           DPPPPPD
..PPP..           .PCCCP.
PPPCPPP           PPCCCPP
..PPP..           .PCCCP.
.P.P.P.           DPPPPPD
...P...           .D.P.D.
```
- **12 jenis bunga** bergilir dengan `(N-1) % 12`: Aster, Tulip, Bunga Matahari, Mawar, Lavender, Sakura, Teratai, Melati, Anggrek, Poppy, Dahlia, Krisan (nama dan warna dapat diubah di `CONFIG.taman.jenis`).
- **Bunga Emas** (N kelipatan 12): kepala emas dengan mahkota 3 piksel dan kilau di sudut, ukuran petak sama.
- Tinggi SVG petak ±40px, `aria-hidden`; informasi untuk pembaca layar ada di `aria-label` tombol.

### 3.3 Ketuk bunga → papan nama
Mengetuk bunga mengisi `#tmInfo` (tanpa modal): `"Bunga ke-{N} · {nama jenis} · {tanggal}"`, tahap/tingkat dalam kata ("mekar, segar"), dan satu keterangan dari `CONFIG.taman.kenangan[N]` (isi sendiri oleh Arkan) atau, jika kosong, dari `captionUmum` (bergiliran menurut N). Bunyi kecil `sfx.tap()` dan lompatan kecil sprite (dilewati saat reduced-motion). Petak yang dipilih diberi kontur kuning.

### 3.4 Animasi menyiram (≤ 2,2 detik, bisa dilewati dengan ketukan)
1. Ember pixel bergeser di atas kebun dan miring (CSS transform).
2. Tetes air pixel (elemen kecil dengan animasi jatuh) turun di atas petak, maksimal 24 tetes sekaligus.
3. Bunga `bounce` sekali, kilau muncul, bunyi hujan (deret nada pendek `triangle`, 6 nada).
4. Bunga yang tahapnya naik berganti sprite; bunga yang baru mekar mendapat animasi mekar (skala 0,6 → 1) dan arpeggio naik 4 nada.
5. Setelah selesai: status berubah, kupu-kupu muncul (3.5), teks "kata hari ini" tampil.
- `prefers-reduced-motion`: tanpa ember, tetes, bounce, dan kupu-kupu bergerak; keadaan akhir langsung tampil; bunyi tetap mengikuti mute.
- Semua `setTimeout` animasi disimpan di larik dan dibersihkan di `cleanup()`.

### 3.5 Kata hari ini, kupu-kupu, hadiah kecil
- **Kata hari ini**: kalimat pendek dari Arkan, dipilih **deterministik per hari**: `idx = selisihHari(mulai, hariIni()) % kataHarian.length`. Tidak berulang selama panjang daftar. Tampil sekali di bawah tombol setelah menyiram, dan bisa dibaca lagi hari itu.
- **Kupu-kupu**: setelah menyiram muncul satu kupu-kupu pixel yang terbang melintas kebun (CSS keyframes, ±8 detik, dihapus saat `animationend`). Diketuk: `toast` dengan kalimat dari `CONFIG.taman.kupu`. Tidak ada kupu-kupu di mode reduced-motion.
- **Hari tanggal 3**: spanduk khusus, confetti kecil saat menyiram, dan tombol `✉️ BUKA AMPLOP TANGGAL 3` (hanya jika `amplopManager` ada).

### 3.6 Langit mengikuti jam nyata
Di `masuk()`, setelah handler langit bawaan berjalan, panggil `sky.setStage(tahapDariJam(jam))` dengan `jam = new Date().getHours()` (bisa ditimpa `?jam=HH` untuk uji):
```
04–05 subuh · 06–09 pagi · 10–13 siang · 14–15 siangakhir · 16 sore · 17 senja · 18 petang · 19–03 malamIndah
```
Tabel ini ada di `CONFIG.taman.langitJam` agar bisa diubah. Hanya tahap yang ada di `CONFIG.langit.tahap` yang boleh dipakai; jika tidak ada, biarkan langit bawaan.

---

## 4. Perilaku data dan peristiwa

### 4.1 Kunjungan pertama (intro sekali seumur hidup)
- Simpan `awal`, `kesehatan: 100`, `intro: true`, `pertama`.
- Intro: kebun kosong → teks `CONFIG.taman.intro.teks` → bunga muncul **satu per satu dari bulan ke-1 sampai terakhir**, total ≤ 3,5 detik (jeda dihitung dari jumlah bunga), tiap kemunculan beep naik yang dibatasi 8 nada. Ketuk di mana saja melewati intro. Reduced-motion: langsung tampil semua.
- Emit `taman:intro {n}`. Sesudah intro, ajakan menyiram.

### 4.2 Membuka layar (`masuk()`)
1. Muat state (`tamanStore`), hitung `bulanSelesai`, tambahkan bunga baru (tahap 0) bila ada.
2. Hitung `kesehatanKini`, tingkat, dan beruntun efektif.
3. Render kebun, status, spanduk, lencana.
4. Maskot menyapa sesuai kondisi (3 kemungkinan, lihat 5).
5. Pasang `setInterval` 60 detik (`tick`) yang hanya memeriksa pergantian tanggal; kalau berganti, hitung ulang dan render, tanpa suara. Bersihkan di `cleanup()`.
6. Pasang `visibilitychange` **sekali** di `init()` (bukan tiap masuk) yang memanggil `tick` saat tab kembali terlihat.

### 4.3 Menyiram (`siram()`)
1. Jika sudah disiram hari ini: `toast(CONFIG.taman.teksUI.sudahSiram)` dan berhenti (tanpa efek).
2. Hitung `kesehatanKini`. Bila `< 55` sebelum menyiram, tandai sebagai pemulihan.
3. Perbarui `kesehatan`, `terakhirSiram`, `beruntun`, `rekor`, `totalSiram`, tahap bunga baru (+1, maks 3). Simpan.
4. Jalankan animasi 3.4. Emit `taman:siram {beruntun, hari}`; bila pulih emit `taman:pulih {}`; tiap bunga yang mencapai tahap 3 emit `taman:mekar {bulanKe}`; lencana baru emit `taman:lencana {hari}`.
5. Tampilkan kata hari ini dan teks status baru. Fokus tetap pada tombol (jangan hilangkan tombol secara tiba-tiba; ganti isinya).

### 4.4 Tabel peristiwa
| Event | Data | Pemicu |
|---|---|---|
| `taman:intro` | `{n}` | Intro kunjungan pertama |
| `taman:siram` | `{beruntun, hari}` | Menyiram berhasil |
| `taman:mekar` | `{bulanKe}` | Bunga mencapai tahap 3 |
| `taman:pulih` | `{}` | Menyiram saat kesehatan < 55 |
| `taman:lencana` | `{hari}` | Lencana baru terbuka |

Jangan mengubah event yang sudah ada. Boleh menambahkan pendengar baru, misalnya maskot.

---

## 5. Aksesibilitas
- Tiap petak adalah `<button>` dengan `aria-label`, misalnya "Bunga bulan ke-7, Mawar, mekar, segar, tanggal 3 Juni 2025". Petak benih/tunas/kuncup menyebut tahapnya.
- Wadah kebun `role="group"` dengan `aria-label` dari `CONFIG`. Papan nama `aria-live="polite"`.
- Tombol siram: `aria-label` mengikuti status; saat sudah disiram, tombol tetap bisa difokus dan memberi toast, bukan `disabled` total (agar tidak menjadi jalan buntu bagi pembaca layar).
- Fokus terlihat (`:focus-visible` outline kuning, sudah ada gaya serupa di fitur amplop).
- Kontras: teks status dan papan nama cukup besar (≥ 18px VT323), warna hanya penegas.
- `prefers-reduced-motion`: lihat 3.4 dan 3.5.

---

## 6. Pembersihan (`cleanup()`)
Tambahkan `if (typeof tamanManager !== 'undefined') tamanManager.cleanup();` di `cleanup()` (pola yang sama dengan `amplopManager`). Hentikan: interval `tick`, semua timeout animasi/intro, hapus elemen tetes air dan kupu-kupu, dan lepaskan pendengar sementara. Listener `visibilitychange` yang dipasang sekali tidak menimbulkan kebocoran, tetapi harus memeriksa `cur === 'taman'` sebelum bekerja.

---

## 7. Skema CONFIG (isi teks siap pakai)

Edit seperlunya, tetapi **pertahankan nada Arkan yang proaktif dan hangat**, bukan pasif atau lebay.

```js
taman: {
  aktif: true,
  judul: "TAMAN CINTA",
  subjudul: "Satu bunga untuk tiap bulan kita. Siram sekali sehari ya, biar tamannya senang.",
  mulai: "2024-11-03",
  turunPerHari: 10, naikSiram: 40, lantai: 25,
  penandaBelumSiram: " 💧",
  tombolFinale: "KUNJUNGI TAMAN 🌻",
  tombolLock: "🌻 TAMAN CINTA",
  kartuFinale: {
    judul: "TAMAN CINTA 🌻",
    teks: "Arkan menanam satu bunga untuk setiap bulan kita. Tiap tanggal 3 bertambah satu. Siram sekali sehari ya, tamannya senang ditengok."
  },
  intro: { teks: "Arkan menanam satu bunga untuk setiap bulan kita. Sampai hari ini sudah {n} bunga. Jagain ya, cukup disiram sekali sehari." },
  teksUI: {
    siram: "💧 SIRAM TAMAN", sudahSiramLabel: "SUDAH DISIRAM HARI INI ✓",
    sudahSiram: "Hari ini sudah disiram. Kembali lagi besok ya, tamannya senang ditengok 🌼",
    ajakBesok: "Besok ada air lagi buat bunga-bunganya.",
    chipBunga: "{n} BUNGA", chipBeruntun: "{n} HARI", chipBeruntunMulai: "MULAI LAGI HARI INI!",
    chipBelum: "BELUM DISIRAM", chipSudah: "SUDAH DISIRAM ✓",
    kembali: "← KEMBALI", keAmplop: "✉️ LACI AMPLOP", keTaman: "🌻 KE TAMAN",
    kebunLabel: "Kebun bunga, satu bunga untuk tiap bulan",
    kataHariIni: "KATA HARI INI", lencanaJudul: "LENCANA",
    petakKosong: "Bunga berikutnya ditanam {tanggal}",
    infoBunga: "Bunga ke-{n} · {jenis} · {tanggal}",
    ariaBunga: "Bunga bulan ke-{n}, {jenis}, {tahap}, {tingkat}, tanggal {tanggal}",
    tahap: ["benih", "tunas", "kuncup", "mekar"],
    tingkat: { segar: "segar", lesu: "agak lesu", layu: "layu, perlu air", tidur: "tidur siang" },
    deskTingkat: {
      segar: "Segar dan senang 🌼",
      lesu: "Agak lesu, haus nih 💧",
      layu: "Layu sedikit, tapi masih sayang kamu 🥀",
      tidur: "Lagi tidur siang. Bangunin pakai air ya 😴"
    }
  },
  spanduk: {
    tanggal3: "Hari ini tanggal 3, hari jadian kita. Bunga baru ditanam. 🌱",
    emas: "Bunga Emas tahun ke-{tahunKe} ditanam. Spesial buat kamu. 🌟",
    balik: "Tamannya kangen kamu 🥺 Tapi masih sehat kok. Siram dulu yuk, nanti pulih pelan-pelan.",
    pulih: "Yeay, tamannya mulai pulih! Besok disiram lagi ya, biar segar penuh. 💚",
    mekar: "Bunga ke-{n} mekar! Cantik banget, seperti kamu. 🌸"
  },
  jenis: [
    { nama: "Aster",           bentuk: "daisy",     kelopak: "#fff4e0", pusat: "#ffd93d", tangkai: "#4fbf8b" },
    { nama: "Tulip",           bentuk: "tulip",     kelopak: "#ff6fae", pusat: "#c2417f", tangkai: "#4fbf8b" },
    { nama: "Bunga Matahari",  bentuk: "matahari",  kelopak: "#ffd93d", pusat: "#8a5a36", tangkai: "#3da878" },
    { nama: "Mawar",           bentuk: "mawar",     kelopak: "#e8466e", pusat: "#a62a4c", tangkai: "#3da878" },
    { nama: "Lavender",        bentuk: "lavender",  kelopak: "#b79ced", pusat: "#8b6fd1", tangkai: "#4fbf8b" },
    { nama: "Sakura",          bentuk: "sakura",    kelopak: "#ffc2dc", pusat: "#ff8fbd", tangkai: "#8a5a36" },
    { nama: "Teratai",         bentuk: "teratai",   kelopak: "#ffb3d9", pusat: "#ffd93d", tangkai: "#3da878" },
    { nama: "Melati",          bentuk: "melati",    kelopak: "#ffffff", pusat: "#ffe9a8", tangkai: "#4fbf8b" },
    { nama: "Anggrek",         bentuk: "anggrek",   kelopak: "#d9a6ff", pusat: "#ffd93d", tangkai: "#4fbf8b" },
    { nama: "Poppy",           bentuk: "poppy",     kelopak: "#ff7b54", pusat: "#1a1033", tangkai: "#3da878" },
    { nama: "Dahlia",          bentuk: "dahlia",    kelopak: "#ff8fbd", pusat: "#ffd93d", tangkai: "#4fbf8b" },
    { nama: "Krisan",          bentuk: "krisan",    kelopak: "#9bc1ff", pusat: "#fff4e0", tangkai: "#3da878" }
  ],
  jenisEmas: { nama: "Bunga Emas", kelopak: "#ffd93d", pusat: "#ff6fae", tangkai: "#3da878", mahkota: "#fff4e0" },
  kenangan: {
    /* Isi sendiri, satu kalimat per bulan, contoh:
       1: "Bulan pertama. Arkan gugup, tapi senang banget.",
       12: "Setahun! Terima kasih sudah bertahan." */
  },
  captionUmum: [
    "Bulan ke-{n}. Satu bulan lagi kita lewati bareng.",
    "Bunga ini tumbuh karena kamu rajin ada di cerita Arkan.",
    "Kalau bunga ini bisa ngomong, dia bakal bilang: terima kasih, {n} bulan ini seru.",
    "Arkan nanam ini sambil senyum-senyum, mikirin kamu.",
    "Bulan ke-{n}, dan Arkan masih pengin nambah bunga lagi.",
    "Tiap bunga di sini, satu bulan Arkan bersyukur punya kamu."
  ],
  kataHarian: [
    "Hari ini Arkan mau lebih peka dari kemarin.",
    "Kalau kamu capek, kabari Arkan. Arkan yang datang duluan.",
    "Kamu itu alasan Arkan semangat belajar jadi lebih baik.",
    "Hari ini Arkan doain urusanmu lancar, satu per satu.",
    "Minum air dulu ya. Bunga aja butuh air, apalagi kamu.",
    "Arkan bangga sama usaha kecilmu hari ini, sekecil apa pun.",
    "Kalau ada yang bikin kesal, ceritain ke Arkan. Arkan dengerin dulu.",
    "Senyum kamu hari ini sudah cukup bikin hari Arkan bagus.",
    "Jangan lupa makan. Kalau Arkan di dekatmu, udah Arkan beliin.",
    "Arkan janji terus berusaha jadi tempat pulang yang nyaman.",
    "Kamu nggak harus kuat setiap hari. Istirahat itu boleh.",
    "Hari ini giliran Arkan yang bikin kamu ketawa.",
    "Terima kasih sudah mampir ke taman ini. Arkan senang banget.",
    "Satu hal yang Arkan yakin: kamu lebih hebat dari yang kamu kira."
    /* Tambahkan sampai 30+ agar tidak cepat berulang. */
  ],
  kupu: [
    "Kupu-kupu bilang: Arkan sayang kamu.",
    "Hihi, geli! Kupu-kupunya kabur sambil senyum.",
    "Kupu-kupu ini titipan Arkan buat nemenin kamu."
  ],
  lencana: [
    { hari: 3,   judul: "Tukang Kebun Cilik",    pesan: "3 hari berturut-turut! Bunganya senang banget." },
    { hari: 7,   judul: "Penjaga Taman",         pesan: "Seminggu penuh! Arkan bangga sama kamu." },
    { hari: 14,  judul: "Ahli Bunga",            pesan: "Dua minggu! Tamanmu makin cantik." },
    { hari: 30,  judul: "Ratu Taman",            pesan: "30 hari! Arkan rencanain hadiah kecil buat ini." },
    { hari: 50,  judul: "Pawang Kupu-Kupu",      pesan: "50 hari! Kupu-kupunya makin banyak." },
    { hari: 100, judul: "Legenda Taman Cinta",   pesan: "100 hari! Arkan nggak nyangka kamu sesetia ini. Terima kasih." }
  ],
  langitJam: [[4, "subuh"], [6, "pagi"], [10, "siang"], [14, "siangakhir"], [16, "sore"], [17, "senja"], [18, "petang"], [19, "malamIndah"], [24, "malamIndah"]],
  maskot: {
    masuk: ["Yuk siram tamannya!", "Bunganya nunggu kamu, nih."],
    sudahSiram: ["Tamannya udah seger. Kembali besok ya!"],
    balik: ["Tamannya kangen. Siram yuk, pelan-pelan juga nggak apa-apa."],
    mekar: ["Wah, ada yang mekar!"],
    lencana: ["Lencana baru! Hebat!"]
  }
}
```

> Catatan untuk agent: `langitJam` dibaca sebagai "jam mulai, tahap" yang berurutan; pilih entri terakhir dengan jam mulai ≤ jam sekarang. Semua string melewati fungsi isi-template (`tmFill`) yang menyuntik `{n}`, `{jenis}`, `{tanggal}`, `{tahunKe}`, plus `{namaPanggilan}`/`{pengirim}` dari `CONFIG`. Placeholder tanpa nilai harus jatuh ke teks kosong, jangan tampil mentah sebagai `{…}`.

---

## 8. Kasus tepi (wajib ditangani)
- `localStorage` rusak/penuh/dinonaktifkan atau JSON rusak: mulai dari state baru di memori; tidak crash.
- Tanggal perangkat dimundurkan atau dimajukan jauh: tidak crash; kesehatan tidak pernah di bawah `lantai`; bunga tidak pernah hilang atau mundur tahap.
- Ketuk tombol siram berkali-kali: hanya satu animasi berjalan (guard), efek hanya sekali per hari.
- Pindah layar saat animasi berjalan: `cleanup()` membersihkan timeout, tetes air, dan kupu-kupu; tidak ada error.
- Tab dibiarkan terbuka melewati tengah malam: `tick`/`visibilitychange` memperbarui tombol siram tanpa memuat ulang.
- Banyak bunga (misalnya bulan ke-60 atau 120): tetap lancar, grid tumbuh ke bawah; sprite dibuat sekali per petak.
- Layar 320×568: tidak ada scroll horizontal, petak ≥ 44px, teks tidak terpotong.
- `CONFIG.taman.kenangan[N]` kosong: pakai `captionUmum`. `kataHarian` kosong: sembunyikan bagian "kata hari ini".
- Fitur amplop tidak ada: sembunyikan tombol amplop tanpa error.
- `taman.aktif === false`: sembunyikan kartu finale dan tombol layar kunci.

---

## 9. Urutan kerja (jalankan `node --check` setelah tiap tahap)
1. **Pra-syarat 0.2**: pastikan helper tanggal, `maskot`, `.btn-yl`, dan refactor kecil `LAYAR_LUAR` (2.1). Jalankan ulang uji amplop sebagai garis dasar.
2. **CONFIG + penyimpanan**: tambah `CONFIG.taman`, `tamanStore` (try/catch + memori cadangan), kunci `taman-2610` ke `?reset=1`, fungsi rumus waktu (1.2 sampai 1.5) sebagai fungsi murni yang mudah diuji.
3. **Sprite bunga**: `spriteBunga()` dan `campurWarna()`, 12 jenis + Bunga Emas, 4 tahap × 4 tingkat.
4. **Layar kebun**: HTML `#taman`, `inits.taman`, grid, status, papan nama, petak kosong, langit mengikuti jam.
5. **Menyiram**: `siram()`, animasi, kupu-kupu, kata hari ini, lencana, event.
6. **Intro dan kondisi khusus**: intro kunjungan pertama, spanduk tanggal 3/balik/pulih/mekar, Bunga Emas.
7. **Titik masuk**: kartu finale, `#lockPintas` dengan dua tombol (+ penanda 💧), tombol silang dengan laci amplop, `kembaliKeFin()` bersama.
8. **Aksesibilitas, reduced-motion, mute, `cleanup()`**.
9. **Uji penuh** (bagian 10), lalu serahkan.

---

## 10. Uji penerimaan (Playwright Chromium, 360×640 dan 320×568)
Catatan: CDN Tailwind bisa tidak terjangkau di sandbox. Buat salinan uji dengan CSS Tailwind statis, **bukan** untuk dikirim. Uji lintas hari memakai `?tanggal=` pada konteks browser **yang sama** (localStorage bertahan antar `goto`), dan `?jam=` untuk langit.

1. `?reset=1&lompat=taman&tanggal=2026-10-21`: intro berjalan, **23 bunga** semuanya mekar, ada satu petak kosong berlabel "3 November 2026", tanpa scroll horizontal, tanpa error.
2. Ketuk **Siram**: status "SUDAH DISIRAM ✓", `beruntun=1`, kata hari ini muncul, `localStorage` terisi; ketuk lagi → toast, tanpa efek ganda.
3. `?tanggal=2026-10-22` (tanpa reset): `beruntun` tampil 1, siram → 2, kesehatan 100.
4. `?tanggal=2026-10-30`: kesehatan = **30** (`tidur`), `beruntun` tampil 0 dengan teks "MULAI LAGI HARI INI!", **tidak ada bunga yang hilang**; siram → kesehatan **70** (`lesu`), event `taman:pulih`, spanduk pulih; `?tanggal=2026-10-31` siram → **100** (`segar`).
5. `?tanggal=2026-11-02`: tetap 23 bunga. `?tanggal=2026-11-03`: bunga ke-**24** muncul sebagai **benih** dan berjenis **Bunga Emas**, spanduk tanggal 3 tampil; siram → tunas; `2026-11-04` → kuncup; `2026-11-05` → **mekar** dengan event `taman:mekar {bulanKe:24}`.
6. Kasus tanggal mundur (`?tanggal=2026-10-15` setelah state di atas): tidak crash, dianggap sudah disiram.
7. Ketuk bunga ke-7: papan nama berisi "Bunga ke-7 · Aster/…" dan tanggal "3 Juni 2025" yang benar; `aria-label` benar.
8. Emulasi `prefers-reduced-motion: reduce`: tanpa ember/tetes/kupu-kupu; keadaan akhir langsung; fungsi tetap benar.
9. Mute aktif: tidak ada bunyi (`beep` melewati `sfxOn`).
10. Simulasi `localStorage` melempar error: fitur tetap jalan (memori); tanpa error.
11. Pindah layar saat animasi menyiram: tidak ada timer yatim, tetes air, atau kupu-kupu tersisa; tanpa error.
12. Titik masuk: alur finale → kartu taman → buka → KEMBALI ke finale (tanpa menjalankan ulang `inits.fin`, `#after` tetap terbuka). Muat ulang → `#lockPintas` menampilkan **dua tombol** dengan penanda 💧 saat belum disiram; kode kunci lama tetap berfungsi. Silang taman ↔ amplop dan kembali berjalan.
13. Langit: `?jam=21` → `malamIndah`; `?jam=7` → `pagi`.
14. Fitur lama tidak berubah: kuis, gacha, boss, kontrak, kue, pet, dialog error, pemindai, **dan seluruh uji amplop** tetap lolos.

---

## 11. Di luar lingkup
Tidak ada backend, akun, notifikasi push, atau sinkronisasi antar perangkat (kebun tersimpan per perangkat). Jenis bunga dan teks diubah lewat `CONFIG.taman` lalu `index.html` di-upload ulang.

---

## 12. Bagian CONFIG yang perlu kamu sesuaikan
- `kenangan` — satu kalimat nyata untuk setiap bulan (bulan 1, 6, 12, 18, 24 paling berkesan). Ini yang membuat tiap bunga terasa pribadi.
- `kataHarian` — tambah sampai 30 atau lebih supaya tidak cepat berulang.
- `lencana` — ganti pesan 30 hari dengan hadiah nyata yang benar-benar sanggup kamu berikan.
- `jenis` — ganti nama atau warna bunga sesuai bunga favorit Zahwa.
- `kupu` dan `captionUmum` — sesuaikan dengan gaya bicaramu.
- `turunPerHari`, `naikSiram`, `lantai` — kalau mau kebun lebih cepat atau lebih lambat layu.

---

## 13. Prompt siap tempel untuk agent

> Kamu akan memodifikasi file `index.html` yang sudah ada (web ulang tahun bertema 8-bit + misi rahasia, Tailwind CDN + JS vanilla, satu file) yang sudah memuat fitur Amplop "Buka kalau…". Jangan menulis ulang dari nol dan jangan mengubah perilaku fitur yang sudah ada. Baca seluruh file dulu, terutama `CONFIG`, `go()`, `order`, `inits`, `cleanup()`, `revealAfter()`, layar `lock` (`#btnLockAmplop`, `tampilTombolLock`), `amplopManager` (terutama `kembali()`), `ampStore`, `hariIni()`, `parseYMD`, `selisihHari`, `fmtTanggal`, `maskot`, `sky`, dan helper `$`, `$$`, `rand`, `beep`, `sfx`, `confetti`, `toast`. Tambahkan fitur "Taman Cinta" sesuai dokumen `SPEC-taman-cinta.md` yang terlampir. Ikuti persis bagian 0 (aturan umum dan cek pra-syarat), aturan permainan dan rumus waktu (bagian 1), alur layar (bagian 2), skema `CONFIG` dan isi teks yang sudah ditulis (bagian 7), dan urutan kerja (bagian 9). Layar `taman` TIDAK boleh masuk ke `order`/`STEPS` dan TIDAK boleh memunculkan tombol SKIP. Bunga TIDAK PERNAH mati: kesehatan kebun tidak boleh turun di bawah `lantai`, dan tahap bunga tidak pernah mundur. Hitung kesehatan secara malas dari selisih tanggal (tanpa timer penghitung), dan semua perbandingan tanggal lewat `hariIni()`. Aturan keras: mobile-first (360×640, juga 320×568), tanpa library atau file gambar/audio baru (pixel art lewat SVG/CSS/Canvas), semua teks Indonesia dan lewat `CONFIG`, hormati `prefers-reduced-motion` dan tombol mute, tidak ada game over atau jalan buntu, `localStorage` selalu dalam `try/catch`, semua timer dibersihkan di `cleanup()`, tidak ada kamera atau data yang dikirim keluar, semua desain orisinal. Nada teks: Arkan proaktif dan hangat, bukan pasif atau lebay. Setelah tiap tahap, ekstrak skrip dan jalankan `node --check`, lalu pastikan tidak ada error di console, dan jalankan uji penerimaan bagian 10 (termasuk uji amplop sebagai garis dasar). Output: `index.html` yang sudah diperbarui, ringkasan singkat perubahan, dan daftar bagian `CONFIG` yang perlu disesuaikan pengguna.
