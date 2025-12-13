# Repository Guidelines

## Project Structure & Module Organization

- `package/`: Package recipes (`package/<name>/Makefile`) and patches (`package/<name>/patches/0001-*.patch`).
- `target/`: Target/board definitions, kernel config fragments, and image recipes (e.g., `target/linux/<arch>/`).
- `include/` + `scripts/`: Core build system `.mk` includes and helper scripts (feeds, config utilities, tooling).
- `toolchain/` + `tools/`: Cross toolchain and host tools built as part of the firmware build.
- `config/`, `Config.in`, `feeds.conf.default`: Kconfig and feed configuration inputs.
- Build outputs are generated under `bin/`, `build_dir/`, and `staging_dir/` (do not commit).

## Build, Test, and Development Commands

```sh
./scripts/feeds update -a        # fetch/update feed package definitions
./scripts/feeds install -a       # install feed packages into package/feeds/
make menuconfig                  # select target, toolchain, packages
make defconfig                   # normalize/expand .config defaults
make -j"$(nproc)"                # build firmware/images and selected packages
make package/<pkg>/compile V=s   # build one package (verbose)
make clean | make dirclean       # remove build artifacts (increasingly thorough)
```

## Coding Style & Naming Conventions

- Makefiles: use tabs for recipe lines; prefer existing OpenWrt patterns (`PKG_*`, `define Package/<name>`, `DEPENDS:=+foo`).
- Patches: keep minimal and upstream-friendly; name sequentially (`0001-...patch`) and include clear subjects.
- Shell: prefer POSIX `sh` in `scripts/` and package hooks; avoid bashisms unless the file explicitly requires them.

## Testing Guidelines

- There is no mandatory top-level test suite in this tree; validate changes by rebuilding the affected target/package.
- Use incremental checks: `make package/<pkg>/{clean,compile} V=s` and `make check` (host/toolchain/package sanity checks).

## Commit & Pull Request Guidelines

- Commit subjects follow `area: short imperative summary` (examples in history: `base-files: ...`, `kernel: ...`, `bcm27xx: ...`).
- Include a brief rationale in the body and add trailers when applicable: `Signed-off-by:`, `Fixes: <sha>`, `Closes: #NNNN`, `Link: <PR>`.
- PRs should state target/device (if relevant), how to reproduce/verify, and attach key build logs (e.g., failing command with `V=s`).
