# CSC2026 — Portal Pendaftaran Atlet & Papan Markah (v3)

> **Reka bentuk UI/UX kekal sama seperti repo asal.** Markup dan CSS diambil
> terus daripada `mykelabdraft` — nav, ticker berita, glass-card, kad hero,
> penanda STEP 1/2/3, pas sukan, sidebar admin, semuanya sama. Yang diganti
> hanyalah enjin di belakangnya.


Sistem pendaftaran atlet untuk CBP Sports Carnival 2026. Dibina sebagai PWA
tanpa kos hosting: frontend statik (GitHub Pages / Netlify), Google Apps Script
sebagai backend, Google Sheets sebagai pangkalan data, Google Drive untuk gambar.

Direka untuk 700+ peserta, maksimum 2 sukan setiap atlet, dengan papan markah
langsung yang boleh dilihat oleh semua staf sepanjang carnival.

---

## Apa yang berubah berbanding repo asal

| Kekal sama | Diganti |
|---|---|
| Semua markup & CSS (reka bentuk, susun atur, warna, animasi) | Seluruh lapisan JS |
| Nama fungsi yang dipanggil markup (`app.navigate`, `adminApp.saveSport`, …) | Apa yang fungsi itu lakukan |
| Aliran 3 langkah atlet, sidebar admin, semua modal | Pengesahan, kapasiti, audit — kini dikuatkuasakan di pelayan |

**Yang ditambah pada UI (dalam gaya yang sama):**

- Skrin log masuk admin — sebelum ini portal admin terbuka tanpa sebarang kata laluan
- Pemilihan kategori di dalam kad sukan (backend memerlukan kategori bagi setiap sukan)
- Tiga menu sidebar baharu: QR / Check-In, Tetapan Sistem, Log Audit
- Medan 4-digit pengesahan pada skrin login atlet (tersembunyi melainkan diaktifkan)
- Kod QR sebenar menggantikan corak jalur hiasan pada pas sukan
- **Papan markah langsung menggantikan skrin Berita sebagai skrin utama kedua**
  (berita kekal, diturunkan menjadi kad dan ticker)
- **Portal juri berasingan** di `web/juri/` untuk kemasukan markah di gelanggang
- Empat menu sidebar admin baharu: Pengesahan Markah, Jadual Acara,
  Kontinjen & Mata, Akaun Juri

**Aset tempatan, bukan CDN.** `index.html` memuatkan Tailwind dan ikon dari
folder `vendor/` dan bukan dari cdn.tailwindcss.com / unpkg. Rupa 100% sama —
CSS dijana oleh Tailwind CLI daripada fail HTML yang sama, dan setiap kelas
yang muncul dalam DOM hidup telah disemak wujud dalam CSS terkumpul. Sebabnya
praktikal: versi CDN memuat turun ~750KB dan mengkompil CSS dalam pelayar
setiap kali halaman dibuka, dan jika rangkaian venue menyekat CDN, halaman
terpapar tanpa sebarang gaya langsung. Versi tempatan ialah 247KB, di-cache
oleh service worker selepas lawatan pertama.

Fail `index.cdn.html` dan `admin/index.cdn.html` ialah versi CDN asal, disimpan
sebagai rujukan jika anda mahu membandingkan.

---

## Struktur fail

```
apps-script/          → tampal ke editor Apps Script (satu fail = satu fail .gs)
  00_Config.gs        konfigurasi, skema tab, tetapan keselamatan
  01_Setup.gs         pemasangan idempoten, pengguna admin, data contoh
  02_Router.gs        doGet / doPost, jadual peranan
  03_Auth.gs          log masuk admin, sesi, had kadar
  04_Athlete.gs       katalog, pengesahan Staff ID, penghantaran pendaftaran
  05_Admin.gs         papan pemuka, CRUD, muat naik, laporan, check-in
  06_Util.gs          pembantu sheet / cache / kripto / audit / Drive
  07_Scoring.gs       enjin markah, standings, papan markah awam + cache
  08_Judge.gs         peranan juri, hantar markah, pengesahan admin, acara
  appsscript.json     zon masa, skop OAuth, tetapan web app

web/                  → hos di GitHub Pages
  index.html          portal atlet + papan markah (aset tempatan)
  index.cdn.html      versi sama, menggunakan CDN (rujukan)
  admin/index.html    portal admin (aset tempatan)
  admin/index.cdn.html
  juri/index.html     portal kemasukan markah untuk juri
  juri/index.cdn.html
  vendor/             tailwind.css (134KB) + lucide-subset.js (12KB)
  manifest.json       manifes PWA
  sw.js               service worker
  assets/             ikon PWA + favicon (dijana oleh build-icons-pwa.js)

tools/                → skrip pembinaan (tidak perlu untuk deploy)
  build-icons.js      jana semula subset ikon Lucide dari HTML
  build-icons-pwa.js  jana ikon PWA + favicon daripada logo rasmi
  build-cdn.js        jana semula varian *.cdn.html
  logo-source.png     logo rasmi — sumber untuk build-icons-pwa.js
```

