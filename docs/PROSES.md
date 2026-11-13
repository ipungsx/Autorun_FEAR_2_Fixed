# Proses analisis dan perbaikan F.E.A.R. 2 di Autorun 258

Dokumen ini mencatat urutan analisis dari awal sampai akhir: gejala yang diamati, bukti dari log, akar masalah, perbaikan yang dibuat, dan hasil pengujian. Jalan yang ternyata salah juga dicatat, karena alasan sebuah hipotesis ditinggalkan sama pentingnya dengan hipotesis yang akhirnya terbukti.

## Lingkungan

| Komponen | Versi |
|----------|-------|
| Perangkat | Nintendo Switch |
| Runtime | Autorun 258, runtime build `nx-amd64-fex-2640`, Wine 11.18 untuk Horizon, WoW64 |
| Game | F.E.A.R. 2: Project Origin, `FEAR2.exe` 32-bit (modul mesin bernama `Engine.exe`) |
| Grafis | DXVK Sarek 1.14.1 (D3D9) |
| Emulasi CPU | Box64 dan FEX (keduanya dicoba) |
| Sumber DLL | `autorunhq/autorun` commit `1ac9b6cf` |
| Toolchain | llvm-mingw 20260505 (UCRT) |

Bahan analisis:
- Log `FEAR2.log` dari Autorun (`sdmc:/switch/wine/logs/`), baik mode biasa maupun Verbose.
- Disassembly `FEAR2.exe`, `GameClient.dll`, dan `binkw32.dll` dengan `llvm-objdump`, hanya dibaca dan tidak diubah.
- Kode sumber Autorun, termasuk riwayat git-nya.

## Tahap 1. Crash di XAudio2

**Gejala.** Game langsung crash. Log mencatat page fault di alamat `00000000` dengan instruksi di `xaudio2_3.dll+0x16443`.

**Analisis.** Offset itu berada di `FAudio_AudioClientThread` (`libs/faudio/src/FAudio_platform_win32.c`). Thread audio meminta `IAudioRenderClient` dari `IAudioClient::GetService`, tetapi hasilnya tidak diperiksa: `FAudio_assert` tidak aktif pada build rilis. Saat game menutup perangkat audio, thread memanggil `Release()` pada pointer NULL.

**Perbaikan bertahap.**
1. **v1** hanya mencegah crash. Bila render client tidak ada, thread audio diam. Hasilnya game macet di layar loading, karena XAudio2 tidak pernah mengirim callback `OnBufferEnd` yang ditunggu game.
2. **v2**: bila render client tidak ada, mixer XAudio2 tetap berjalan menurut waktu nyata (berbasis `QueryPerformanceCounter`, mengejar paling banyak 0,25 detik), lalu hasilnya dibuang. Semua callback tetap terkirim.
3. **v3 (akar masalah)**: log v2 menunjukkan F.E.A.R. 2 meminta audio 6 kanal, 48000 Hz, float, dan `IAudioClient::Initialize` ditolak dengan `AUDCLNT_E_UNSUPPORTED_FORMAT` (`0x88890008`) oleh driver audio Autorun (`horizon-wine/source/audio_unix.c`). Kini, bila format multikanal ditolak, perangkat dibuka ulang dalam stereo float. Mesin XAudio2 tetap mencampur 6 kanal seperti permintaan game, lalu hasilnya dilipat ke stereo:
   - depan kiri dan kanan langsung;
   - tengah 0,707 ke kiri dan ke kanan;
   - belakang dan samping 0,707 ke sisinya masing-masing;
   - LFE dibuang;
   - keluaran dijepit pada rentang ±1.

Log yang menandakan fallback berjalan (ini bukan error):

```
err:xaudio2:FAudio_PlatformInit device refused 6 channels; mixing 6 channels and folding them down to stereo
```

Patch: `patches/0001-faudio-xaudio2-refused-stream-stereo-fold.patch`.

## Tahap 2. Layar hitam dan pilihan DXVK

**Gejala.** Dengan DXVK 1.10.3 maupun DXVK resmi 3.1.1, layar hitam atau crash. Log DXVK mencatat `Reset` gagal dengan `Remaining resources: 7`.

**Keputusan.** Opsi `d3d9.countLosableResources = False` dapat mengatasi kegagalan ini. Tetapi `dxvk.conf` di folder game ternyata tidak dibaca oleh Autorun 258, sehingga pendekatan itu tidak bisa dipakai. DXVK Sarek tidak mengalami kegagalan `Reset` ini, maka Sarek ditetapkan sebagai pilihan.

## Tahap 3. Loading tanpa akhir dan "Sound Acceleration Error"

**Gejala.** Animasi loading berputar terus dengan beban CPU rendah.

**Jalan yang ditinggalkan.**
- *Urutan memori di emulasi CPU.* `BOX64_DYNAREC_STRONGMEM=3` lewat `FEAR2.box64.txt` tidak mengubah apa pun.
- *Thread pekerja Scaleform berhenti terlalu dini.* Patch diagnostik `msvcrt` (`patches/diagnostics/msvcrt-beginthreadex-trace.patch`) mencatat alamat awal dan nilai kembali setiap thread. Hasilnya, fungsi `Run` milik loader (`FEAR2.exe+0x141980`) memang selalu mengembalikan 1, jadi perilakunya normal.

