# `wine-nvml`

## NVIDIA Management Library wrapper for Wine

`wine-nvml` allows applications running under Wine to call some NVML functions as if native Windows driver was installed into the prefix for purposes like monitoring GPU temperature, utilization, etc. Obviously, for this to work, the host system should have `libnvidia-ml.so` installed (it comes with the NVIDIA driver) and findable by the dynamic linker.

## Building

`wine-nvml` can be built with the Meson build system. You'll need reasonably recent versions of MinGW, GCC, Meson, Ninja and Wine (more specifically: `winebuild`, `winegcc` and `wrc` programs available in your `$PATH`) to do so.

1. Navigate to `src` subdirectory and execute `./make_nvml` to acquire `nvml.h` from `nvidia-settings` repo on GitHub and generate code to compile.
2. Run `meson setup` to generate build trees. Use `cross-{mingw,wine}{64,32}.txt` cross files to setup builds for both PE and Unixlib components using 64–bit and/or 32–bit variant.
3. Run `ninja` in each component's build directory to build it.

Please refer to `build.sh` helper script for automated (but simplified and not very flexible) building procedure.

## Installing

In order for Wine to find and make use of `wine-nvml`, built wrapper libraries must be placed alongside other Wine PEs and Unixlibs.

**Wine / Proton versions older than 7.0 are not supported.**

### Wine ≥ 7.0

Find Wine's library dirs for each given arch and copy built `nvml.{dll,so}` into appropriate subdirs. For example, on Arch Linux using the default `wine` package it would be:

```sh
build-wine64/src/nvml.so   → /usr/lib/wine/x86_64-unix/nvml.so
build-mingw64/src/nvml.dll → /usr/lib/wine/x86_64-windows/nvml.dll
build-wine32/src/nvml.so   → /usr/lib32/wine/i386-unix/nvml.so
build-mingw32/src/nvml.dll → /usr/lib32/wine/i386-windows/nvml.dll
```

If you had any Wine prefixes created before you installed `wine-nvml`, each one would need to be updated with `wineboot -u` for NVML to become available in that prefix.

### Proton ≥ 7.0

Assuming that files of your Proton installation live in `${HOME}/.local/share/Steam/steamapps/common/Proton - Experimental/files` (henceforth referred to as `${files}`), copy resulting build artifacts like so:

```sh
build-wine32/src/nvml.so   → ${files}/lib/wine/i386-unix/nvml.so
build-mingw32/src/nvml.dll → ${files}/lib/wine/i386-windows/nvml.dll
build-wine64/src/nvml.so   → ${files}/lib64/wine/x86_64-unix/nvml.so
build-mingw64/src/nvml.dll → ${files}/lib64/wine/x86_64-windows/nvml.dll
```

It is possible that Proton won't copy `nvml.dll` into game's prefixes on its own even on prefix updates. In that case, you should copy them manually. Assuming that your compatdata lives in `${HOME}/.local/share/Steam/steamapps/compatdata/${appid}` where `${appid}` is your game's Steam AppId (henceforth referred to as `${compatdata}`):

```sh
build-mingw32/src/nvml.dll → ${compatdata}/pfx/drive_c/windows/syswow64/nvml.dll
build-mingw64/src/nvml.dll → ${compatdata}/pfx/drive_c/windows/system32/nvml.dll
```

## Usage

