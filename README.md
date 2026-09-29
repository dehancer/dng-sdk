# Adobe DNG SDK

This repository is a clone of Adobe DNG SDK, managed by Dehancer.

Please make sure the commit history is extra clean so that we can update DNG SDK in the future.

Watch out for DNG SDK releases here: https://helpx.adobe.com/camera-raw/desktop/dng-and-file-formats/digital-negative.html

# Differences

`.gitignore` is exclusive to this repository and is not present in original zip file.

`xmp/toolkit/third-party/zuid/interfaces/MD5.cpp` is only present in this repository.

# Build instructions for macOS

## Deps

You'll *probably* need some or all of these packages preinstalled. At Dehancer this is what we have as deps for DNG SDK and LibRAW.

* lz4
* xz
* zstd
* brotli
* giflib
* jpeg-turbo
* libpng
* webp
* libtiff
* jasper
* little-cms2
* libomp
* openjpeg

## Build DNG SDK

```bash
# Install into this location
export macos_deps_install_dir=/opt/dehancer-dependencies

cd dng-sdk

xcodebuild \
  -project dng_sdk/projects/mac/dng_validate.xcodeproj \
  -configuration Release \
  -scheme dng_validate\ release \
  ARCHS="arm64 x86_64" \
  ONLY_ACTIVE_ARCH=NO \
  MACOSX_DEPLOYMENT_TARGET=15.0 \
  build

# We need to store intermediate artifacts somewhere temporarily outside of the tree, because we are going to
# git clean the repository folder.
local staging_dir=$(mktemp -d)

# Cryptic path where all object files are stored after build
local dng_build_dir="dng_sdk/projects/mac/build/dng_validate.build/Default/dng_validate release.build/Objects-normal"

# create libdng.a with all dng_* files EXCEPT the dng_validate.o
# (which is a sample program with `main()`)
(cd "$dng_build_dir/arm64" && ar rcs $staging_dir/libdng.a.arm64 $(ls -1 *.o | grep -v dng_validate.o))
(cd "$dng_build_dir/x86_64" && ar rcs $staging_dir/libdng.a.x86_64 $(ls -1 *.o | grep -v dng_validate.o))

# Cook fat binary
lipo -create -output $staging_dir/libdng.a $staging_dir/libdng.a.{arm64,x86_64}

# Also copy two of the Adobe XMP SDK static libraries.
# Yes, it's fat binary despite having "intel_64" in the path.
# It's because people are dead inside at Adobe.
p=xmp/toolkit/public/libraries/macintosh/intel_64_libcpp/Release
cp ${p}/libXMPCoreStatic_Release.a $staging_dir/libXMPCore.a
cp ${p}/libXMPFilesStatic_Release.a $staging_dir/libXMPFiles.a

# Make fat binary
p=libjxl/client_projects/mac/build/jxl.build/Default/jxl_release.build/Objects-normal
lipo -create -output "$staging_dir/libjxl.a" ${p}/{arm64,x86_64}/Binary/libjxl_release.a

# Copy libs to the final destination
cp $staging_dir/lib*.a $macos_deps_install_dir/lib/

# Copy includes
mkdir -p $macos_deps_install_dir/include/dng
cp dng_sdk/source/dng_*.h $macos_deps_install_dir/include/dng/
cp -R libjxl/libjxl/lib/include/jxl $macos_deps_install_dir/include/
```

## Build LibRaw

```bash
# Install into this location
export macos_deps_install_dir=/opt/dehancer-dependencies

cd LibRaw

export CFLAGS="$(pkg-config --cflags jasper) \
  -DUSE_DNGSDK=1 \
  -I\"$DNG_SDK_DIR\"/dng_sdk/source \
  -mmacosx-version-min=15"

export CXXFLAGS="$CFLAGS"

export LDFLAGS="-mmacosx-version-min=15 \
  -framework CoreFoundation \
  -framework Carbon \
  -L${macos_deps_install_dir}/lib \
  -lomp \
  $(pkg-config --libs jasper) \
  -lXMPCore \
  -lXMPFiles \
  -ldng \
  -ljxl"

local configure_options="--disable-examples \
  --disable-shared \
  --enable-static \
  --enable-lcms \
  --enable-openmp"

autoreconf --install --force

./configure --prefix=$macos_deps_install_dir $configure_options

# A large chunk is built only when `install`` target is specified
make -j 10 install

