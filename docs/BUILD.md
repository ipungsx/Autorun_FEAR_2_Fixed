# Membangun ulang DLL

DLL di `sdcard/` dapat dibangun ulang dari kode sumber Autorun ditambah patch di folder `patches/`. Yang dibangun hanya DLL PE 32-bit (i386). Runtime Autorun untuk Switch tidak perlu dibangun.

## Kebutuhan

- Linux x86_64. Contoh di bawah diuji pada Ubuntu 24.04.
- [llvm-mingw](https://github.com/mstorsjo/llvm-mingw) 20260505 (UCRT, ubuntu-22.04-x86_64).
- Paket untuk configure Wine: `gcc`, `make`, `flex`, `bison`, `perl`.

## Langkah

```sh
git clone https://github.com/autorunhq/autorun
cd autorun
git checkout 1ac9b6cf37b90f2d5783dd7ef389bebbe1c68877

git apply /path/ke/patches/0001-faudio-xaudio2-refused-stream-stereo-fold.patch
git apply /path/ke/patches/0002-user32-mouse-motion-flush-and-fear2-mouselook.patch

export PATH=/path/ke/llvm-mingw-20260505-ucrt-ubuntu-22.04-x86_64/bin:$PATH

mkdir build-pe-i386 && cd build-pe-i386
../configure --enable-archs=i386 --disable-tests --without-x --without-freetype \
  --without-alsa --without-pulse --without-dbus --without-fontconfig --without-udev \
  --without-usb --without-gstreamer --without-vulkan --without-wayland --without-gnutls \
  --without-sdl --without-capi --without-cups --without-krb5 --without-netapi \
  --without-opencl --without-pcap --without-pcsclite --without-v4l2 --without-xinerama \
  --without-osmesa --without-unwind --without-ffmpeg
cd ..

make -C build-pe-i386 -j"$(nproc)" dlls/xaudio2_3/i386-windows/xaudio2_3.dll
make -C build-pe-i386 -j"$(nproc)" dlls/user32/i386-windows/user32.dll

llvm-strip --strip-debug -o xaudio2_3.dll build-pe-i386/dlls/xaudio2_3/i386-windows/xaudio2_3.dll
llvm-strip --strip-debug -o user32.dll   build-pe-i386/dlls/user32/i386-windows/user32.dll
```

Hasil build tidak dijamin identik byte demi byte dengan berkas di repositori ini, karena jalur folder dan waktu build ikut tertanam. Perilakunya akan sama.

## Patch diagnostik

`patches/diagnostics/` berisi patch yang hanya menambah catatan ke log dan dipakai selama analisis:

- `msvcrt-beginthreadex-trace.patch`: mencatat modul, alamat awal, dan nilai kembali setiap thread yang dibuat lewat `_beginthreadex`. Bangun `dlls/msvcr80/i386-windows/msvcr80.dll`.
- `user32-cursor-warp-trace.patch`: mencatat posisi kursor setiap kali program memanggil `SetCursorPos` dan `ClipCursor`. Patch ini tidak dapat dipakai bersamaan dengan patch `0002`.

Patch diagnostik tidak diperlukan untuk bermain.
