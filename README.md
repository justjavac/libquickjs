# libquickjs

Static QuickJS build outputs for MoonBit.

## What Gets Built

The archive produced here is the embedder-facing QuickJS core:

- `quickjs.c`
- `cutils.c`
- `dtoa.c`
- `libregexp.c`
- `libunicode.c`

It intentionally does **not** include `quickjs-libc.c`.

That keeps the Windows build compatible with the MSVC toolchain used by MoonBit
native builds and avoids dragging in the POSIX/pthread-based shell helpers from
the upstream CLI.

On Windows, this project also disables QuickJS atomics support inside
`quickjs.c`, because the upstream implementation depends on `pthread`.

## Local Build

Windows:

```powershell
cmake -S . -B build -G Ninja -D CMAKE_BUILD_TYPE=Release
cmake --build build --config Release
cmake --install build --config Release --prefix stage
```

macOS:

```bash
cmake -S . -B build -G Ninja -D CMAKE_BUILD_TYPE=Release \
  -D CMAKE_OSX_ARCHITECTURES="x86_64;arm64" \
  -D CMAKE_OSX_DEPLOYMENT_TARGET=11.0
cmake --build build --config Release
cmake --install build --config Release --prefix stage
```

Linux:

```bash
cmake -S . -B build -G Ninja -D CMAKE_BUILD_TYPE=Release
cmake --build build --config Release
cmake --install build --config Release --prefix stage
```

Installed files are written to `stage/`:

- `include/quickjs/quickjs.h`
- `include/quickjs/LICENSE`
- `include/quickjs/VERSION`
- `lib/quickjs.lib` on Windows
- `lib/libquickjs.a` on Linux and macOS
- `lib/cmake/quickjs/...`

## MoonBit Usage Notes

For MoonBit native FFI usage, point your C include path at `stage/include/quickjs`
and link the static archive from `stage/lib`.

Typical link notes:

- Linux: add `-lm`
- macOS: no extra system libraries are usually needed beyond the default runtime
- Windows: no extra system libraries are needed for the core archive

The public header shipped by this builder is `quickjs.h`.

If you need the upstream shell helpers from `quickjs-libc.c` (`std`/`os` module
bootstrap helpers, worker helpers, etc.), they are intentionally out of scope
for this MoonBit-oriented package.

## GitHub Actions

The workflow lives in `.github/workflows/ci.yml`.

It runs on:

- every push
- pull requests
- manual `workflow_dispatch`

Uploaded artifacts:

- `libquickjs-linux-x64`
- `libquickjs-windows-x64`
- `libquickjs-macos-universal`

Each artifact contains the installed headers, the static library, the CMake
package files, and a `BUILD_INFO.txt` file with the pinned upstream revision and
Windows compatibility notes.