`wine-nvml` is mainly used by [DXVK-NVAPI](https://github.com/jp7677/dxvk-nvapi) to provide GPU readings for games that include their own performance overlays (or other ways to display GPU metrics) and use NVAPI to query such information for Nvidia GPUs. When both are installed in a Wine prefix, `wine-nvml` will automatically be used when a game queries NVAPI for GPU stats.

An example application by NVIDIA and a `Makefile` to build it for Wine can be found in `example/` directory. It can be used to test if the wrapper is working correctly. In order to build it, you need to either install `wine-nvml` into Wine that provides your `winegcc` or add path to `nvml.dll` to `LDFLAGS` like so: `LDFLAGS='-L/path/to/wine-nvml' make`. Alternatively, it can be built using MinGW directly by linking against Nvidia's `nvml.dll` for Windows or `nvml.lib` import library from Nvidia's CUDA SDK. Please note that call to `nvmlDeviceSetComputeMode` is expected to fail because Wine cannot run as `root`.

## Potential issues

* On Windows, `nvml.dll` can reside in `%ProgramW6432%/NVIDIA Corporation/NVSMI`, in `%SystemRoot%/System32`, or in some other directory. Because of this, it's possible that some applications may attempt to use NVML by means other than `LoadLibrary("nvml.dll")` or linking against it with `-lnvml`. Such attempts will most likely fail unless additional workarounds are applied.

* Most functions are implemented on a best–effort basis either by passing the call to native NVML as–is or immediately returning an error if the documentation indicates that it would be the expected behavior when ran on Windows without administrator privileges. Any undocumented differences in behavior between the platforms will likely cause programs to observe Linux behavior.

* Some functions expose PIDs of native Linux processes to Windows programs, potentially confusing them if they attempt to do anything with given numbers other than call `nvmlSystemGetProcessName` on them.

Please report issues if you encounter any programs that use NVML and don't work as they would on Windows, _with the one exception below_.

## Known issues

* Undocumented/internal functions, like the one used by Nvidia's own `nvidia-smi.exe` tool, are entirely unsupported.
  * When encountered, the Wine log will usually contain lines like `fixme:nvml:nvmlInternalGetExportTable (…): stub`.
  * This is unlikely to ever be fixed without proper documentation from Nvidia.

## Licensing

This project is released on the terms of LGPL-2.1-or-later. However, this license _does not apply to files provided by NVIDIA Corporation_. Such files will have a header with their own license at the top similar to the one in the section below.

## NVIDIA License

> ```
> Copyright 1993-2022 NVIDIA Corporation.  All rights reserved.
>
> NOTICE TO USER:
>
> This source code is subject to NVIDIA ownership rights under U.S. and
> international Copyright laws.  Users and possessors of this source code
> are hereby granted a nonexclusive, royalty-free license to use this code
> in individual and commercial software.
>
> NVIDIA MAKES NO REPRESENTATION ABOUT THE SUITABILITY OF THIS SOURCE
> CODE FOR ANY PURPOSE.  IT IS PROVIDED "AS IS" WITHOUT EXPRESS OR
> IMPLIED WARRANTY OF ANY KIND.  NVIDIA DISCLAIMS ALL WARRANTIES WITH
> REGARD TO THIS SOURCE CODE, INCLUDING ALL IMPLIED WARRANTIES OF
> MERCHANTABILITY, NONINFRINGEMENT, AND FITNESS FOR A PARTICULAR PURPOSE.
> IN NO EVENT SHALL NVIDIA BE LIABLE FOR ANY SPECIAL, INDIRECT, INCIDENTAL,
> OR CONSEQUENTIAL DAMAGES, OR ANY DAMAGES WHATSOEVER RESULTING FROM LOSS
> OF USE, DATA OR PROFITS,  WHETHER IN AN ACTION OF CONTRACT, NEGLIGENCE
> OR OTHER TORTIOUS ACTION,  ARISING OUT OF OR IN CONNECTION WITH THE USE
> OR PERFORMANCE OF THIS SOURCE CODE.
>
> U.S. Government End Users.   This source code is a "commercial item" as
> that term is defined at  48 C.F.R. 2.101 (OCT 1995), consisting  of
> "commercial computer  software"  and "commercial computer software
> documentation" as such terms are  used in 48 C.F.R. 12.212 (SEPT 1995)
> and is provided to the U.S. Government only as a commercial end item.
> Consistent with 48 C.F.R.12.212 and 48 C.F.R. 227.7202-1 through
> 227.7202-4 (JUNE 1995), all U.S. Government End Users acquire the
> source code with only those rights set forth herein.
>
> Any use of this source code in individual and commercial software must
> include, in the user documentation and internal comments to the code,
> the above Disclaimer and U.S. Government End Users Notice.
> ```