**Petunjuk.** Loading selalu berhenti tepat saat video Bink (`m01.vib`) dibuka. Sebagai uji, folder `Game\interface\videos` diganti nama. Game lalu menampilkan dialog:

> **Sound Acceleration Error**
> Please refer to the readme.txt for instructions on setting up the sound hardware on your computer.

**Analisis disassembly.**
1. `GameClient.dll` menampilkan dialog itu (string `IDS_UNABLETOINITSOUND_HEADER`) bila fungsi penyala suara milik mesin mengembalikan kode `0x23`.
2. Di `FEAR2.exe` ada tiga jalur yang mengembalikan kode itu. Jalur XAudio2 (`GetDeviceDetails`, `CreateMasteringVoice`, `CreateSubmixVoice` dengan reverb) lolos, sesuai log. Jalur yang gagal ada di fungsi `0x6099aa`: mesin mendata driver ACM dengan `acmDriverEnum` dan mewajibkan tiga tag format, yaitu `0x11` (IMA ADPCM), `0x55` (MPEG Layer-3), dan `0x01` (PCM). Bila salah satunya tidak ada, sistem suara dinyatakan gagal.
3. Berkas codec bawaan Wine (`imaadp32.acm`, `msadp32.acm`, dan lainnya) sudah ada di `syswow64`. Tetapi program setup Autorun (`horizon-wine/tools/autorun_setup.c`) hanya mendaftarkan `msacm.l3acm` di registry, dan Autorun tidak menyertakan `C:\windows\system.ini`. Log mengonfirmasi: hanya `l3codeca.acm` yang pernah dimuat, `imaadp32.acm` tidak pernah dimuat.

**Perbaikan.** `C:\windows\system.ini` dengan bagian `[drivers32]` yang mendaftarkan `imaadp32.acm`, `msadp32.acm`, `msg711.acm`, dan `msgsm32.acm`. Codec MP3 tidak dicantumkan karena sudah terdaftar di registry. `msacm32` membaca kedua sumber itu.

**Hasil.** Game berhasil masuk ke gameplay, sekitar 50 fps.

Tafsiran: dengan sistem suara gagal menyala secara diam-diam, pemutaran video Bink pada layar loading tidak pernah selesai. Jadi loading tanpa akhir kemungkinan besar merupakan gejala dari masalah codec yang sama. Ini tafsiran dari urutan kejadian dan belum dibuktikan secara langsung.

## Tahap 4. Kontrol

**Temuan dari biner.** `FEAR2.exe` dan `GameClient.dll` tidak memuat XInput maupun DirectInput, dan tidak memakai raw input. Keyboard dibaca melalui pesan jendela dan `GetKeyboardState`. Kait `WH_KEYBOARD_LL` hanya dipakai untuk memblokir tombol Windows. Mouse dibaca melalui `WM_MOUSEMOVE`, dengan `SetCursorPos` ke tengah area klien setiap frame. Akibatnya, mode "Controller" di Autorun tidak akan pernah berfungsi untuk game ini.

**Temuan dari runtime.** Teks log `Autorun's controls, not its own` dan `FEAR2.keys.txt: 25 controls remapped` hanya ada di kode Autorun sampai sekitar 3 Oktober 2026. Jadi runtime pada perangkat lebih tua daripada kode sumber terbaru, walaupun nomor build-nya sama-sama 258. Pada versi itu:
- Berkas kontrol per game berformat `NAMA=kode` (misalnya `LUP=0x57`, `RSTICK=mouse`) dan hanya dibaca bila pengaturan **Controls** bernilai **Its own**.
- Kontrol bawaan membuat analog kiri mengirim tombol **panah**, mengikuti D-pad. F.E.A.R. 2 berjalan dengan W A S D, sehingga analog kiri tampak mati.

Profil kontrol pertama sempat ditulis dalam format versi terbaru (`key.LUP=...`), dan tidak terbaca oleh runtime tersebut. Profil di repositori ini memakai format lama.

**Hasil.** Log mencatat `FEAR2.keys.txt: 21 controls remapped`, dan penekanan tombol W, A, S, D, C, G, Shift, serta Esc tercatat sampai ke game.

## Tahap 5. Kamera dan analog kanan

**Gejala.** Analog kanan tidak menggerakkan kamera. Sesekali kamera melompat atau berputar liar.

**Pengukuran.** Patch diagnostik `user32` (`patches/diagnostics/user32-cursor-warp-trace.patch`) mencatat posisi kursor setiap kali game memanggil `SetCursorPos`. Dalam sekitar 2,5 menit gameplay:
- game memusatkan kursor ke `640,360` sebanyak 6858 kali, kira-kira sekali per frame;
- runtime mencatat gerakan stick (`[NXINPUT] the cursor is at 640,360, the stick moved 4,0`), tetapi kursor Windows tetap di `640,360`;
- hanya 4 kali kursor ditemukan bergeser, dan setiap kali sejauh ratusan piksel sekaligus (`+288,+129`, `-53,+247`, `-80,+238`, `-217,+144`).