Untuk membina semula `vendor/tailwind.css` selepas mengubah markup:

```bash
npx tailwindcss@3 -c tailwind.config.js -i src.css -o web/vendor/tailwind.css --minify
```

(`tailwind.config.js` mesti sepadan dengan blok `tailwind.config` dalam HTML —
`darkMode:'class'`, fon Inter/Rajdhani/JetBrains Mono, warna `motorsport`.)

Selepas menambah ikon baharu dalam markup, jana semula subsetnya:

```bash
node tools/build-icons.js    # imbas HTML, tulis vendor/lucide-subset.js
node tools/build-cdn.js      # segerakkan varian *.cdn.html
```

---

## Data ujian (dummy)

Selain `setupSystem()` yang membina struktur, ada set fungsi berasingan untuk
menjana data ujian. Fungsi ini **selamat dijalankan pada sistem yang sudah
mempunyai data sebenar** — ia hanya MENAMBAH baris bertanda `DUMMY`, tidak
pernah mengubah atau memadam rekod sedia ada, dan tidak menyentuh konfigurasi
sukan/kategori.

Melalui menu **CSC2026 → Data Ujian (Dummy)**, atau terus dari editor:

| Fungsi | Apa yang berlaku |
|---|---|
| `seedDummyData()` | 40 staf ujian, kira-kira separuh didaftarkan |
| `seedDummyData(200)` | 200 staf ujian |
| `seedDummyData(700, 1)` | 700 staf, semua didaftarkan — ujian beban penuh |
| `countDummyData()` | Berapa banyak rekod ujian ada sekarang |
| `removeDummyData()` | Buang SEMUA rekod ujian, dan tiada yang lain |

Semua Staff ID ujian bermula dengan `DUMMY` (contoh `DUMMY0001`), jadi ia mudah
dikenal pasti dalam Sheet dan dalam portal admin. Pendaftaran ujian menghormati
peraturan sebenar: maksimum 2 sukan, kategori mesti milik sukan itu, dan
kelayakan jantina dipatuhi — jadi kiraan kapasiti pada papan pemuka kelihatan
realistik.

Seed juga menjana **acara dan markah ujian** supaya papan markah ada isi untuk
diuji: acara ditandakan `DUMMY-EV-…`, markah `DUMMY-RES-…`. Sebahagian markah
sengaja ditinggalkan sebagai DRAF supaya panel pengesahan admin juga ada bahan.

Menjalankan `seedDummyData()` dua kali tidak menduplikasi apa-apa; ID yang
sudah wujud dilangkau.

Untuk kembali bersih sebelum acara sebenar: `removeDummyData()`.

---

## Pemasangan (ikut turutan)

### 1. Google Sheet + Apps Script

1. Cipta Google Sheet baharu. Namakan `CSC2026 Database`.
2. **Extensions → Apps Script**.
3. Padam `Code.gs` lalai. Cipta tujuh fail dengan nama yang sama seperti dalam
   `apps-script/` dan tampal kandungannya.
4. Klik ikon gear (Project Settings) → tandakan *Show `appsscript.json`* →
   ganti kandungannya dengan `apps-script/appsscript.json`.
5. Pilih fungsi `setupSystem` → **Run**. Benarkan kebenaran apabila diminta.

   Anda akan lihat dialog mengesahkan tab mana yang dicipta. Menjalankan
   `setupSystem()` semula pada bila-bila masa adalah selamat — ia menambah tab
   dan kolum yang hilang tetapi tidak pernah memadam data.

### 2. Folder Google Drive

1. Cipta folder Drive, contoh `CSC2026 Gambar Atlet`.
2. Salin ID folder daripada URL:
   `https://drive.google.com/drive/folders/`**`ID_DI_SINI`**
3. Dalam tab `Config` Sheet, tetapkan `DRIVE_FOLDER_ID` kepada ID tersebut.

### 3. Pengguna admin