# Now let's dance for the Intel
make distclean

export CFLAGS="-arch x86_64 $CFLAGS"
export CXXFLAGS="$CFLAGS"

# There is no need to run configure and make under emulation
# as `-arch x86_64` specified in `CFLAGS` above is already enough to build Intel binaries
./configure --prefix=$macos_deps_install_dir/x86_64 $configure_options

# A large chunk is built only when `install` target is specified
make -j 10 install

mv $macos_deps_install_dir/lib/libraw.a $macos_deps_install_dir/lib/libraw.a.arm64
mv $macos_deps_install_dir/lib/libraw_r.a $macos_deps_install_dir/lib/libraw_r.a.arm64

lipo -create -output $macos_deps_install_dir/lib/libraw.a \
  $macos_deps_install_dir/lib/libraw.a.arm64 \
  $macos_deps_install_dir/x86_64/lib/libraw.a

lipo -create -output $macos_deps_install_dir/lib/libraw_r.a \
  $macos_deps_install_dir/lib/libraw_r.a.arm64 \
  $macos_deps_install_dir/x86_64/lib/libraw_r.a

# Cleanup leftover libs and dirs that may break the build if left alone
rm $macos_deps_install_dir/lib/libraw*.arm64
rm -rf $macos_deps_install_dir/x86_64
```

# Build instructions for Windows

## Deps

You will *probably* need all or some of these vcpkg packages:

* openssl
* curl
* expat
* libiconv
* gtest
* openblas
* lapack
* libzip
* dlfcn-win32
* lcms
* 'tiff[jpeg,lzma,zip]'
* 'libwebp[libwebpmux,nearlossless,simd,unicode]'
* 'libavif[aom,dav1d,svt]'
* openjpeg
* openexr

## Build DNG SDK

We build inside bash from [Git for Windows](https://github.com/git-for-windows/git)
because we like to deal with Windows as less as possible. YMMV and patches are most
welcome.

```bash
# Install into this location
export windows_deps_install_dir=/c/dehancer-dependencies

cd dng-sdk

# dng sdk only has a single solution to build a single binary dng_validate.
# So this is what we do: we build this binary and then manually collect all the
# .obj files into a lib.
"/c/Program Files/Microsoft Visual Studio/2022/Community/MSBuild/Current/Bin/MSBuild.exe" \
  dng_sdk/projects/win/dng_validate.sln \
  //t:Build \
  //p:Configuration="Validate Release" \
  //p:Platform="x64" \
  //m

# Now let's collect the scraps (*.obj) and bake a croissant (dng.lib) out of it.
local dng_lib_file="$PWD/dng.lib"

# Obj files are left over in this folder:
local obj_files_dir="dng_sdk/projects/win/dng_validate/Validate Release/x64"

# Check that we actually did build something
if [[ ! -d "$obj_files_dir" ]]; then
  echo "Object files directory $obj_files_dir does not exist, build failed"
  exit 1
fi

(
  cd "$obj_files_dir"

  # Delete that one file with the `main()` function.
  rm -rf dng_validate*

  # Now create .lib from all .obj files in that folder.
  /c/Program\ Files/Microsoft\ Visual\ Studio/2022/Community/VC/Tools/MSVC/14.*/bin/Hostx64/x64/lib.exe \
    //out:"$(cygpath.exe -w "$dng_lib_file")" \
    *.obj
)

mkdir -p "$windows_deps_install_dir/lib"

# Copy actual dng library
cp $dng_lib_file "$windows_deps_install_dir/lib/"

# We need auxiliary libraries from libjxl and it's deps
local libjxl_output_dir=libjxl/client_projects/win/x64_Release
cp \
  $libjxl_output_dir/brotli.lib \
  $libjxl_output_dir/highway.lib \
  $libjxl_output_dir/jxl.lib \
  "$windows_deps_install_dir/lib/"

# XMP toolkil libraries should be renamed while copying
local xmp_output_dir=xmp/toolkit/public/libraries/windows_x64/Release
cp $xmp_output_dir/XMPCoreStaticRelease.lib "$windows_deps_install_dir/lib/XMPCore.lib"
cp $xmp_output_dir/XMPFilesStaticRelease.lib "$windows_deps_install_dir/lib/XMPFiles.lib"

# While the measly pathetic open source libraries do have separate include directories...
mkdir -p "$windows_deps_install_dir/include"
cp -R libjxl/libjxl/lib/include/jxl "$windows_deps_install_dir/include/"

