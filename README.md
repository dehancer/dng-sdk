# Adobe DNG SDK

This repository is a clone of Adobe DNG SDK, managed by Dehancer.

Please make sure the commit history is extra clean so that we can update DNG SDK in the future.

# Differences

`.gitignore` is exclusive to this repository and is not present in original zip file.

# Build instructions for Windows and macOS

See build scripts in `dehancer-ci` repo in `windows/` and `macos/` folders.

# CMake

CMake build is only used for Linux.

Prereqs: recent CMake.

Install packages:

```bash
apt install build-essential liblcms2-dev autoconf libtool libjxl-dev libboost-dev clang-21 git zlib1g-dev libjpeg-turbo8-dev libexpat1-dev pkg-config
update-alternatives --install /usr/bin/clang clang /usr/bin/clang-21 100
update-alternatives --install /usr/bin/clang++ clang++ /usr/bin/clang++-21 100
```

Prepare env:

```bash
export linux_deps_install_dir=/opt/dehancer-dependencies
```

`jasper` build is required:

```
git clone https://github.com/jasper-software/jasper.git
cd jasper
mkdir b
cd b
cmake -DALLOW_IN_SOURCE_BUILD=1 -DCMAKE_INSTALL_PREFIX=$linux_deps_install_dir ..
cmake --build . --parallel $(nproc)
cmake --install . --parallel $(nproc)
```

Then build dng sdk:

```bash
cd dng-sdk
mkdir build
cd build
export PKG_CONFIG_PATH=$linux_deps_install_dir/lib/pkgconfig/
cmake -DDNG_WITH_JPEG=ON -DDNG_WITH_XMP=ON -DDNG_THREAD_SAFE=ON ..
cmake --build . --parallel $(nproc)
```

And finally LibRaw:

```bash
export DNG_SDK_DIR=$PWD/dng-sdk/

export CFLAGS="$(pkg-config --cflags jasper) \
  -DUSE_DNGSDK=1 \
  -I\"$DNG_SDK_DIR\"/dng_sdk/source \
  -DqLinux"

export CXXFLAGS="$CFLAGS"

export LDFLAGS="$(pkg-config --libs jasper) \
  -L${DNG_SDK_DIR}/build \
  -lXMPCore \
  -ldng \
  -ljxl"

configure_options="--disable-examples \
  --disable-shared \
  --enable-static \
  --enable-lcms \
  --enable-openmp"

cd LibRaw

git clean -xfd

autoreconf --install --force

./configure --prefix=$linux_deps_install_dir $configure_options
make -j$(nproc)
make install
./libtool  --finish $linux_deps_install_dir/lib/
```