Dalam editor Apps Script, jalankan `createAdminUserPrompt()` (atau guna menu
**CSC2026 → Cipta Pengguna Admin** dalam Sheet). Masukkan username, nama penuh,
peranan dan kata laluan (minimum 10 aksara).

Peranan:

| Peranan      | Boleh buat                                                        |
|--------------|-------------------------------------------------------------------|
| `juri`       | Masukkan markah untuk sukan dalam skopnya sahaja — tiada akses lain |
| `viewer`     | Lihat papan pemuka, cari atlet, eksport, check-in                  |
| `admin`      | Semua di atas + pinda/batal pendaftaran, urus sukan, sahkan markah |
| `superadmin` | Semua di atas + padam kekal, ganti data induk, urus akaun juri      |

Untuk `juri`, dialog akan meminta senarai `SportID` (contoh `bd,tt`). Akaun juri
tanpa sukan ditolak — gagal-selamat, supaya tiada juri yang senyap-senyap boleh
memarkah semua acara.

Kata laluan disimpan sebagai salt + SHA-256 berulang 2,500 kali. Tiada kata
laluan teks biasa disimpan di mana-mana.

### 4. Deploy Web App

1. **Deploy → New deployment → Web app**
2. *Execute as*: **Me**
3. *Who has access*: **Anyone**
4. Salin URL `/exec` yang dikeluarkan.

> Setiap kali anda menukar kod, buat **Deploy → Manage deployments → Edit →
> New version**. Jika anda buat deployment baharu, URL berubah dan anda perlu
> mengemas kini frontend.

### 5. Frontend

1. Dalam `web/index.html`, `web/admin/index.html` dan `web/juri/index.html`, ganti:
   ```js
   var SCRIPT_URL = 'GANTI_DENGAN_URL_WEB_APP_ANDA';
   ```
   dengan URL `/exec` tadi. **Ketiga-tiga fail mesti sama.**
2. Push folder `web/` ke repo GitHub, aktifkan GitHub Pages.
3. Kembali ke tab `Config`, tetapkan `PORTAL_URL` kepada URL GitHub Pages anda
   (contoh `https://akaun.github.io/csc2026/`). Ini digunakan untuk kandungan QR.

### 6. Semakan akhir

Jalankan `healthCheck()` (menu **CSC2026 → Semak Kesihatan Sistem**). Ia akan
menyenaraikan apa-apa yang belum ditetapkan.

---

## Tetapan (tab `Config`, atau tab Tetapan dalam portal admin)

| Kunci | Maksud |
|---|---|
| `REG_OPEN` / `REG_CLOSE` | Tempoh pendaftaran, format ISO (`2026-09-01T09:00:00+08:00`). Kosong = tiada had. |
| `MAX_SPORTS` | Maksimum sukan setiap atlet (lalai 2) |
| `MAINTENANCE_MODE` | `TRUE` menyekat semua pendaftaran baharu serta-merta |
| `REQUIRE_SECOND_FACTOR` | **Biarkan FALSE.** Suis kecemasan sahaja — lihat Nota reka bentuk |
| `PUBLIC_STATS` | `TRUE` memaparkan kiraan ringkas di landing page |
| `PHOTO_MAX_KB` | Had saiz gambar di pelayan (frontend memampatkan ke ~150KB) |
| `REG_PREFIX` / `REG_PAD` | Format nombor pendaftaran (`CSC26` + 5 digit) |
| `REG_COUNTER` | **Jangan edit manual semasa sistem live.** |
| `SCOREBOARD_ENABLED` | `FALSE` menyembunyikan papan markah sepenuhnya |
| `SCOREBOARD_PUBLIC` | `FALSE` memerlukan Staff ID yang sah sebelum markah dipapar |
| `SCORE_REQUIRE_VERIFY` | `TRUE` (disyorkan) — markah juri perlu disahkan admin |
| `SCORE_POLL_SECONDS` | Selang semakan papan markah oleh pelayar, minimum 10 |
| `SCORE_VERSION` | Kaunter versi. **Jangan edit manual.** |
| `BRAND_NAME` | Teks jenama di sebelah logo pada nav portal |
| `LOGO_URL` / `LOGO_FILE_ID` | Diisi automatik semasa muat naik logo. **Jangan edit manual.** |

---

## Muat naik senarai induk staf

Tab **Data Induk** → muat naik `.csv` / `.xlsx`, atau tampal CSV terus.

Kolum wajib: `StaffID`, `Name`, `Department`, `Position`, `Contingent`
Kolum pilihan: `Gender` (M/F), `Email`, `Phone`

