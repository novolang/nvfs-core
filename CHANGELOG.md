# Changelog

All notable changes to nvfs-core are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## 0.1.0 — 2026-11-03

### Changed

- Internal: ladders rewritten as match. The operation phases, the metadata-pair dispatch and the volume checks read as one match on the phase or the code. No signature, constant or behaviour changed.

This release requires novo 0.23.0 or later.

## 0.1.0 — 2026-10-12

The bodies.  Every `todo()` in the six modules is implemented, the eight
suites under `tests/` pass (100 tests), and the package's first working
release is published over the interface release: no signature changed and no
effect row was added, removed or narrowed.

### Added

- `nvfsop.begin_next(m, op)` — begin another operation on a machine that has
  already finished one, keeping its volume, its handle table and its action
  count.  **The interface had a gap.**  A handle is a number the MACHINE
  remembers and nothing on disk, and `begin_on` builds a fresh machine, so
  with only `begin` and `begin_on` a handle could never be used twice: every
  `NvfsReadFile`, `NvfsWrite`, `NvfsSeek` and `NvfsTruncate` answered
  `NvfsNotFound` on every reachable call, and the handle `NvfsOpen` answers
  was decorative.  A consumer that opens a file and then reads it needs
  exactly this call, which is why it is here rather than a table that lives
  in a test.

- `tests/nvfsop_ops_tests.nv` — the whole operation surface driven through
  `begin`/`begin_next` on a RAM device that counts what it was asked: 18
  tests over the directory operations, the handle table with more than one
  handle in it, the commit that has to compact and the one that cannot, a
  check over a damaged image, and the refusals a damaged image reaches (a
  root pair that is neither sealed nor blank, a superblock record whose
  payload does not decode, a directory whose pair was erased, a handle that
  outlives its file, and a chain larger than the volume allows).  Its device
  answers a short block on request and refuses a program into a byte that
  was not erased, so the two failures a real part produces — an erase that
  does not take, and a read shorter than the block it named — are reachable
  rather than described.
- The suites that were already here gained the cases the coverage run showed
  missing, one module at a time: `nvfsvol` the superblock payload refused by
  length and by version, the fold's arms that copy an entry through, and a
  payload that is not an entry; `nvfsmeta` the second block winning when its
  sealed revision is the higher one, in both orders; `nvfswear` a cursor
  that cannot be a block and an allocation over a device with nothing to
  allocate; `nvfsctz`, `nvfsgeo` and `nvfswear` their arithmetic's unusable
  inputs — a zero capacity, a negative offset, a shift past what an `Int`
  holds, a fifth magic byte, an alignment of zero — each answered rather
  than divided by.

### Changed

- `NvfsMachine` is widened: every field a step moves is `var` (so a resume
  advances the machine it was handed instead of rebuilding a literal per
  step), and the fields past `volume` hold what no signature carries — the
  first block's scan while a pair is arbitrated, the record being committed,
  the staged image, the handle table, where a chain read stops, the erase
  attempts in flight, and the two tallies a check's report carries.  The
  reason the walk is three `Int`s rather than an `NvfsCtzWalk` is a rule and
  not a preference: a `@value` struct may not be a boxed struct's field
  (SPEC § 14.3).
- A handle-based operation now refuses a handle the machine never issued
  BEFORE the walk that would resolve its path, which is the same
  refuse-before-use the two device operations already had.
- The resume family dispatches on a new predicate, `action_is_ack`, rather
  than on `action_is_write`: a `NvfsSync` changes nothing and still needs an
  acknowledgement, and answering a sync with `resume_read` is now
  `NvfsCorrupt` rather than a silent mis-step.
- An erase that failed is asked for again, up to `nvfswear.allocation_attempts`
  times (an erase is idempotent, and this is the bounded retry the
  `resume_ok` doc now describes); a program or a sync that failed is
  `NvfsIoFailed` with no retry, because a half-written page must not be
  written twice.
- `novo.lint`'s `redundant-bool` rule rewritten at all 56 sites in the
  suites, every one of them pre-existing (`x == false` → `not x`, which the
  linter itself suggests and which cannot change a predicate).  No `src/`
  file had one.
- Two wrong expected values, both fixtures that contradicted every other
  statement around them, are corrected rather than worked around:
  `tests/nvfsgeo_tests.nv` asserted `record_span(8, 8) == 8` where
  `record_size(8)` is 16 and `record_span` is `align_up` over it, and the
  device probe compared its own count against 27 where `score` counts 28
  checks.  In both cases the code that disagreed with the number is the code
  every neighbouring case agrees with.
- The suites, the README and `novo.toml` no longer claim the package is an
  interface release: the `**Status: NOT IMPLEMENTED**` block, the
  implementation-status table and the "every body is a `todo()`" sentence in
  each module are gone, and nothing replaced the first two.
- `novo.toml`: version `0.1.0`, `stability = "experimental"`, and the `novo`
  floor raised to `>= 0.20.2`, the version every result below was measured
  against.
- **The device claim now covers the whole package.**  Every function of
  `nvfsmeta`, `nvfsvol` and `nvfsop` carries `@tier(app)`: a device build
  compiles every module under `src/` and checks an unannotated function as
  device code, fail-closed, so without the annotation those three modules
  refuse with `E4000` at the first list, boxed struct or `str` call.  With
  it, `novo build --target=nrf52-qemu` on `tests/embedded_probe.nv` staged
  under `src/` compiles all six modules and `crc-nv` for a Cortex-M4, and
  the three modules that allocate are emitted in no image because the probe
  never reaches them.
