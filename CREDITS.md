# Kredit

## Perbaikan, pengujian, dan dokumentasi

- **ipungsx**: analisis, perbaikan, pengujian di Nintendo Switch, dan dokumentasi repositori ini.

## Proyek yang menjadi dasar

- **[Autorun](https://github.com/autorunhq/autorun)** (autorunhq) beserta para kontributornya, termasuk danfromtico, naga, schiminsky, Aleksandr Semenenko, isrmicha, nebadasSwifty, dan yubikong. Mereka membuat runtime Wine untuk Horizon, driver tampilan dan input Switch, serta paket DLL `horizon-dlls`. Semua DLL di repositori ini dibangun dari kode sumber Autorun.
- **Aleksandr Semenenko ([@Deskri](https://github.com/Deskri))**: menulis perbaikan resmi Autorun [`0cc936d1`](https://github.com/autorunhq/autorun/commit/0cc936d1863209af659977cdb001c8c6fdde94cc), "win32u: flush accumulated mouse motion in the Switch display driver". Perbaikan kamera di `user32.dll` meniru cara kerja commit tersebut dari sisi DLL, untuk runtime Autorun yang dibuat sebelumnya.
- **[Wine](https://www.winehq.org/)** dan para pengembangnya: dasar dari `user32.dll`, `xaudio2_3.dll`, msacm32, dan codec ACM bawaan. Berlisensi GNU LGPL 2.1.
- **[FAudio](https://github.com/FNA-XNA/FAudio)** oleh Ethan Lee, Luigi Auriemma, dan MonoGame Team: implementasi XAudio2 yang diperbaiki pada `xaudio2_3.dll`. Berlisensi zlib.
- **[DXVK](https://github.com/doitsujin/dxvk)** oleh Philip Rebohle dan kontributornya, serta **[DXVK-Sarek](https://github.com/pythonlover02/DXVK-Sarek)** oleh pythonlover02: lapisan Direct3D 9 ke Vulkan yang dipakai untuk menjalankan game ini.
- **[llvm-mingw](https://github.com/mstorsjo/llvm-mingw)** oleh Martin Storsjö: toolchain untuk membangun DLL.

## Game

F.E.A.R. 2: Project Origin dikembangkan oleh Monolith Productions dan diterbitkan oleh Warner Bros. Interactive Entertainment. Seluruh merek dagang adalah milik pemiliknya masing-masing. Repositori ini tidak berafiliasi dengan mereka dan tidak memuat berkas game apa pun.