Aliran dua langkah — **tiada apa-apa ditulis sehingga anda tekan Simpan**:

1. **Sahkan Fail** — mengesan Staff ID kosong/berulang, nama kosong, format emel
   dan telefon tidak sah, jantina tidak sah. Memaparkan ringkasan dan laporan
   ralat yang boleh dimuat turun.
2. **Simpan ke Pangkalan Data** — mod:
   - `upsert` (lalai) — kemas kini yang sedia ada, tambah yang baharu
   - `append` — tambah yang baharu sahaja
   - `replace` — kosongkan dan ganti (superadmin sahaja, perlu pengesahan
     tambahan jika sudah ada pendaftaran)

Butang **Muat Turun Templat CSV** memberi fail permulaan dengan kolum yang betul.

---

## Papan markah & peranan juri

Skrin **Berita** lama diturunkan menjadi kad + ticker; skrin utama kedua
sekarang ialah **Papan Markah** — kedudukan kontinjen, pecahan pingat, trend
mata harian, keputusan terkini dan jadual akan datang.

### Aliran markah

```
juri hantar   ->  DRAF    (tidak kelihatan oleh sesiapa di papan awam)
admin sahkan  ->  SAH     (versi papan markah naik, semua orang nampak)
admin tolak   ->  TOLAK   (kekal dalam rekod untuk audit, tidak dikira)
```

Langkah pengesahan itu sengaja. Satu salah taip pada markah final kelihatan
oleh ratusan orang serta-merta, dan menariknya balik jadi isu kredibiliti,
bukan sekadar isu data. Kalau urus setia mahu markah terus naik tanpa
pengesahan, tetapkan `SCORE_REQUIRE_VERIFY=FALSE`.

### Peranan juri

Akaun juri dicipta dari **Portal Admin → Akaun Juri** (superadmin sahaja) atau
melalui menu spreadsheet **Cipta Pengguna Admin**. Setiap akaun diikat kepada
satu senarai `SportID`. Juri badminton yang cuba menghantar markah bola sepak
ditolak **di pelayan** — menyembunyikan butang di UI bukan kawalan keselamatan.

Tangga peranan: `juri` < `viewer` < `admin` < `superadmin`. Juri sengaja
diletakkan di bawah viewer supaya mereka tidak nampak senarai peserta, laporan
atau log audit.

Juri log masuk di `web/juri/` — halaman ringkas yang direka untuk telefon di
gelanggang:

- Hanya acara dalam skop mereka disenaraikan
- Memilih peserta berdaftar mengisi kontinjen secara automatik
- Mata **dikira di pelayan** daripada kedudukan; nilai mata yang dihantar klien
  tidak pernah dipercayai
- Setiap hantaran dimasukkan ke **baris giliran dalam peranti** sebelum
  permintaan rangkaian dibuat. Talian mati bermakna "3 hantaran menunggu",
  bukan markah yang hilang.
- Setiap hantaran membawa **kunci idempotency**. Tekan dua kali, cuba semula
  automatik, atau muat semula halaman — pelayan tetap merekod satu kali sahaja.
  Tanpa ini, markah dikira dua kali.

### Kenapa papan markah tidak mematikan pelayan

Ini beban yang berbeza sepenuhnya daripada pendaftaran. Pendaftaran ialah 700
tulisan tersebar seminggu; papan markah ialah ratusan **bacaan serentak** dalam
beberapa minit semasa final. Pada 500 penonton menyemak setiap 20 saat, itu 25
permintaan sesaat, sedangkan had pelaksanaan serentak Apps Script sekitar 30.

Empat lapisan menanganinya:

1. **Muatan papan markah dikira sekali dan dicache 20 saat.** 500 penonton =
   tetap satu bacaan spreadsheet setiap 20 saat.
2. **Endpoint `scoreVersion` (kira-kira 40 bait).** Pelayar menyemak nombor
   versi dan hanya menarik muatan penuh apabila ia berubah. Semasa tiada markah
   baharu — iaitu kebanyakan masa — setiap semakan hampir kosong.
3. **Cache berkeping.** `CacheService` menolak nilai melebihi 100KB *secara
   senyap*. Papan markah besar yang tidak pernah dicache bermakna setiap
   penonton membaca spreadsheet — kegagalan senyap yang paling merbahaya di
   sini. Muatan dipecahkan kepada beberapa kunci.
4. **Polling berhenti** apabila tab tersembunyi atau pengguna beralih skrin,
   dan selang dipanjangkan selepas kegagalan berturut-turut.

