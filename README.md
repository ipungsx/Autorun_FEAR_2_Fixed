# Autorun F.E.A.R. 2 Fixed

Kumpulan perbaikan agar **F.E.A.R. 2: Project Origin** (versi PC, 32-bit) dapat dimainkan di Nintendo Switch melalui [Autorun](https://github.com/autorunhq/autorun) 258, yaitu runtime Windows berbasis Wine untuk Horizon.

Repositori ini **tidak** memuat file game. Isinya hanya DLL Wine yang dibangun ulang dari kode sumber Autorun, berkas konfigurasi, patch kode sumber, dan dokumentasi proses perbaikannya.

> English summary: fixes that get F.E.A.R. 2: Project Origin (32-bit PC) running on Autorun 258 for Nintendo Switch. It contains rebuilt Wine DLLs (xaudio2_3, user32), an ACM codec registration (`system.ini`), a control profile and the source patches. See [English notes](#english-notes) below.

## Ringkasan masalah dan perbaikan

| No | Gejala | Akar masalah | Perbaikan | Status |
|----|--------|--------------|-----------|--------|
| 1 | Game crash di `xaudio2_3.dll` saat audio dimulai | Driver audio Autorun 258 menolak stream 5.1 (6 kanal), lalu FAudio memanggil `Release()` pada pointer NULL | `xaudio2_3.dll` baru: bila format multikanal ditolak, perangkat dibuka stereo dan hasil campuran 6 kanal dilipat ke stereo | Teruji |
| 2 | Layar hitam atau crash dengan DXVK selain Sarek | `IDirect3DDevice9::Reset` gagal di DXVK 1.10.3 dan DXVK resmi ("Remaining resources: 7") | Gunakan **DXVK Sarek** di pengaturan Autorun | Teruji |
| 3 | Dialog "Sound Acceleration Error", lalu layar hitam lama | Codec ACM IMA ADPCM tidak terdaftar, sedangkan game mewajibkan IMA ADPCM, MP3, dan PCM | `C:\windows\system.ini` yang mendaftarkan codec ACM bawaan Wine | Teruji, game masuk gameplay |
| 4 | Analog kiri dan kanan tidak berfungsi | F.E.A.R. 2 versi PC tidak membaca gamepad sama sekali; format file kontrol Autorun 258 berbeda dengan versi terbaru | `FEAR2.keys.txt` (format lama) dengan analog kiri = W A S D, analog kanan = mouse | Teruji untuk analog kiri dan tombol |
| 5 | Analog kanan tidak menggerakkan kamera, kamera sesekali melompat atau berputar sendiri | Wine 11.18 mengumpulkan gerakan mouse relatif, dan driver tampilan Autorun 258 tidak mengirimkannya sampai ada klik | `user32.dll` baru: mengirim gerakan yang terkumpul (semua program 32-bit) dan menangani pemusatan kursor khusus `FEAR2.exe` | **Belum teruji di perangkat** |

Penjelasan lengkap tiap langkah, bukti dari log, dan alasan setiap keputusan ada di [docs/PROSES.md](docs/PROSES.md).

## Isi repositori

```
sdcard/                         salin isi folder ini ke root SD card
  switch/wine/drive_c/
    windows/system.ini          pendaftaran codec ACM
    windows/syswow64/xaudio2_3.dll
    windows/syswow64/user32.dll
    FEAR 2 Project Origin Complete/FEAR2.keys.txt
patches/                        perubahan kode sumber terhadap Autorun
  0001-faudio-xaudio2-refused-stream-stereo-fold.patch
  0002-user32-mouse-motion-flush-and-fear2-mouselook.patch
  diagnostics/                  patch diagnostik yang dipakai selama analisis
docs/PROSES.md                  dokumentasi proses analisis dan perbaikan
docs/BUILD.md                   cara membangun ulang DLL dari kode sumber
licenses/                       lisensi Wine (LGPL 2.1) dan FAudio (zlib)
CREDITS.md                      kredit
CHECKSUMS.sha256                checksum seluruh berkas di sdcard/
```

## Cara memasang

1. Matikan Autorun.
2. Salin folder `sdcard/switch` ke **root** SD card dan timpa berkas yang ada.
   - Bila `sdmc:/switch/wine/drive_c/windows/system.ini` sudah ada, jangan ditimpa. Tambahkan baris di bawah bagian `[drivers32]` dari berkas di repositori ini.
   - `FEAR2.keys.txt` harus berada **di samping `FEAR2.exe`**. Bila nama folder game Anda berbeda dari `FEAR 2 Project Origin Complete`, pindahkan berkas itu ke folder yang benar. Bila di sana sudah ada `FEAR2.keys.txt`, ubah dulu namanya sebagai cadangan.
3. Di Autorun, buka pengaturan F.E.A.R. 2:
   - **DXVK**: pilih **Sarek**.
   - **Controls** (bagian Diagnostics): pilih **Its own**, agar `FEAR2.keys.txt` dibaca.
4. Jangan mengganti nama folder `Game\interface\videos`. Video Bink dibiarkan apa adanya.
5. Jalankan game.

Jangan menjalankan **Settings > System > Windows DLLs** atau **Verify files** setelah memasang. Keduanya mengunduh ulang DLL resmi dan menimpa `xaudio2_3.dll` serta `user32.dll` dari repositori ini. Untuk **kembali ke kondisi resmi**, justru jalankan **Verify files**, lalu hapus `system.ini` bila perlu.

### Pemetaan kontrol (`FEAR2.keys.txt`)

| Kontrol Switch | Masukan ke game |
|----------------|-----------------|
| Analog kiri | W A S D |
| Analog kanan | Gerakan mouse (kamera) |
| A / B | Klik kiri (tembak) / klik kanan |
| ZR / ZL | Space (lompat) / C (jongkok) |
| X / Y | E (pakai) / R (reload) |
| L / R | G (granat) / Shift |
| Klik analog kiri / kanan | Ctrl / F |
| D-pad | 1, 2, 3, 4 (senjata) |
| + / - | Esc / Tab |

Fungsi tombol di dalam game dapat disesuaikan melalui **Options > Controls** milik F.E.A.R. 2. Format berkas: `NAMA=kode`, dengan kode berupa *virtual-key code* Windows dalam heksadesimal. `LSTICK` dan `RSTICK` menerima `keys` atau `mouse`.

## Dampak terhadap game lain

- `xaudio2_3.dll`: berlaku untuk semua game 32-bit yang memakai XAudio2 2.3. Perilakunya hanya berubah bila driver audio menolak format multikanal. Pada kondisi itu suara diturunkan ke stereo, alih-alih crash atau macet.
- `system.ini`: mendaftarkan codec yang berkasnya sudah ada di Autorun. Game lain yang membutuhkan codec tersebut ikut terbantu.
- `user32.dll`:
  - Pengiriman gerakan mouse yang terkumpul berlaku untuk **semua program 32-bit**. Bila tidak ada gerakan yang terkumpul, tidak ada yang dikirim.
  - Penanganan pemusatan kursor dan pencatatan input ke log **hanya aktif untuk `FEAR2.exe`**.
- Program 64-bit tidak terpengaruh, karena memakai `system32\user32.dll` (ARM64X).

## Batasan

- Perbaikan kamera (nomor 5) belum dikonfirmasi pada perangkat. Laporan hasil pengujian sangat diharapkan melalui *Issues*.
- Versi Autorun yang lebih baru sudah memuat perbaikan resmi untuk pengiriman gerakan mouse (commit [`0cc936d1`](https://github.com/autorunhq/autorun/commit/0cc936d1863209af659977cdb001c8c6fdde94cc), 12 Oktober 2026). Bila Autorun Anda sudah memuat commit tersebut, `user32.dll` dari repositori ini tidak diperlukan untuk bagian itu.
- DLL dibangun dari kode sumber Autorun commit `1ac9b6cf` (16 Oktober 2026) dan diuji pada Autorun 258 dengan runtime build `nx-amd64-fex-2640`. Kombinasi lain belum diuji.
- F.E.A.R. Perseus Mandate dan F.E.A.R. Extraction Point berada di luar cakupan repositori ini.

## English notes

1. **Audio crash**: Autorun 258's audio driver rejects F.E.A.R. 2's 6-channel float stream (`AUDCLNT_E_UNSUPPORTED_FORMAT`). FAudio then released a NULL `IAudioRenderClient`. The patched `xaudio2_3.dll` retries in stereo and folds the 6-channel mix down. If no render client is available at all, it keeps mixing in real time into a silent buffer, so voice callbacks still fire.
2. **Black screen**: use DXVK Sarek. D3D9 `Reset` fails in the other bundled DXVK builds.
3. **"Sound Acceleration Error"**: the engine refuses to start sound unless ACM drivers for IMA ADPCM (0x11), MPEG Layer-3 (0x55) and PCM are present. Autorun only registers the MP3 codec, so `system.ini` registers Wine's built-in codecs under `[drivers32]`.
4. **Controls**: the PC game reads only keyboard and mouse. This Autorun build reads `FEAR2.keys.txt` (`NAME=code`) only when *Controls* is set to *Its own*.
5. **Camera**: Wine 11.18 coalesces relative mouse motion, and this Autorun build's display driver never flushes it. The patched `user32.dll` flushes it with `MOUSEEVENTF_MOVE_NOCOALESCE` for every 32-bit program. For `FEAR2.exe` only, it also keeps the game's per-frame `SetCursorPos` warps from moving the real cursor and rewrites mouse messages to match. **Not yet confirmed on hardware.**

## Lisensi

DLL di `sdcard/` adalah turunan Wine dan FAudio. Wine berlisensi GNU LGPL 2.1 (lihat `licenses/Wine-LGPL-2.1.txt`), FAudio berlisensi zlib (lihat `licenses/FAudio-zlib.txt`). Kode sumber lengkap tersedia di [autorunhq/autorun](https://github.com/autorunhq/autorun) commit `1ac9b6cf37b90f2d5783dd7ef389bebbe1c68877`, ditambah patch di folder `patches/`. Cara membangunnya dijelaskan di [docs/BUILD.md](docs/BUILD.md).

F.E.A.R. dan F.E.A.R. 2: Project Origin adalah merek dagang milik pemiliknya masing-masing. Repositori ini tidak berafiliasi dengan pengembang maupun penerbit game, dan tidak mendistribusikan berkas game apa pun.

Kredit lengkap ada di [CREDITS.md](CREDITS.md).
