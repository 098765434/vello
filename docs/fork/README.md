# vello fork docs

This fork of [vello](https://github.com/linebender/vello) exists for one change, used by the
[amane](https://github.com/098765434/amane) fork and the
[amane-shell](https://github.com/098765434/amane-shell) desktop. It is branch `ocbk` on top of
tag `v0.11.0`, which is what crates.io's `vello 0.11.0` and `vello_encoding 0.11.0` were published
from (the trees are identical). Upstream's own docs and code are left as they are; everything
the fork adds is here.

## Shell-sized scratch buffers

`research/vello_encoding/src/config.rs` sizes the bump-allocated scratch buffers that the
compute pipeline fills on the GPU: `tiles`, `lines`, `seg_counts`, `segments`, `ptcl`,
`bin_data` and `blend_spill`. Upstream hand-picked them "to accommodate the vello test scenes as
well as paris-30k", about 165 MB a set, and `vello::Renderer`'s pool keeps every buffer it ever
made for the renderer's lifetime. A desktop shell drawing a few hundred paths a frame paid some
360 MB of GPU memory for that, measured on an integrated Radeon.

The fork sizes them for a shell's scenes:

| Buffer | Upstream | Fork | Bytes (fork) |
|---|---|---|---|
| `bin_data` | 2¹⁸ | 2¹⁸ | 1 MB |
| `tiles` | 2²¹ | 2¹⁸ | 2 MB |
| `lines` | 2²¹ | 2¹⁸ | 6 MB |
| `seg_counts` | 2²¹ | 2¹⁸ | 2 MB |
| `segments` | 2²¹ | 2¹⁸ | 6 MB |
| `blend_spill` | 2²⁰ | 2²⁰ | 4 MB |
| `ptcl` | 2²³ | 2²¹ | 8 MB |

About 29 MB a set instead of 165. Measured on amane's examples (2026-10-26, read back from the
GPU), the heaviest scene is `canvas`, a stroked clock face and spirograph at 1216×320: 36k lines,
37k segments, 940 tiles. `blur_shapes` at full output (2881×1769) takes 11.7k tiles and 3.1k
segments. The amane-shell desktop itself (widgets, launcher, Quick Settings, bar, dock; 231 runs)
peaks at 38.7k tiles, 104k ptcl entries, 3.5k segments and 1.1k lines. Each limit is 3–7× above
the largest value seen. amane's `gradient` example went from 193 MB to 65 MB of VRAM.

**An overflow is silent.** The shaders set `BumpAllocators::failed`, coarse and fine
rasterisation exit early, and `render_to_texture` still returns `Ok`: the run comes out blank or
partial. Two ways to check a scene's headroom:

- `VELLO_SCRATCH=upstream` in the environment restores upstream's sizes without a rebuild. If a
  blank run fills in, the scene overflowed.
- vello's `debug_layers` cargo feature makes `render_to_texture_async` read the `BumpAllocators`
  counters back from the GPU: `failed`, plus what the run took of each buffer. amane's
  `vello-stats` feature draws that way and prints a `scratch …` line per vello run under
  `AMANE_FRAMES=1`. (vello's `bump_estimate` feature gives a CPU-side bound instead, but it
  overshoots stroked scenes by 10×.)

## Using it

Both `amane` and `amane-shell` patch the crate from their root manifests (a `[patch.crates-io]`
only applies from the root workspace, and amane is a git dependency of the shell):

```toml
[patch.crates-io]
vello_encoding = { git = "https://github.com/098765434/vello", branch = "ocbk" }
```

The nix builds need the fork's content hash in `cargoLock.outputHashes`
(`vello_encoding-0.11.0`); the first build prints the expected `sha256-…` on mismatch.

## Maintaining

- `upstream` remote: `https://github.com/linebender/vello.git`. The fork follows the tag amane's
  `vello` dependency is pinned to; when amane moves to a new vello, rebase `ocbk` onto that tag.
- Upstream has moved its effort to the sparse-strips renderers (`vello_cpu`, `vello_gpu`), where
  this problem does not exist. Moving amane to `vello_gpu` once it has feature parity and a
  stable API retires this fork.