Selang polling ditetapkan oleh pelayan (`SCORE_POLL_SECONDS`, minimum 10) dan
setiap pelayar menambah jitter rawak supaya 500 peranti tidak mengetuk pada
saat yang sama.

Kalau anda menjangka lebih 500 penonton serentak, langkah seterusnya ialah
menulis snapshot JSON ke fail Drive awam supaya bacaan langsung tidak menyentuh
Apps Script. Struktur muatan sedia ada boleh digunakan terus.

### Ikon PWA & favicon

Ada **dua lapisan** di sini, dan bezanya penting:

| Lapisan | Sumber | Bila berubah |
|---|---|---|
| Ikon pemasangan PWA + favicon lalai | fail statik dalam `web/assets/` | bila anda jana semula dan push |
| Favicon tab + gesaan pasang (belum pasang) | logo yang dimuat naik superadmin | serta-merta |

**Kenapa ikon PWA mesti fail statik.** Sistem pengendalian membaca manifest
dan menyalin ikon **sebelum** sebarang JavaScript berjalan, dan ikon yang
sudah berada di skrin utama pengguna tidak dibaca semula selepas itu. Jadi
logo yang dimuat naik melalui portal admin tidak boleh — dan tidak akan
pernah — menukar ikon pada telefon yang sudah memasang aplikasi. Pengguna itu
perlu buang dan pasang semula.

Sebab itu ikon dijana daripada fail logo rasmi:

```bash
node tools/build-icons-pwa.js tools/logo-source.png
```

Menghasilkan ke `web/assets/`:

| Fail | Guna |
|---|---|
| `icon-192.png`, `icon-512.png` | ikon PWA biasa |
| `icon-maskable-512.png` | pelancar Android bulat — logo dimuatkan dalam zon selamat 66% supaya tidak dipotong |
| `apple-touch-icon.png` | skrin utama iOS |
| `favicon.ico` (16/32/48) + `favicon-16/32/48.png` | ikon tab pelayar |

Favicon menggunakan potongan monogram **CBP** sahaja, bukan logo penuh. Logo
penuh pada 16px menjadi calitan kelabu yang tidak boleh dibaca; bahagian atas
logo masih dikenali pada saiz itu.

Latar belakang ikon ialah navy `#0f172a`. Logo PNG telus di atas latar telus
kelihatan buruk pada kebanyakan pelancar Android, dan garis merah pada tepi
logo memberi kontras yang jelas terhadap navy.

**Lapisan kedua** — apabila logo dimuat naik melalui Tetapan Sistem, portal
menukar favicon tab dan membina semula manifest dalam pelayar. Ini menukar
ikon dalam gesaan "Add to Home Screen" untuk pengguna yang **belum** memasang.
Manifest yang dibina itu mengekalkan ikon statik sebagai sandaran, jadi ia
kekal sah walaupun fail logo dipadam dari Drive kemudian.

Kedua-dua penukaran hanya berlaku **selepas logo disahkan boleh dimuatkan**.
Logo yang rosak tidak menyentuh favicon atau manifest langsung — lebih baik
kekal dengan aset lalai yang berfungsi.

---

### Logo organisasi

**Portal Admin → Tetapan Sistem → Logo Organisasi** (superadmin sahaja). Fail
disimpan dalam folder Google Drive yang **sama** seperti gambar peserta, dalam
subfolder `Sistem`.

Logo dipapar pada nav portal atlet, header portal juri, dan sidebar panel
admin. Kosong = ikon lalai digunakan.

Beberapa keputusan yang menjadikannya tidak menyusahkan kemudian:

- **Diubah saiz di pelayar** kepada 512px sebelum dihantar, dengan nisbah aspek
  dikekalkan (logo bukan gambar profil — tiada pemotongan segi empat sama).
  Muat naik fail 8MB terus ke Apps Script ialah punca paling biasa masa tamat
  pada endpoint jenis ini.
- **PNG kekal PNG** supaya latar telus tidak menjadi kotak putih. Hanya JPEG
  yang dimampatkan berulang.
- **Fail lama dibuang selepas** yang baharu berjaya dimuat naik, bukan sebelum.
  Kalau muat naik gagal separuh jalan, sistem masih ada logo yang berfungsi.
- **`LOGO_URL` tidak boleh ditulis melalui borang Tetapan.** Ia ditulis hanya
  oleh tindakan muat naik/buang, supaya URL dan ID fail Drive sentiasa sepadan
  dan tiada fail yatim terkumpul.