**Percobaan pertama: menahan pemusatan kursor.** Khusus untuk `FEAR2.exe`, `SetCursorPos` tidak diteruskan, kursor dibiarkan bebas, dan pesan mouse ditulis ulang ke posisi yang diharapkan game.
- *Versi pertama*: kamera berputar pelan ke satu arah. Penyebabnya, di Windows setiap `SetCursorPos` menghasilkan `WM_MOUSEMOVE` di titik tujuan. F.E.A.R. 2 menyimpan posisi mouse terakhir dan terus memutar kamera sejauh jaraknya dari tengah sampai menerima posisi baru.
- *Versi kedua*: pesan itu dikirim sendiri sebagai `WM_MOUSEMOVE` yang diberi tanda `0x8000` pada `wParam`. Tanda itu dihapus sebelum pesan sampai ke game. Kamera menjadi diam. Tetapi dalam satu sesi penuh, game hanya menerima 7 pesan gerakan mouse, salah satunya lompatan `-795` piksel tepat setelah tombol Esc ditekan.

**Akar masalah.** Wine 11.18 memperkenalkan `accum_mouse_motion` di `dlls/win32u/input.c`. Gerakan mouse relatif dari driver dikumpulkan per thread dan baru dikirim ke server bila ada masukan lain:
- tombol mouse atau roda;
- `MOUSEEVENTF_MOVE_NOCOALESCE`;
- pergantian jendela.

Driver tampilan Switch (`dlls/win32u/winnx_drv.c`) pada runtime perangkat tidak pernah mengirim pemicu itu. Pengembang Autorun memperbaikinya pada commit [`0cc936d1`](https://github.com/autorunhq/autorun/commit/0cc936d1863209af659977cdb001c8c6fdde94cc) ("win32u: flush accumulated mouse motion in the Switch display driver", 12 Oktober 2026). Runtime di perangkat dibuat sebelum commit itu.

Akibatnya, gerakan analog kanan tertahan sampai ada klik atau sentuhan, lalu terkirim sekaligus sebagai satu lompatan. Ini menjelaskan semua gejala kamera sejak awal.

**Perbaikan (`user32.dll`).** Driver ada di sisi unix runtime dan tidak dapat dibangun ulang tanpa toolchain Switch. Karena itu perbaikannya dikerjakan dari sisi DLL:
1. Setelah setiap `PeekMessageW` dan `GetMessageW`, `user32` mengirim `MOUSEEVENTF_MOVE_NOCOALESCE` melalui `NtUserCallHwndParam(..., NtUserCallHwndParam_SendHardwareInput)`. Itu panggilan yang sama dengan yang dipakai driver, dan dengan begitu gerakan yang terkumpul terkirim.
   - Thunk WoW64 meneruskan pointer `INPUT` tanpa konversi, sehingga struktur `INPUT` disusun dalam tata letak 64-bit (40 byte).
   - Pengiriman dibatasi paling sering sekali per 4 ms per thread.
   - Bila tidak ada gerakan yang terkumpul, Wine tidak mengirim apa pun.
   - Bagian ini berlaku untuk semua program 32-bit.
   - Penomoran `NtUserCallHwndParam` diperiksa tidak berubah sejak pembaruan ke Wine 11.18, sehingga cocok dengan runtime perangkat.
2. Khusus `FEAR2.exe`, penanganan pemusatan kursor dari percobaan di atas tetap dipakai:
   - kursor dikembalikan ke tengah layar hanya bila mendekati tepi;
   - lompatan karena pengembalian itu tidak dihitung sebagai gerakan;
   - penanganan nonaktif sendiri bila game berhenti memusatkan kursor selama 300 ms, misalnya saat kembali ke menu.
3. Khusus `FEAR2.exe`, input yang diterima game dicatat ke log untuk keperluan diagnosis.

Patch: `patches/0002-user32-mouse-motion-flush-and-fear2-mouselook.patch`.

**Status.** Belum dikonfirmasi pada perangkat. Baris log yang diharapkan:

```
err:win:autorun_mouse_flush flushing the display driver's collected mouse motion
err:win:SetCursorPos mouse look on: program warps to 640,360, ...
err:win:ml_log_input input: mouse at X,Y at T (look on)
```

## Catatan metodologis

- Semua perubahan dibuat sebagai DLL PE yang dapat dipasang dan dicabut sendiri. Tidak ada berkas game yang diubah.
- Setiap DLL dikirim bersama salinan DLL resmi sebagai cadangan, sehingga setiap uji dapat dibatalkan.
- Diagnostik dibuat sebagai DLL terpisah yang hanya mencatat, sebelum perbaikan sebenarnya ditulis. Hipotesis yang tidak didukung data, misalnya urutan memori dan thread pekerja, ditinggalkan setelah diuji, bukan dipertahankan.
- Nomor build Autorun tidak berubah antarversi. Karena itu versi runtime di perangkat ditentukan dengan mencocokkan teks log terhadap riwayat git Autorun, dan format berkas kontrol mengikuti versi tersebut.
