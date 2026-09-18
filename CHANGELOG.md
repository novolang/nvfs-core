# Changelog

All notable changes to nvfs-core are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## 0.0.2 — 2026-09-15

README rewritten to the package README style guide (docs/writing-a-readme.md).

`NvfsError` now declares the `impl Error` its own `Result` position
requires.  `Result<T, E>` has carried the bound `E: Error` since SPEC
§ 3.4, and the compiler enforced it only when `E` was declared in the
module that named it — so `nvfsvol.decode_superblock`, the one call
here that answers a `Result` rather than the bare enum, was accepted
with no impl anywhere.  The impl is the signature this package always
meant; nothing else about the interface changed.

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

### Design notes

The migration of `orbit/novofs`, recorded here because the README no
longer carries it.

**novofs keeps** the driver, which is everything that touches a machine:

| file | why it stays |
| --- | --- |
| `src/flash.nv` | the `Flash` trait, `RamFlash` with its strict NOR semantics and power-cut injection, `FileFlash` over an image file |
| `src/haladapt.nv` | the adapter from `hal.BlockDevice` to `Flash` |
| `src/vfs.nv` | the `FileSystem` trait consumers program against |
| `src/main.nv` | the image-tool CLI — `mk`, `ls`, `cat`, `extract`, `fsck`, `df` |

**novofs takes**, replacing code it has today:

| novofs symbol | this package |
| --- | --- |
| `nvfs.FsConfig` / `nvfs.default_config` / `nvfs.validate` | `nvfsvol.NvfsConfig`, `nvfsvol.default_config`, `nvfsvol.check_config` |
| `nvfs.FsError` / `nvfs.err_str` | `nvfsgeo.NvfsError`, `nvfsgeo.error_name` |
| `nvfs.FsInfo` / `nvfs.MountRes` / `nvfs.parse_sb` / `nvfs.sb_payload` | `nvfsvol.NvfsVolume`, `nvfsvol.NvfsSuperblock`, `nvfsvol.decode_superblock`, `nvfsvol.encode_superblock` |
| `nvfs.DirEntry` / `mk_dir_entry` / `mk_inline_entry` / `mk_ctz_entry` / `enc_dirent` / `dec_dirent` | `nvfsvol.NvfsEntry` and the four constructors and codecs beside it |
| `nvfs.fold_dir` / `entries_find` / `compaction_recs` | `nvfsvol.fold`, `nvfsvol.find`, `nvfsvol.compaction_records` |
| `nvfs.path_comps` / `last_comp` | `nvfsvol.path_components`, `nvfsvol.basename` |
| `nvfs.ctz_k` / `ctz_cap` / `ctz_nblocks` / `ctz_seek` / `ctz_ptr` | `nvfsctz.pointer_count`, `capacity`, `block_count_for`, the `NvfsCtzWalk` walk, `pointer_offset` |
| `nvfs.alloc_blocks`' cursor / `alloc_cursor` / `maybe_relocate`'s threshold | `nvfswear.NvfsAlloc`, `cursor_seed`, `should_relocate` |
| `nvfs.sb_cycles_enc` / `sb_cycles_dec` | `nvfswear.encode_block_cycles`, `decode_block_cycles` |
| `meta.MetaRec` / `MetaScan` / `mk_rec` / `build_record` / `scan` | `nvfsmeta.NvfsRecord`, `NvfsScan`, `record`, `encode_record`, `scan_block` |
| `meta.meta_off` / `align_up` / `pad_to` / `recs_size` | `nvfsgeo.meta_offset`, `nvfsgeo.align_up`, `nvfsmeta.pad_to`, `nvfsmeta.records_span` |
| `meta.append_at` / `compact_to` | `nvfsmeta.append_image`, `compaction_image`, `seal_image` — three images the driver programs, rather than three calls that program |
| `nvfs.pair_state` / `ps_*` | `nvfsmeta.arbitrate`, `NvfsPairState` |
| `src/crc32.nv` (the whole file) | `nvfsmeta.record_crc`, over **crc-nv** |
| `nvfs.format` / `mount` / `mkdir` / `dir_list` / `remove` / `rename` / `file_read` / `file_read_at` / `file_write` / `file_append` / `file_truncate` / `file_size` | one `NvfsOp` each, driven through `nvfsop.begin` / `resume_read` / `resume_ok` |

What novofs **gains**: its `RamFlash` power-cut harness stops being the
only way to test a cut, its `crc32.nv` deletes, and the twelve
filesystem entry points become one loop.  What it **keeps paying**: the
loop is the driver's, so `vfs.NovoFs`'s ten methods each wrap one.

## What widened, and it is named rather than hidden

**File handles.**  `orbit/novofs` is path-based and stateless — every
operation walks from the root — so an append is O(n): it reads the whole
file and writes it back.  Its own README names the fix ("an O(1) tail
append needs file handles").  `NvfsOpen`, `NvfsSeek` and `NvfsClose` are
that fix, and they exist here because the state machine has a place to
keep a cursor that the stateless API had nowhere to put.  The cost is
stated: a handle is a number the machine remembers, so a host that drops
a machine drops its handles, and there is nothing on disk to make stale.

**fsck as an operation.**  novofs's `fsck` is a CLI verb over the `vfs`
trait; here it is `NvfsCheck`, one `NvfsOp` like the others, answering a
report rather than printing one.

## Two deferrals kept, by name

- **Cross-directory rename** is `NvfsNotSupported`.  It needs littlefs's
  global move state — two pairs committed together — which this format
  cannot express.  `nvfsvol.same_parent` is how a caller finds out
  before it starts.
- **Revision wraparound is undefined.**  The revision is a u32 and at
  one compaction a second it wraps in about 136 years.
  `nvfswear.revision_near_wrap` exists so a long-lived logger can detect
  the deferral rather than meet it.