- **Imej yang gagal dimuatkan jatuh balik kepada ikon lalai** di ketiga-tiga
  portal. Kalau fail dipadam dari Drive atau rangkaian venue menyekat
  `googleusercontent`, pengguna nampak ikon biasa, bukan kotak imej pecah.

---

### Kontinjen & jadual mata

`Contingents` menyimpan kod, nama penuh, nama ringkas dan warna. Kod dipadankan
dengan medan `Contingent` dalam `Staff_Master` mengikut **kod, nama penuh ATAU
nama ringkas** — sebab data staf selalunya menyimpan "Kontinjen Alpha"
sedangkan tab kontinjen menyimpan "ALPHA". Tanpa padanan itu setiap kontinjen
muncul dua kali di papan markah.

`Points_Rules` menetapkan mata bagi setiap kedudukan. Baris dengan `SportID = *`
ialah lalai (emas 5, perak 3, gangsa 1); baris khusus sukan mengatasinya —
berguna apabila acara berpasukan patut membawa mata lebih besar. Selepas
mengubah jadual mata, tekan **Kira Semula Semua Mata** supaya markah sedia ada
diselaraskan.

---

## Nota reka bentuk

### Kenapa ia tidak perlahan dengan 700 orang

Halangan sebenar Apps Script bukan saiz data (700 baris adalah kecil untuk
Sheets) tetapi **bilangan pelaksanaan serentak** dan **tempoh kunci dipegang**.
Perkara berikut menanganinya:

- Muatan landing page **dicache 20 saat** dan mengandungi statistik teragregat
  sahaja — bukan senarai peserta.
- Konfigurasi sukan/kategori **dicache 60 saat**.
- Semakan kapasiti di dalam kunci membaca **dua julat kolum sempit** (kira-kira
  3,500 sel) dan bukan keseluruhan 26 kolum (18,000 sel) — kira-kira 8x lebih
  pantas dalam ujian.
- Muat naik gambar berlaku **sebelum** kunci diambil, bukan sambil memegangnya.
- Penulisan audit berlaku **selepas** kunci dilepaskan.
- Gambar dimampatkan di pelayar (dipotong segi empat sama, 600px, JPEG) —
  turun daripada beberapa MB kepada kira-kira 100-200KB.
- Frontend mencuba semula dengan backoff eksponen, jadi lonjakan sementara tidak
  menyebabkan pengguna menekan Hantar berulang kali.

Kiraan yang dicache boleh lapuk beberapa saat. Itu selamat: penghantaran sebenar
mengesahkan semula kapasiti di dalam kunci, jadi paparan lapuk paling teruk
menyebabkan mesej "sudah penuh", bukan pendaftaran berlebihan.

### Nombor pendaftaran

Berurutan daripada kaunter dalam tab `Config`, dijana sambil memegang
`LockService` dan disahkan unik sebelum ditulis. Kaunter dibaca terus daripada
sheet, memintas cache konfigurasi.

### Kod QR

Dijana **sepenuhnya dalam pelayar** oleh encoder terbenam (`QR` dalam
`index.html`) — tiada CDN, jadi ia berfungsi walaupun rangkaian venue menyekat
skrip luar. Kandungan QR ialah URL portal + token rawak 24 aksara. Tiada nama,
Staff ID atau maklumat peribadi dalam kod. Mengimbas hanya membawa ke portal;
pengguna masih perlu memasukkan Staff ID.

### Keselamatan

- Setiap tindakan admin, termasuk bacaan, melalui **POST** dengan token sesi
  dalam badan permintaan — bukan dalam URL.
- Peranan dikuatkuasakan di pelayan pada setiap permintaan. Penyembunyian
  butang di frontend hanyalah kemudahan.
- Log masuk dikunci 15 minit selepas 5 percubaan gagal.
- Setiap mutasi pentadbiran ditulis ke `Audit_Log` dengan sebelum/selepas,
  pelaku, cap masa dan sebab.
- Service worker tidak pernah mencache respons API.

### Log masuk atlet: satu tetapan, dua mod

Cara atlet log masuk dikawal oleh `REQUIRE_SECOND_FACTOR` dalam **Tetapan
Sistem**. Boleh ditukar bila-bila masa; berkuat kuasa serta-merta tanpa deploy
semula.

