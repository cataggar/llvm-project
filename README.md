# llvm-zig — LLVM prebuilt for Zig

This is an **orphan branch** of `cataggar/llvm-project`. It carries no LLVM
source — only the CI that produces Zig-oriented prebuilt LLVM toolchains
(`llvm-zig-<version>-<target>`). The workflow checks out `llvm/llvm-project`
at the requested `llvmorg-*` upstream tag and builds it. Releases are tagged
`llvm-zig-<version>`.

See cataggar/llvm-project#4.

## What it produces

Per-target archives published as a GitHub Release named after the release tag
(e.g. `llvm-zig-22.1.2`, whose LLVM source comes from `llvmorg-22.1.2`):

| Target                | Runner            | Archive  |
| --------------------- | ----------------- | -------- |
| `x86_64-linux`        | ubuntu-24.04      | `tar.xz` |
| `aarch64-linux`       | ubuntu-24.04-arm  | `tar.xz` |
| `x86_64-windows-msvc` | windows-2025      | `zip`    |
| `aarch64-macos`       | macos-15          | `tar.xz` |
| `x86_64-macos`        | macos-14 (cross)  | `tar.xz` |

Asset name: `llvm-zig-<version>-<target>.{tar.xz,zip}`. No `.sha256` sidecars —
GitHub publishes a built-in SHA-256 digest on every asset.

## Build configuration

- Projects: `clang;lld` only, `CMAKE_BUILD_TYPE=Release`.
- Unix: compiled with **Zig as the C/C++ compiler**
  (`cataggar/zig@v0.17.0-dev.704+b8cb78023`), installed in CI via
  [`ghr`](https://github.com/cataggar/ghr) and minisign-verified. `zig cc`
  cross-compiles the `*-musl` Linux and `x86_64-macos` targets.
- Windows: MSVC with static CRT (`/MT`).
- No external deps: zlib, zstd, libxml2, terminfo all OFF (empty stub
  `libz`/`libzstd` are created so downstream `find_package` is satisfied).
- Tools / utils / tests / docs / benchmarks / examples OFF; `bin/`, shared
  libs, `libexec/`, `share/` pruned from the install tree.

## Running it

Trigger from the GitHub Actions UI (`workflow_dispatch`) on the `zig` branch:

- **tag**: `llvm-zig-22.1.2` — the release tag. The LLVM source is checked out
  from the matching `llvmorg-<version>` upstream tag.
- **target**: leave empty for all, or one target to rebuild just that one

Or push a `llvm-zig-*` (or `llvmorg-*`) tag whose commit carries this workflow.

## Consuming it (Zig)

```sh
zig build -Dstatic-llvm --search-prefix <extracted llvm-zig dir>
```

Downstream `cataggar/zig` (`libc17-704`, which pins `find_package(llvm 22)`)
fetches `llvm-zig-22.1.2-<target>` from this repo's releases.
