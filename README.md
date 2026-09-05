# cataggar/llvm-project

A personal mirror of [LLVM](https://llvm.org) plus the CI that builds
Zig-oriented prebuilt LLVM toolchains.

## Branches

- **`main`** — a mirror of the upstream source from
  [github.com/llvm/llvm-project](https://github.com/llvm/llvm-project).
  Synced occasionally; carries no local changes.
- **`mirror`** (default) — release and CI tooling only, no LLVM source. This
  is an **orphan branch** holding the GitHub Actions workflows and Copilot
  instructions for this fork.

## CI / tooling on this branch

- **`.github/workflows/build-zig-llvm.yml`** — builds the Zig-oriented prebuilt
  LLVM toolchains (`llvm-zig-<version>-<target>`). The workflow checks out
  `llvm/llvm-project` at the requested `llvmorg-*` upstream tag and builds it.
  Releases are tagged `llvm-zig-<version>`. See cataggar/llvm-project#4.
- **`.github/copilot-instructions.md`** — repository instructions for GitHub
  Copilot.

## What the build produces

Per-target archives published as a GitHub Release named after the release tag
(e.g. `llvm-zig-22.1.2`, whose LLVM source comes from `llvmorg-22.1.2`):

| Target                | Runner            | Archive  |
| --------------------- | ----------------- | -------- |
| `x86_64-linux`        | ubuntu-24.04      | `tar.xz` |
| `aarch64-linux`       | ubuntu-24.04-arm  | `tar.xz` |
| `x86_64-windows-msvc` | windows-2025      | `zip`    |
| `aarch64-macos`       | macos-15          | `tar.xz` |

Asset name: `llvm-zig-<version>-<target>.{tar.xz,zip}`. No `.sha256` sidecars —
GitHub publishes a built-in SHA-256 digest on every asset.

The `x86_64-linux` build also publishes
`llvm-tools-<version>-x86_64-linux.tar.xz`, a tools-only archive containing
`llvm-nm`, `llvm-objcopy`, `llvm-objdump`, `llvm-readelf`, and `llvm-strip`.
It is staged from the same LLVM install tree, without a second compilation.

## Build configuration

- Projects: `clang;lld` only, `CMAKE_BUILD_TYPE=Release`.
- Unix: compiled with **Zig as the C/C++ compiler**
  (`cataggar/zig@v0.17.0-dev.704+b8cb78023`), installed in CI via
  [`ghr`](https://github.com/cataggar/ghr) and minisign-verified. `zig cc`
  cross-compiles the `*-musl` Linux targets.
- Windows: MSVC with static CRT (`/MT`).
- No external deps: zlib, zstd, libxml2, terminfo all OFF (empty stub
  `libz`/`libzstd` are created so downstream `find_package` is satisfied).
- Tests / docs / benchmarks / examples OFF. Tool binaries **ON**
  (`LLVM_BUILD_TOOLS`/`LLVM_BUILD_UTILS`/`CLANG_BUILD_TOOLS`/`LLD_BUILD_TOOLS`),
  then `bin/` is slimmed to just the toolchain binaries Bun needs (clang,
  clang++, clang-cl, llvm-ar, llvm-ranlib, llvm-lib, ld.lld, lld-link,
  llvm-strip, llvm-rc, dsymutil, …) so the archive stays under GitHub's 2 GB
  release-asset limit. Shared libs, `libexec/`, `share/` are pruned; the static
  libs link directly into the tools (`LLVM_BUILD_LLVM_DYLIB=OFF`).

## Running the build

Trigger from the GitHub Actions UI (`workflow_dispatch`) on the `mirror` branch:

- **tag**: `llvm-zig-22.1.2` — the release tag. The LLVM source is checked out
  from the matching `llvmorg-<version>` upstream tag.
- **target**: leave empty for all, or one target to rebuild just that one

Or push a `llvm-zig-*` (or `llvmorg-*`) tag whose commit carries this workflow.

## Consuming it

**Zig (static libs):**

```sh
zig build -Dstatic-llvm --search-prefix <extracted llvm-zig dir>
```

Downstream `cataggar/zig` (`libc17-704`, which pins `find_package(llvm 22)`)
fetches `llvm-zig-22.1.2-<target>` from this repo's releases.

**Bun (`buz`) toolchain:** point the toolchain search at
`<extracted llvm-zig dir>/bin`. Bun invokes the Clang/LLD tool binaries
directly (it does not link any LLVM library).

**LLVM command-line tools:** extract the matching
`llvm-tools-<version>-x86_64-linux.tar.xz` asset and invoke the tools from its
`bin/` directory. The tools archive does not contain the static development
libraries, headers, CMake metadata, or Clang/LLD tools from the full package.