| Tetapan | Yang atlet buat | Bila sesuai |
|---|---|---|
| `FALSE` (lalai) | Masukkan Staff ID, tekan Log Masuk, terus masuk | Senarai Staff ID tidak beredar luas; utamakan kelajuan |
| `TRUE` | Staff ID, kemudian 4 digit akhir no. telefon dari rekod induk | Senarai Staff ID mudah didapati; utamakan privasi |

Kedua-duanya sah. Pada mod `FALSE`, satu medan bermakna tiada apa-apa untuk
diingat oleh 700 orang dan panggilan ke urus setia jauh berkurang. Pada mod
`TRUE`, satu medan tambahan menutup sebahagian besar risiko orang menyelongkar
pas orang lain — nombor telefon datang dari `Staff_Master`, jadi tiada apa-apa
untuk diedar dahulu.

Menukarnya tidak memerlukan perubahan kod dan tidak menjejaskan pendaftaran
yang sudah masuk.

**Apa yang terdedah pada mod `FALSE`.** Sesiapa yang meneka Staff ID orang
lain akan melihat pas sukan mereka: nama, kontinjen, sukan yang disertai,
gambar, nombor pendaftaran. Itu maklumat yang sama seperti yang tertera pada
bib di padang.

Yang **tidak** dihantar langsung kepada pemanggil yang hanya menaip Staff ID:

- nombor telefon
- alamat emel
- nama dan nombor waris kecemasan

Ini dikuatkuasakan di pelayan oleh `athleteSafeRegistration_()`, bukan sekadar
disembunyikan di paparan. Ujian memeriksa muatan yang dikembalikan secara
literal untuk memastikan medan-medan ini tidak muncul. Urus setia tetap melihat
medan penuh melalui endpoint admin yang memerlukan sesi.

Skrin pas tidak pernah memaparkan medan itu, jadi menanggalkannya tidak
menghilangkan apa-apa fungsi.

**Had kadar** memperlahankan pengumpulan senarai penuh: 25 carian setiap Staff
ID sejam, 60 setiap peranti sejam, 600 seluruh sistem seminit. Had setiap
peranti sengaja tidak diketatkan lagi — kaunter pendaftaran di venue selalunya
satu komputer dikongsi berpuluh orang, dan had yang terlalu ketat akan menyekat
pengguna sah pada waktu paling sibuk.

Kedua-dua perkara di atas — muatan yang dikecilkan dan had kadar — berkuat
kuasa dalam **mana-mana** mod. Menghidupkan faktor kedua menambah lapisan; ia
tidak menggantikannya.

---

## Penyelenggaraan

- **Sesi luput** — jalankan `purgeExpiredSessions()`, atau tetapkan trigger
  harian untuknya.
- **Log audit** — bertumbuh secara berterusan. Arkibkan ke Sheet berasingan
  setiap beberapa bulan supaya tab kekal pantas.
- **Emel** — Gmail biasa terhad kepada 100 emel/hari; akaun Workspace 1,500.
  Sahkan sebelum bergantung pada pengumuman emel untuk 700 orang.
- **Reset** — `DANGER_resetAllData('SAYA FAHAM PADAM SEMUA')` mengosongkan
  pendaftaran, markah, check-in dan audit, serta me-reset kaunter. Tab `Events`,
  `Contingents` dan `Points_Rules` TIDAK dipadam — itu konfigurasi carnival,
  bukan data pendaftaran. Tiada cara lain untuk memadam secara pukal, dengan
  sengaja.
- **Markah tidak berubah di telefon staf** — tekan **CSC2026 → Pemarkahan →
  Segarkan papan markah sekarang**. Ini diperlukan hanya jika anda mengedit tab
  `Results` terus dalam Sheet, kerana suntingan manual tidak menaikkan versi.

---

## Menyelesaikan masalah

| Gejala | Punca berkemungkinan |
|---|---|
| "Gambar profil gagal dimuat naik" pada setiap pendaftaran | `DRIVE_FOLDER_ID` tidak ditetapkan atau folder tiada akses |
| Semua permintaan gagal dari GitHub Pages | Deployment ditetapkan *Who has access: Anyone*? URL `/exec` betul dalam KEDUA-DUA fail HTML? |
| "Sesi tamat" serta-merta selepas log masuk | Jam pelayar/pelayan berbeza jauh, atau deployment ditukar tanpa versi baharu |
| Kod QR tidak muncul | Token QR kosong pada baris tersebut — periksa kolum `QRToken` |
| Perubahan kod tidak berkesan | Anda perlu buat **versi deployment baharu**, bukan sekadar simpan |
| Nombor pendaftaran melompat | Normal — nombor dilepaskan apabila pendaftaran ditolak selepas kaunter bertambah |
| Juri hantar markah tetapi papan markah kosong | Markah masih DRAF — sahkan di Portal Admin → Pengesahan Markah |
| Juri nampak "Tiada Acara" | Belum ada acara untuk sukan dalam skopnya — cipta di Jadual Acara |
| Kontinjen muncul dua kali di papan markah | Nama dalam `Staff_Master` tidak sepadan mana-mana kod/nama/nama ringkas dalam `Contingents` |
| Papan markah tidak kemas kini walaupun sudah disahkan | Tab pelayar tersembunyi (polling berhenti); buka semula tab, atau tunggu selang berikutnya |

