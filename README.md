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
apt install build-essential liblcms2-dev autoconf libtool libjxl-dev libboost-dev
```

`jasper` build is required:

```
git clone https://github.com/jasper-software/jasper.git
cd jasper
mkdir b
cd b
cmake -DALLOW_IN_SOURCE_BUILD=1 -DCMAKE_INSTALL_PREFIX=/opt/dehancer-dependencies ..
cmake --build . --parallel $(nproc)
cmake --install . --parallel $(nproc)
```
