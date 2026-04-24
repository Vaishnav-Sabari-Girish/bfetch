# bfetch

A lightweight, ultra-optimized system fetch tool written in C.

`bfetch` is designed for absolute maximum speed. It utilizes low-level techniques like inline assembly and memory mapping to achieve performance levels that exceed even purpose-built "fast" tools.

## Features

- **Blazing Performance**: Execution time is typically **~2ms** (up to 55x faster than fastfetch).
- **Accurate Memory**: Directly parses `/proc/meminfo` to calculate available memory correctly, excluding cache.
- **Instant GPU Detection**: Scans `/sys/class/drm` for cards instead of traversing the entire PCI bus, using `mmap` for instant model lookup.
- **Buffered Output**: Builds the entire output in memory and flushes with a single `write()` syscall.
- **Universal Package Counting**:
  - **Nix**: Deep manifest scanning via `mmap` substring search.
  - **Pacman**: Optimized directory entry counting.
  - **Flatpak & Snap**: Native filesystem-based counting.
- **Zero-Copy Architecture**: Minimal memory allocations and no subprocess spawning.
- **Aesthetic**: Custom professional ASCII art for **CachyOS**, **Gentoo**, **Endeavour OS**, and **Bedrock Linux**.

## Usage

```bash
# Build (defaults to fast mode)
make

# Run
./bfetch

# Show Help
./bfetch --Help
```

## Microoptimized C Notes

The main C path is still the core of this repo, but i  wanted another where most of the serious performance work actually happens which is in [bfetchmicro](./bfetchmicro).

This is the version that kept my rules while being pushed as far as possible:

- no caching
- no daemon
- no subprocess cheating
- real package counting
- real GPU detection
- one-buffer render path

If you want the full writeup on what [`fetch.c`](./bfetchmicro/fetch.c) is actually doing, where the time goes, and why the remaining wall is mostly kernel and metadata overhead instead of "C being slow", read [`README.md`](./bfetchmicro/README.md).

## Austral Port

There is also an Austral rewrite in [`bfetchaust/`](./bfetchaust).

I did not want a shitty "ported" version that shells out to half the system and calls it done. The goal was to make the Austral build behave like the C build, keep the same output, keep the same one-buffer render style, and rebuild the missing OS bits that Austral does not hand you out of the box which taught me alot.

That turned into a lot more work than the C version:

- Austral does not come with the filesystem, directory walking, environment access, `/proc`, `/sys`, or package manager probing that this project demands.
- The fetch logic had to stay in Austral, so the missing runtime pieces had to be built under `bfetchaust/runtime/` instead of cheating with shell commands.
- Matching `fetch.c` exactly meant diffing live output against `./bfetch` and copying even the annoying little things, like GPU naming heuristics.
- Austral's ownership and region rules are awesome and strick. Simple C cleanup patterns, early returns, and "just borrow this buffer here" code often had to be rewritten into stricter forms to make the compiler accept them.

The result is good enough to be worth reading if you want to see how far Austral can be pushed on a low-level Linux CLI tool. Output parity is there on the tested paths. Speed parity is not quite there yet. The optimized Austral build is still slower than the C binary, but it is a lot closer once the generated C is compiled with real optimization flags.

If you want the full breakdown , build steps, project layout, and notes on what had to change compared to the C version,and basically my glorified blog post, read [`bfetchaust/README.md`](./bfetchaust/README.md).
do note that i included [`bfetchaust/bfetchaust.c`](./bfetchaust/bfetchaust.c) for study alone and should never be used other than for study and reading

## Assembly Rewrite

There is also a static assembly rewrite in [`Bfetchasm/`](./Bfetchasm).

This one exists because after enough C-side microoptimization, the obvious next question becomes whether the remaining gap is still in userspace glue or whether the kernel and metadata walks are already the real lower bound.

That version is:

- x86_64 assembly
- syscall-only
- statically linked
- machine-targeted
- still doing live checks instead of hardcoded bullshit

The useful part is not just that it runs. The useful part is that it gives a cleaner answer about where the time is actually going. If you want the full build notes, design intent, and what the assembly rewrite did and did not prove, read [`README.md`](./bfetchasm/README.md).

## Comparisons

| Tool       | Approx. Time | Approach                          |
| ---------- | ------------ | --------------------------------- |
| **bfetch** | **~0.002s**  | **ASM + DRM Lookup + Direct I/O** |
| fastfetch  | ~0.10s       | Optimized C / libpci              |
| neofetch   | ~0.80s       | Shell script                      |

## Technical Implementation

- **CPU**: Uses `cpuid` inline assembly to fetch the processor brand string.
- **GPU**: Direct `/sys/class/drm/card*` lookup to identifying vendor/device IDs, then memory-maps `pci.ids` for model name resolution.
- **Memory**: Parses `/proc/meminfo` to calculate `Used = Total - Available` without `sysinfo()` syscall overhead.
- **I/O**: Combined `/etc/os-release` read for both distro name and system type detection.
- **Packages**: optimized recursive directory counting and memory-mapped manifest scanning.

## Installation

### From Source

```bash
git clone https://github.com/Mjoyufull/bfetch.git
cd bfetch
make
./bfetch
```

### Nix

```bash
nix run github:Mjoyufull/bfetch
```

## License

AGPL License