---

## Status ujian

Backend disahkan terhadap tiruan Apps Script (159 ujian) dan frontend melalui
Playwright terhadap backend sebenar (77 + 13 + 13 ujian, UI asal):

- 700 pendaftaran menghasilkan 700 nombor unik, tiada perlanggaran
- Kapasiti, had 2 sukan, kelayakan jantina dan perakuan semuanya dikuatkuasakan
  di pelayan, bukan hanya di UI
- Kegagalan muat naik gambar membatalkan pendaftaran dan membuang fail yatim
- `setupSystem()` yang dijalankan berulang kali tidak memadam data
- Muat naik data induk tidak menulis apa-apa semasa pengesahan
- Endpoint awam tidak membocorkan data peribadi
- Carian Staff ID tidak memulangkan telefon, emel atau butiran waris kecemasan
  — muatan diperiksa secara literal, bukan hanya medan demi medan
- Kod QR yang dirender pada skrin telefon berjaya diimbas semula
- Portal admin tidak memuatkan sebarang data ke dalam DOM sebelum log masuk
- Semua 700+ rekod dimuatkan ke portal admin (bukan hanya 200 pertama)
- Setiap kelas Tailwind dalam DOM hidup wujud dalam CSS terkumpul
- Setiap ikon dirender sebagai SVG sebenar pada semua skrin (0 kosong, 0 sisa)
- Butang Kembali hadir pada setiap skrin dalam; pas sukan ada jalan balik ke Utama
- `seedDummyData()` tidak mengubah walau satu baris data sebenar; `removeDummyData()`
  memulihkan kiraan tepat kepada asal
- Markah DRAF tidak pernah muncul dalam muatan papan markah awam, termasuk
  kiraannya
- Juri di luar skop ditolak di pelayan pada senarai acara, butiran acara dan
  penghantaran markah
- Kunci idempotency yang sama dihantar dua kali menghasilkan satu baris, bukan dua
- Markah yang dihantar semasa talian mati disimpan dalam peranti dan dihantar
  semula automatik apabila talian pulih
- Mata dikira semula di pelayan; nilai mata yang dihantar klien diabaikan
- 500 permintaan papan markah berturut-turut menghasilkan sifar bacaan
  spreadsheet tambahan
- Cache berkeping memulangkan muatan 260KB dengan tepat, dan menganggap
  kepingan yang hilang sebagai cache-miss dan bukan JSON separa
- Mengesahkan markah menaikkan versi, dan pelayar atlet mengambil versi baharu
  itu pada semakan seterusnya
- Menghidupkan dan mematikan `REQUIRE_SECOND_FACTOR` dari borang Tetapan
  benar-benar mengubah aliran log masuk atlet, dalam kedua-dua arah
- Menyimpan borang Tetapan tidak pernah mematikan papan markah secara tidak
  sengaja (medan kosong diabaikan)
- Muat naik logo menulis ke Drive, membuang fail lama, dan sampai ke ketiga-tiga
  portal; `LOGO_URL` tidak boleh ditulis melalui borang Tetapan
- Logo yang gagal dimuatkan jatuh balik kepada ikon lalai di ketiga-tiga portal
- Setiap fail ikon PWA wujud, boleh dicapai, dan disenaraikan dalam manifest
- Ikon maskable mempunyai zon selamat sebenar — sudut ialah latar belakang,
  bahagian tengah ada kandungan (disemak piksel demi piksel)
- `favicon.ico` mengandungi ketiga-tiga saiz 16/32/48
- Ketiga-tiga portal mengisytiharkan favicon sebelum JavaScript berjalan
- Logo yang dimuat naik menggantikan favicon dan membina manifest baharu yang
  boleh dihurai, dengan ikon statik dikekalkan sebagai sandaran
- Logo rosak tidak menyentuh favicon mahupun manifest
