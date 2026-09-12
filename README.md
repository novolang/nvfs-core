# nvfs-core

A littlefs-class filesystem with the device left out: the on-disk
format, the commit cycle and every operation as a state machine that
**answers** the block reads and writes a host must perform.

**Status: NOT IMPLEMENTED — interface only.**  Every `pub fn` body is a
`todo()`, so the signatures, the effect rows and the tests are published
and nothing is implemented.  The first implementation is the `0.1.0`
published over this.

## What this is

`orbit/novofs` is a power-loss-resilient filesystem for raw NOR flash —
metadata pairs, CRC'd commit logs, CTZ skip-lists, copy-on-write by
construction — and it is an `app`, because its filesystem calls a
`Flash` trait whose every method is `[io, hw, mutate]`.  That is the
inversion littlefs has too: four function pointers in an `lfs_config`,
called from inside.

This package is the same filesystem with the call turned around.  The
state machine **returns** the action and the host performs it, so the
format is arithmetic over bytes the caller already holds, the package is
`core` with an empty effect budget, and three of its six modules link
for a Cortex-M.

| module | holds | device |
| --- | --- | --- |
| `nvfsgeo` | the block and record layout, and the error set | ✓ |
| `nvfsctz` | the CTZ skip-list, and a walk as three integers | ✓ |
| `nvfswear` | arbitration, allocation, and the power-loss rules | ✓ |
| `nvfsmeta` | the record log, its scan and its commit-and-seal cycle | |
| `nvfsvol` | the superblock, the entries, and the directory fold | |
| `nvfsop` | every operation as a machine, and `NvfsAction` | |

## Adding it, and checking it

```console
$ novo pkg add nvfs-core
$ novo pkg build
$ novo test tests/nvfsop_tests.nv
```

`novo pkg add` says `NOT IMPLEMENTED — interface only` on the way in,
because an interface resolves, downloads and builds exactly like an
implemented package and the difference only shows the first time
something calls it.

## The one example that will work

```novo
use std.bytes
use nvfsop
use nvfsvol

fn main() [io]
    let g = nvfsvol.geometry(16, 16, 512, 32)
    var p = nvfsop.begin(g, nvfsvol.default_config(), NvfsMount)
    var running = true
    while running
        match p.step
            NvfsNeed(a) =>
                match a
                    // A real driver reads the part here; this one has
                    // an erased device, which is what a fresh board has.
                    NvfsRead(_, _, len) => p = nvfsop.resume_read(p.machine, bytes.zeros(len))
                    _                   => p = nvfsop.resume_ok(p.machine, true)
            NvfsDone(_)   =>
                println("mounted")
                running = false
            NvfsFailed(_) =>
                println("no filesystem — format it")
                running = false
```

## The load-bearing interface

`NvfsAction` — littlefs's four `lfs_config` callbacks turned into
values.

```novo norun:pseudo
pub enum NvfsAction
    NvfsRead(block: Int, off: Int, len: Int)
    NvfsProg(block: Int, off: Int, data: Bytes)
    NvfsErase(block: Int)
    NvfsSync
```

The signatures are the callbacks' own; what changed is **who makes the
call**.  Three things follow that could not before.

**1. The whole format is exercisable against a byte array.**  There is
no device to stand up, no trait to implement and no `[io]` anywhere: a
test answers `NvfsRead` out of a `Bytes` it built by hand.
`tests/nvfsop_tests.nv` drives a format and a mount to completion in
forty lines, and the "device" in it is a `while` loop.

**2. A power cut is a test that stops feeding answers.**  `orbit/novofs`
proves its crash resistance with a **1096-cut torture harness** built
around a fault-injecting RAM device that can tear a program mid-page and
fail every operation after it.  Here, a cut at action N is: run N
actions, stop, and mount what the array holds.  Every one of those 1096
cuts is reachable with no device at all, and `actions_taken` is what
makes "cut at N" addressable.  The harness stops being a fixture and
becomes a loop.

**3. One machine serves every device.**  A QEMU flash, a file-backed
image and a real NOR part differ in who performs the action and in
nothing else, so there is no trait parameter and nothing to instantiate
per device.

The cost, stated: the machine holds its own state, so a caller drives a
loop rather than calling a function.  That is the same trade `btree-nv`
makes with `PageRequest`, and for the same reason — the one operation
that touches the machine is the only one this package refuses to do.

## The device claim, and what it covers

`tests/embedded_probe.nv` builds for `--target=nrf52-qemu` and produces
a real Cortex-M4 ELF.  It covers `nvfsgeo`, `nvfsctz` and `nvfswear` —
the layout arithmetic, the skip-list walk and the wear and arbitration
rules — which between them own two `@value` structs of three `Int`s each
and no other type at all.  Those three are what a firmware runs in a
loop: seek into a file, decide whether a commit fits, pick a free block,
arbitrate a pair.