# ...includes for Adobe DNG SDK are in same folder as source,
# because that's how real enterprise software is created.
mkdir -p "$windows_deps_install_dir/include/dng"
cp dng_sdk/source/dng_*.h "$windows_deps_install_dir/include/dng/"
```

## Build LibRaw

```bash
# Install into this location
export windows_deps_install_dir=/c/dehancer-dependencies

cd LibRaw

git checkout Makefile.msvc # because we modify it later
git clean -xfd

# Enable proper static build
sed 's| /MD | /MT /openmp |' < Makefile.msvc > Makefile.msvc.tmp
mv Makefile.msvc.tmp Makefile.msvc

# Here's where things get ugly. LibRaw does not build from bash. We need to build it from cmd.exe.
cat > build-libraw.bat <<EOF
@rem This bat file is not supposed to be run directly. See caller .sh.
call "c:\Program Files\Microsoft Visual Studio\2022\Community\VC\Auxiliary\Build\vcvarsall.bat" x64
echo Now calling jom
jom -j %1 -f Makefile.msvc CFLAGS_DNG="%~2" LDFLAGS_DNG="%~3" lib/libraw_static.lib
echo jom finished
EOF

#
# Arguments:
# 1. Count of parallel jobs
# 2. `CFLAGS_DNG` as explained in `Makefile.msvc`
# 3. `LDFLAGS_DNG` as explained in `Makefile.msvc`
#
# Note that we have to convert all unix paths into window paths using `cygpath -w`
# and that makes this command line a little bit ugly.
#
./build-libraw.bat \
  $(nproc) \
  "/DUSE_DNGSDK /I$(cygpath -w $windows_deps_install_dir/include/dng) /I$(cygpath -w $windows_deps_install_dir/include)" \
  "$(cygpath -w $windows_deps_install_dir/lib/dng.lib) \
  $(cygpath -w $windows_deps_install_dir/lib/XMPCore.lib) \
  $(cygpath -w $windows_deps_install_dir/lib/XMPFiles.lib) \
  $(cygpath -w $windows_deps_install_dir/lib/jxl.lib) \
  $(cygpath -w $windows_deps_install_dir/lib/highway.lib) \
  $(cygpath -w $windows_deps_install_dir/lib/brotli.lib)"

if [[ ! -f lib/libraw_static.lib ]]; then
  echo "libraw_static.lib was not built successfully"
  exit 1
fi

mkdir -p "$windows_deps_install_dir/lib"
cp lib/libraw_static.lib "$windows_deps_install_dir/lib/libraw.lib"

mkdir -p "$windows_deps_install_dir/include"
cp -R libraw "$windows_deps_install_dir/include/"
```

# Build on Linux

## Deps

```bash
apt install build-essential liblcms2-dev autoconf libtool libjxl-dev libboost-dev clang-21 git zlib1g-dev libjpeg-turbo8-dev libexpat1-dev pkg-config cmake
update-alternatives --install /usr/bin/clang clang /usr/bin/clang-21 100
update-alternatives --install /usr/bin/clang++ clang++ /usr/bin/clang++-21 100
```

Also build jasper:

```bash
# Install into this locaation
export linux_deps_install_dir=/opt/dehancer-dependencies

git clone https://github.com/jasper-software/jasper.git
cd jasper
mkdir b
cd b
cmake -DALLOW_IN_SOURCE_BUILD=1 -DCMAKE_INSTALL_PREFIX=$linux_deps_install_dir ..
cmake --build . --parallel $(nproc)
cmake --install . --parallel $(nproc)
```

And you will need a bunch of file format libraries, the exact list of
which you will have to figure out for yourself and then send us a PR
with the correct list. Because we have no idea. We build using Docker
containers with lots of internal machinery.

## Build DNG SDK

```bash
# Install into this locaation
export linux_deps_install_dir=/opt/dehancer-dependencies

# Make sure Jasper is available.
export PKG_CONFIG_PATH=$linux_deps_install_dir/lib/pkgconfig/

cd dng-sdk

cmake -B build -DDNG_WITH_JPEG=ON -DDNG_WITH_XMP=ON -DDNG_THREAD_SAFE=ON .
cmake --build build --parallel $(nproc)
cmake --install build
```

## Build LibRaw

```bash
# Install into this locaation
export linux_deps_install_dir=/opt/dehancer-dependencies

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