- **`novo.lock` re-resolved `crc-nv` 0.1.4 → 0.1.6**, inside the range the
  manifest already declared (`^0.1.4`).  0.1.4 is the version that cannot be
  compiled for a device: its tables are built at run time and its update
  takes `Bytes`, so every function of it trips `E4000` in a device build, a
  program that depends on this one included.  0.1.6 is that release's fix
  (`bugs/orbit/closed/crc-nv-does-not-build-for-a-device.md`, `fixed_in:
  crc-nv 0.1.6`) and answers the same numbers, which is what
  `nvfsmeta.record_crc`'s published check value and the CRC cases in
  `tests/nvfsmeta_tests.nv` and `tests/nvfs_core_edge_tests.nv` assert.
- **A write at a cursor inside a file keeps the tail it did not reach.**
  `nvfsop.write_inline` built the new contents as the prefix before the
  cursor plus the bytes written, and dropped everything after them: a write
  at a cursor inside a file truncated it to the cursor plus the write,
  losing bytes the caller never wrote over, which is the opposite of what
  "write at its cursor" and the function's own comment say.  It now writes
  the prefix, the gap (a write past the end still leaves zeros), the bytes,
  and the tail, and `tests/nvfsop_ops_tests.nv` pins the tail.
- **An operation re-seeds its pair to the constant root pair.**
  `nvfsop.reset_op` cleared the walk's numbers and left `m.pair` where the
  last walk arrived — inside a subdirectory, most often — so an operation
  begun on a machine that had just walked into a directory resolved its path
  against THAT directory's log: `NvfsReadDir("/sub")` on a machine whose
  previous operation ended in `/sub` walked from `/sub`'s pair, looked for
  a name `sub` in `/sub`, and answered `NvfsNotFound` for a directory that
  was there.  The seed is the constant root pair rather than the pair the
  walk last saw, which is also correct after a compaction has flipped the
  two blocks' physical order: `arbitrate` re-derives the active block from
  the sealed revision, never from the order the pair was named in.

### Known

- **Not supported, and each one is named in the code that refuses it:**
  `NvfsMkdir` (a new pair's commit plus the parent's dirent are two pair
  slots this machine has one of), removing a directory (its tombstone would
  orphan whatever its pair holds), removing the root (there is no parent log
  to tombstone in), a rename across directories or of the root, growing a
  chain file, and truncating one.  Each needs the same missing piece — a
  walk that decides which blocks nothing reaches — and each answers
  `NvfsNotSupported` rather than doing half the job.  Files that live inline
  are created, written, read, seeked and truncated.
- **The handle table is the machine's.**  A driver that drops a machine drops
  its handles; `begin_next` keeps them across operations on one machine and
  nothing keeps them across processes, because nothing about a handle is on
  disk.
- **Free space is "entirely erased".**  A check's `blocks_used` counts every
  block that holds anything at all, which is the definition that never hands
  a live block to an allocator, and it is coarser than "reachable from the
  root" — the same walk the refusals above are missing.
- **A record's padding is invisible to a scan**, by design: the next record
  is reached by alignment, so a bit flipped in the erased bytes between two
  records is not damage and is not reported.
- **The device claim is about a compile, not about behaviour on a part.**
  `tests/embedded_probe.nv` builds for `nrf52-qemu` twice — where it sits,
  and staged under `src/` so that the dependencies are in the compilation
  too — and is not run in either form: what it proves is a property of the
  surface, that nothing in the closure it calls needs an allocator or an
  operating system.  Nothing here has been executed on a Cortex-M, and no
  read, program or erase has been timed on one.
- **`@tier(app)` is a promise the compiler takes at its word.**  It is what
  makes a device build accept the three host-face modules, and it is also
  what stops those modules from being checked: a function that allocates
  where the annotation says it may is not reported.  The narrower claim —
  that `nvfsgeo`, `nvfsctz` and `nvfswear` allocate nothing — is the one a
  device build checks on its own, because those three carry no annotation at
  all.
- **No path cache**, so every operation walks from the root, which is what
  the interface's "a handle is not a cache" prose already said.
- **21 arms are excused as unreachable, and each one says why at the site.**
  `novo pkg publish --dry-run` measures 97% of 999 instrumented lines under
  `src/` covered, with 21 regions excused by `cov: skip`; every one of them
  is in `nvfsop`, and every one is a `None` or negative arm that an
  invariant established a few lines earlier keeps out of reach — the
  per-operation phase maps' `else` arms, `first_scan` before a pair's second
  read, `name_bound` before a mount, the two operations `act` cannot see,
  the three `pos < 0` arms behind `handle_start`'s check, the chain walk's
  index being at or above its target by construction, and `clamp`'s callers
  having already refused a negative value.  They are kept rather than
  deleted because each is the refusal that a phase added later or a
  re-ordered check would need, and deleting a guard to raise a percentage is
  the wrong trade.  The other five modules are covered without excuse.

## 0.0.3 — 2026-10-06

README and comments only.  No signature, type or effect row changed, and
every body is still `todo()`.

### Changed

- The README names a block device, as embedded-hal-nv's `BlockDevice`
  trait defines it, as what performs this package's block actions, and
  the language's fixed-capacity collections (SPEC section 14.8) as a
  firmware's buffers.  It named heapless-nv, which is withdrawn.
- The README's "Running on a microcontroller" section says what the
  registry shows: all six modules at every tier, because the tier check
  admits a module that allocates.  The section's claim, that three
  modules allocate nothing, is unchanged.
- The example carries the interface stamp beside it.
- The probe command in the README was run against this release on
  2026-10-06.

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
