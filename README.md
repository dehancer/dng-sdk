# Adobe DNG SDK

This repository is a clone of Adobe DNG SDK, managed by Dehancer. We are patching `dng_validate` source code for Dehancer Desktop, then building manually and shipping a signed binary.

Please make sure the commit history is extra clean so that we can update DNG SDK in the future by re-applying our patches to `dng_validate` sources.

# Differences

`.gitignore` is exclusive to this repository and is not present in original zip file.

`dng_validate.cpp` is where we develop changes.

# Build instructions

## macOS

```bash
common_options="-project dng_sdk/projects/mac/dng_validate.xcodeproj -scheme dng_validate\ release -configuration Release"

destination_dir=dng_sdk/targets/mac/release64
rm -rf $destination_dir # just in case

xcodebuild $common_options ARCHS="arm64" build
cp $destination_dir/dng_validate dns_validate_arm64

xcodebuild $common_options ARCHS="x86_64" build
cp $destination_dir/dng_validate dns_validate_x86_64

lipo dng_validate_{arm64,x86_64} -create -output dng_render
```

No need to sign the binary as CI does it with packaging.

Commit `dng_render` into `dehancer-ci` repository. FIXME: specify path.

## Windows

```bash
destination_dir=dng_sdk/targets/win/release64_x64
rm -rf $destination_dir # just in case

"/c/Program Files/Microsoft Visual Studio/2022/Community/MSBuild/Current/Bin/MSBuild.exe" \
  dng_sdk/projects/win/dng_validate.sln \
  //t:Build \
  //p:Configuration="Validate Release" \
  //p:Platform="x64" \
  //m
cp $destination_dir/dng_validate.exe dng_render.exe
```

Commit `dng_render.exe` into `dehancer-ci` repository. FIXME: specify path.
