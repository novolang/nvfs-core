# Changelog

All notable changes to nvfs-core are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## 0.0.1 — 2026-09-11

The **interface**: every signature and every effect row, and no bodies.
`stability = "draft"`, and the release is recorded `implemented = false`.

### Added

- `nvfsgeo` — the block and record layout as integer arithmetic, and
  the error set as an enum with no payloads so it crosses to a device.
  In the device claim.
- `nvfsctz` — the CTZ skip-list as arithmetic, with `NvfsCtzWalk` as
  three `Int`s in a `@value` struct so a firmware seeks with no heap.
  In the device claim.
- `nvfswear` — arbitration, the rotating allocation cursor and the three
  power-loss rules, each as a predicate. In the device claim.
- `nvfsmeta` — the record log: `scan_block` takes the block's BYTES, and
  the commit cycle is three images the driver programs rather than three
  calls that program.
- `nvfsvol` — the superblock, the four record types, the directory
  entries and the fold that turns a log into a directory.
- `nvfsop` — `NvfsAction`, and every operation from mount to fsck as a
  machine that answers one.

### Known

- **`NvfsAction` is the load-bearing interface.** littlefs's four
  `lfs_config` callbacks, turned into values. The whole format becomes
  exercisable against a byte array, a power cut becomes a test that
  stops feeding answers, and one machine serves a QEMU flash, an image
  file and a real NOR part.
- **The device claim is BUILT** and covers `nvfsgeo`, `nvfsctz` and
  `nvfswear`; `tests/embedded_probe.nv` links for `--target=nrf52-qemu`.
  The other three modules allocate by construction and are not claimed.
- **A rename is ONE record**, which is why a power cut cannot lose the
  file, and why a cross-directory move is `NvfsNotSupported` rather than
  done unsafely.
- **A compaction seals the revision word LAST**, on its own prog page,
  and `nvfsgeo.meta_offset` is the padding that makes it possible.
- **Free means "not reached from the root"**, never a free list, so a
  crash between an allocation and its commit costs nothing.
- **File handles are a WIDENING** of what `orbit/novofs` has. It is
  path-based and stateless, so its append is O(n); the state machine has
  a place to keep a cursor that the stateless API had nowhere to put.
- **The CTZ capacity is uniform**, which littlefs's is not. Inherited
  from `orbit/novofs`: it wastes pointer slots and makes
  `offset -> index` one division.
- **Revision wraparound is undefined**, and `revision_near_wrap` is how
  a caller detects the deferral rather than meeting it.
- **One dependency**, crc-nv, by registry range, named only by
  `nvfsmeta`. heapless-nv is deliberately absent: the buffers are the
  caller's.
- **Every struct carries a constructor** because a struct declared in
  one module cannot be built with a qualified literal from another:
  `nvfsvol.NvfsGeometry { … }` in a caller is a parse error.
