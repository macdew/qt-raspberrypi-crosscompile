# Qt for raspberry pi 3

Notes on cross-compiling Qt5 for raspberry pi using linux.  The resulting Qt intallation should have hardware opengl, touch screen, and multimedia support.

This approach pulls the cross-compiling sysroot from the raspberry pi, so step 1 is setting up the pi so that we start with a valid sysroot.

## Setting up the raspberry pi 3

Download the official [Raspbian (Raspberry Pi OS)](https://www.raspberrypi.org/downloads/raspberry-pi-os/) image. For convenience, use the "Desktop" version as it has X-windows support.  Burn the image to SD card.

Edit /boot/config.txt for my touch screen and to enable the vc4 graphics driver:

```Text
max_usb_current=1
hdmi_force_hotplug=1
hdmi_group=2
hdmi_mode=87
hdmi_drive=1
display_rotate=0
hdmi_cvt 1024 600 60 6 0 0 0

dtoverlay=vc4-fkms-v3d
```

Create a file */boot/ssh* to enable ssh into the pi

Boot the pi and go through basic setup, creating a user account, setting up a default password etc.

```Text
sudo nano /etc/apt/sources.list
-> uncomment deb-src line

sudo apt update
sudo apt upgrade

sudo reboot

sudo raspi-config
-> update keyboard, update wifi region, update
```

Install the libraries that we need to run Qt:

```Text
sudo apt-get install -y sense-hat libatspi-dev build-essential libfontconfig1-dev libdbus-1-dev libfreetype6-dev libicu-dev libinput-dev libxkbcommon-dev libsqlite3-dev libssl-dev libpng-dev libjpeg-dev libglib2.0-dev libraspberrypi-dev

sudo apt-get install -y libegl1-mesa-dev libgbm-dev libgles2-mesa-dev mesa-common-dev libgl1-mesa-dev libgraphite2-dev

sudo apt-get install -y libxcb-xinput-dev bison flex

sudo apt-get install -y libharfbuzz-dev libnspr4-dev libvulkan-dev libxml2-dev libxnvctrl-dev libxnvctrl0 libxslt1-dev libx11-xcb-dev

sudo apt-get install -y libudev-dev libts-dev libxcb-xinerama0-dev libxcb-xinerama0

sudo apt-get install libnss3 libnss3-dev

# multimedia libraries - optional?
sudo apt-get install libgstreamer1.0-0 gstreamer1.0-plugins-base gstreamer1.0-plugins-good gstreamer1.0-plugins-bad gstreamer1.0-plugins-ugly gstreamer1.0-libav gstreamer1.0-doc gstreamer1.0-tools gstreamer1.0-x gstreamer1.0-alsa gstreamer1.0-gl gstreamer1.0-pulseaudio gstreamer1.0-omx gstreamer1.0-omx-rpi libgstreamer-plugins-base1.0-dev

sudo apt-get install pulseaudio
sudo nano /etc/pulse/client.conf -> autospawn = yes
```

Install the remaining Qt libraries (in a way that we can easily uninstall them again)

```Text
sudo apt install devscripts
mk-build-deps libqt5gui5
sudo apt install ./qtbase-opensource-src-build-deps_5.11.3+dfsg1-1+rpi1+deb10u3_armhf.deb
```

## Setting up the linux host

Note: I used WSL, and in particular the superb [WSL Arch distro](https://github.com/yuk7/ArchWSL)

generate a ssh key and copy it to the raspberry pi

```Text
ssh-keygen -t rsa -b 4096
ssh-copy-id -i ~/.ssh/id_rsa.pub pi@raspberrypi
```

Next, sync the necessary files from the r-pi to linux

```Text
mkdir raspi
cd raspi
mkdir sysroot sysroot/usr sysroot/usr/share sysroot/opt sysroot/lib
rsync -avzz pi@raspberrypi:/lib sysroot
rsync -avzz pi@raspberrypi:/usr/include sysroot/usr
rsync -avzz pi@raspberrypi:/usr/lib sysroot/usr
rsync -avzz pi@raspberrypi:/usr/share/pkgconfig sysroot/usr/share

wget https://raw.githubusercontent.com/Kukkimonsuta/rpi-buildqt/master/scripts/utils/sysroot-relativelinks.py
chmod +x sysroot-relativelinks.py
./sysroot-relativelinks.py sysroot
```

Once we have copied all the files across, we can clean up the pi:

```Text
sudo apt remove devscripts qtbase-opensource-src-build-deps libnss3-dev
sudo apt autoremove
sudo apt clean
```

## Build the cross-compiling gcc

For cross-compiling, we're going to match the compiler to the one on the r-pi

```Text
on host (WSL):
export WORKSPACE_ROOT=/home/mark/raspi/toolchain
export PREFIX=${WORKSPACE_ROOT}/toolchain
export TARGET=arm-linux-gnueabihf
export SYSROOT=${WORKSPACE_ROOT}/../sysroot
export SOURCES=${WORKSPACE_ROOT}/sources
export PATH=${PREFIX}/bin:$PATH
mkdir -p ${PREFIX}/sources

cd ${WORKSPACE_ROOT}/sources

# note: match versions with raspbian buster....

# eg ld --version: 2.31.1
wget https://ftpmirror.gnu.org/binutils/binutils-2.31.1.tar.bz2

# gcc --version: 8.3.0
wget https://ftpmirror.gnu.org/gcc/gcc-8.3.0/gcc-8.3.0.tar.gz


tar xf binutils-2.31.1.tar.bz2
tar xf gcc-8.3.0.tar.gz
```

Install the necessary build tools on Arch:

```Text
sudo pacman -S make python2 bison diffutils gperf flex pkg-config --needed
```

Start the build

```Text
cd ${WORKSPACE_ROOT}
mkdir -p build/binutils
cd  build/binutils
../../sources/binutils-2.31.1/configure --target=${TARGET} --prefix=${PREFIX} --with-sysroot=${SYSROOT} --with-arch=armv6 --with-fpu=vfp --with-float=hard --disable-multilib
make -j12
make install

cd ${WORKSPACE_ROOT}/sources
cd gcc-8.3.0
contrib/download_prerequisites

cd ${WORKSPACE_ROOT}
mkdir -p build/gcc2
cd build/gcc2
../../sources/gcc-8.3.0/configure --prefix=${PREFIX} --target=${TARGET} --with-sysroot=${SYSROOT} --enable-languages=c,c++ --disable-multilib --enable-multiarch --with-arch=armv6 --with-fpu=vfp --with-float=hard
make -j12
make install
```

## Build Qt

Download Qt sources

```Text
cd ${WORKSPACE_ROOT}/sources

wget https://download.qt.io/archive/qt/5.12/5.12.8/single/qt-everywhere-src-5.12.8.tar.xz

tar xf qt-everywhere-src-5.12.8.tar.xz
```

### Patching Qt

Qt has a few build failures when compiling for raspberry pi.  

```Diff
diff –git a/src/3rdparty/javascriptcore/JavaScriptCore/wtf/Platform.h b/src/3rdparty/javascriptcore/JavaScriptCore/wtf/Platform.h
index a4695a2..897c90c 100644
— a/src/3rdparty/javascriptcore/JavaScriptCore/wtf/Platform.h
+++ b/src/3rdparty/javascriptcore/JavaScriptCore/wtf/Platform.h
@@ -306,6 +306,9 @@
|| defined(ARM_ARCH_7R)
#define WTF_ARM_ARCH_VERSION 7

+#elif defined(__ARM_ARCH_8A__)
+#define WTF_ARM_ARCH_VERSION 8
+
/* RVCT sets _TARGET_ARCH_ARM */
#elif defined(__TARGET_ARCH_ARM)
#define WTF_ARM_ARCH_VERSION __TARGET_ARCH_ARM
```

### WebEngine

When building Qt, we will likely run out of memory when compiling Webengine (Chrome).  Try to mitigate the number of parallel compile jobs with:

```Bash
export MAKEOPTS="-j4"
export NINJAJOBS="-j4"
export USE="-jumbo-build"
```

WebEngine also needs a 32-bit compiler.  Because my WSL was 64-bit, I required the following 32-bit (Arch multilib) repository modifications:

```Text
sudo nano /etc/pacman.conf
# uncomment the [multilib] section
[multilib]
Include = /etc/pacman.d/mirrorlist

sudo pacman -Syy
sudo pacman -S gcc-multilib lib32-gcc-libs lib32-glibc
```

### Building Qt

```Text
cd ${WORKSPACE_ROOT}

mkdir -p build/qt
cd build/qt

../../sources/qt-everywhere-src-5.12.8/configure -release -opengl es2 -device linux-rasp-pi3-vc4-g++ \
-opensource -confirm-license -nomake tests -no-pch -eglfs -xcb -make libs -no-use-gold-linker \
-device-option CROSS_COMPILE=/home/mark/raspi/toolchain/toolchain/bin/arm-linux-gnueabihf- \
-skip qtwayland -skip qtlocation \
-prefix /usr/local/qt/5.13 \
-extprefix /opt/qt/5.13/raspbian/sysroot \
-hostprefix /opt/qt/5.13/raspbian \
-sysroot /home/mark/raspi/sysroot

make
make install
```

### Install Qt back onto raspberry pi

```Text
rsync -avzz /opt/qt/5.14/raspbian/sysroot/* pi@raspberrypi:/usr/local/qt/5.14
```