`nvfsmeta`, `nvfsvol` and `nvfsop` are **not** claimed.  A record holds
`Bytes`, a directory fold builds a list of entries, and the machine
holds an action whose payload is a buffer — every one of them allocates
by construction and not by accident.  A firmware that wants those wants
a heap; a firmware that wants to walk a chain and commit a record into a
buffer it already owns wants exactly the three that are claimed.

What the tier checks at `0.0.1` is narrower than it will be: every body
is a `todo()`, so what links today is the signatures and the types.
There is no walk in the binary yet, so the probe does not prove a seek
is allocation-free; it proves that nothing in the shape of the surface
needs an allocator or a host.  Keeping it green once the bodies land is
a named cost of the implementation step.

`heapless-nv` is the package a firmware pairs this with, and it is
deliberately **not** a dependency: the buffers — the block image, the
record being built — are the caller's, which is the rule heapless-nv
itself follows.

## The format, in one page

A **metadata pair** is two erase blocks holding an append-only log:

```
[ revision u32 | pad to meta_offset | record | record | 0xFF free … ]
  meta_offset = max(4, prog_size)
  record      = [ type u8 | flags u8 | len u16 | payload | crc32c u32 ]
```

The block with the higher **sealed** revision is active.  Four rules
carry the whole of the crash resistance, and each of them is a function
in this package rather than a paragraph:

- **Every mutation is exactly one self-CRC'd record**, so per-record
  atomicity is the entire story.  A rename is **one** record — not a
  tombstone and an entry — which is why a power cut cannot lose the
  file, and why a cross-directory move is `NvfsNotSupported` rather than
  done unsafely: that one would be two records in two pairs.
- **A compaction seals the revision word last**, on its own prog page.
  `nvfsgeo.meta_offset` is the padding that makes it possible, and
  `orbit/novofs` learned it as a format change that is not backward
  compatible: the first shape packed records against the revision word,
  so one prog page could carry both a partial record and a live
  revision, and neither outcome was recoverable.
- **A torn append ends the log and marks the block full**
  (`nvfswear.append_allowed`).  NOR programming only clears bits, so a
  garbage tail cannot be rewritten and the next commit must compact.
- **Free means "not reached from the root"**, never a free list.  A
  crash between allocating a block and committing the record that names
  it leaves the block unreachable, and unreachable is free — so the
  crash costs nothing and there is no stale list to repair.  The
  rotating cursor is a wear policy on top of that, and changing its pick
  order can waste a block and can never corrupt a volume.

A file at or below `inline_max` lives in its directory entry and costs
no data block.  A larger one is a **CTZ skip-list**: a backwards chain
whose block at index `n` points at `n-1`, `n-2`, `n-4`, … so a seek is
O(log n).  Writing appends and never rewrites an earlier block, which is
what makes a file copy-on-write by construction.

**One deliberate divergence from littlefs**, inherited from
`orbit/novofs` and named so it can be revisited: the CTZ capacity is
**uniform**.  littlefs packs `ctz(n) + 1` pointers into block `n`, so
most blocks carry one and the capacity depends on the index; this
reserves K slots in every block, which wastes `4 * (K - ctz(n) - 1)`
bytes per block and makes `offset -> index` a single division.
`nvfsctz.capacity` is the one function that would change.

## What `orbit/novofs` keeps, and what it takes

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

## The reference implementation

littlefs (BSD-3-Clause) — its `DESIGN.md` is the specification for the
metadata pair, the CTZ skip-list and the wear rules, and `lfs_config` is
the shape `NvfsAction` inverts.  `orbit/novofs` is the second reference:
it is this format already written in novo-lang, with a 110-check
functional suite, a 1096-cut power-loss harness and twelve CLI checks
behind it, and those are the vectors the bodies will be measured
against.  The on-disk format is littlefs-*inspired* rather than
littlefs-compatible, and it stays that way: a volume written by one will
not mount on the other.

## Status

Every function is `todo()`.  The six suites under `tests/` are red on
`not implemented`, which is the expected result until the bodies land,
and `tests/embedded_probe.nv` is not a test — it is the device claim,
built by `scripts/shard_audit.sh`'s `core-embedded` row.

```console
$ novo test tests/nvfsgeo_tests.nv
$ novo test tests/nvfsctz_tests.nv
$ novo test tests/nvfswear_tests.nv
$ novo test tests/nvfsmeta_tests.nv
$ novo test tests/nvfsvol_tests.nv
$ novo test tests/nvfsop_tests.nv
```
