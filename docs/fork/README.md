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
| `lines` | 2²¹ | 2¹⁷ | 3 MB |
| `seg_counts` | 2²¹ | 2¹⁸ | 2 MB |
| `segments` | 2²¹ | 2¹⁸ | 6 MB |
| `blend_spill` | 2²⁰ | 2²⁰ | 4 MB |
| `ptcl` | 2²³ | 2²¹ | 8 MB |

About 26 MB a set instead of 165.

**An overflow is silent.** The shaders set `BumpAllocators::failed`, coarse and fine
rasterisation exit early, and `render_to_texture` still returns `Ok`: the run comes out blank or
partial. Two ways to check a scene's headroom:

- `VELLO_SCRATCH=upstream` in the environment restores upstream's sizes without a rebuild. If a
  blank run fills in, the scene overflowed.
- vello's `bump_estimate` cargo feature adds `Scene::bump_estimate`, a CPU-side estimate of
  what a scene needs, as a `BumpAllocatorMemory`. amane logs it per vello run under
  `AMANE_FRAMES=1` when built with its `vello-stats` feature.

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
