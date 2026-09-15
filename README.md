# nvfs-core

A **filesystem for raw flash** is one that manages the erase blocks of a
memory chip directly, with no disk controller underneath. This package is the
on-disk format and every operation of such a filesystem, in the style of
[littlefs](https://github.com/littlefs-project/littlefs), whose `DESIGN.md`
is the reference. It performs no input or output: it **answers** the block
reads and writes a caller must perform, and the caller performs them. It is
built on [crc-nv](https://novo-lang.org/packages/crc-nv).

**Status: NOT IMPLEMENTED — interface only.** Every function is declared with
its full signature, but every body is a `todo()` that panics when called. The
package is published so its design can be reviewed and depended on before it
is implemented. Version 0.1.0 will be the first working release.

## What it is

NOR flash has three properties that decide the whole design. A write can
only clear bits, never set them. Setting bits again means **erasing**, and
erasing works on a whole **block** at a time. A block wears out after some
tens of thousands of erases. On top of that, the power can go at any moment,
including in the middle of a write.

A filesystem for such a part is therefore append-only, copy-on-write, and
careful about the order in which it writes things.

A **metadata pair** is two erase blocks holding the same directory, as an
append-only log. Each block begins with a **revision number** and then a
sequence of records. The block whose revision is higher and **sealed** is the
live one. When a block fills, the filesystem **compacts**: it writes the live
contents into the other block of the pair, and seals it last.

A **record** is one change: an entry created, an entry removed, a file
renamed. Each record carries its own checksum. That is the whole of the crash
resistance: a record that was completely written is valid, and one that was
not is not.

```
[ revision u32 | pad to meta_offset | record | record | 0xFF free ... ]
  record = [ type u8 | flags u8 | len u16 | payload | crc32c u32 ]
```

A small file lives inside its directory entry and costs no block of its own.
A larger one is a **CTZ skip-list**: a chain of blocks in which the block at
index `n` holds pointers back to `n - 1`, `n - 2`, `n - 4` and so on, so that
seeking to any position takes a number of steps proportional to the logarithm
of the file's length. Writing appends and never rewrites an earlier block.

The thing that makes this package usable without hardware is that the
**direction of the call is reversed**. littlefs is given four function
pointers and calls them from inside. Here an operation is a machine: it is
started, and each step answers either an action to perform, or the result, or
a failure. A caller performs the action and resumes the machine with the
answer.

```
NvfsRead(block, off, len)    NvfsProg(block, off, data)
NvfsErase(block)             NvfsSync
```

| Quantity | Value |
| --- | --- |
| Blocks in a metadata pair | 2 |
| Erased byte | 0xFF |
| Records per mutation | 1 |
| Actions the machine can ask for | 4 |
| Pointers per block in the skip-list | the same number in every block |
| Revision number | 32 bits, wrapping after about 136 years at one compaction a second |

## Install

```
novo pkg add nvfs-core
```

## Example

```novo
use std.bytes
use nvfsop
use nvfsvol

fn main() [io]
    // The part's geometry: read size, program size, block size, block count.
    let g = nvfsvol.geometry(16, 16, 512, 32)

    // Begin a mount. Nothing has been read yet.
    var p = nvfsop.begin(g, nvfsvol.default_config(), NvfsMount)
    var running = true
    while running
        match p.step
            // The machine wants an action performed. A real driver talks to
            // the part here; this one answers as an erased device would.
            NvfsNeed(a) =>
                match a
                    NvfsRead(_, _, len) => p = nvfsop.resume_read(p.machine, bytes.zeros(len))
                    _                   => p = nvfsop.resume_ok(p.machine, true)
            NvfsDone(_) =>
                println("mounted")
                running = false
            NvfsFailed(_) =>
                println("no filesystem, so format it")
                running = false
```

Build and test with `novo pkg build` and `novo test`. Today `novo test` fails
on purpose: every test reaches a `not implemented` panic. The tests are the
specification the implementation will have to satisfy.

## What the package contains

| Module | Contents |
| --- | --- |
| `nvfsgeo` | The block and record layout as integer arithmetic, and the error set. |
| `nvfsctz` | The skip-list: how many pointers a block holds, which block an offset falls in, and a walk held as three integers. |
| `nvfswear` | Which block of a pair wins, where the next free block comes from, when a block should be moved, and the three power-loss rules. |
| `nvfsmeta` | The record log: building a record, scanning a block's bytes into records, and the three images a commit writes. |
| `nvfsvol` | The superblock, the four record types, the directory entries, and the fold that turns a log into a directory. |
| `nvfsop` | Every operation from mount to check, each as a machine, and the action it asks for. |

## How to choose an entry point

**A driver uses `nvfsop` and nothing else.** `begin` starts an operation,
`resume_read` answers a read, and `resume_ok` answers a write, an erase or a
sync. Every operation has the same shape, so the loop is written once.

**A firmware that only walks a chain uses `nvfsctz` directly.** Seeking into
a file, deciding whether a commit fits and picking a free block are
arithmetic, and those three modules do them with no allocator. See "Running
on a microcontroller".

**A tool that reads an image out of a file uses the same machine.** A QEMU
flash, a file-backed image and a real part differ in who performs the action
and in nothing else, so there is no trait to implement per device and nothing
to instantiate.

## The rules a user needs

1. **Every mutation is exactly one record, and the record carries its own
   checksum.** That is the whole of the crash story. A power cut either lands
   before the record, in which case nothing happened, or after it, in which
   case it all happened.
2. **A rename is one record, not a removal and a creation.** That is why a
   power cut cannot lose the file. A move between directories would be two
   records in two pairs, so it answers `NvfsNotSupported`.
   `nvfsvol.same_parent` is how a caller finds out before it starts.
3. **A compaction seals the revision word last, on its own program page.**
   `nvfsgeo.meta_offset` is the padding that makes that possible. Without it
   one program page could carry a partial record and a live revision at once,
   and neither outcome is recoverable.
4. **A torn append ends the log and marks the block full.** Programming NOR
   flash only clears bits, so a garbage tail cannot be rewritten. The next
   commit must compact. `nvfswear.append_allowed` is the rule.
5. **Free means not reachable from the root, and there is no free list.** A
   crash between allocating a block and committing the record that names it
   leaves the block unreachable, and unreachable is free. The crash therefore
   costs nothing and there is no stale list to repair.
6. **The allocation cursor is a wear policy, not correctness.** Changing the
   order it picks in can waste a block and can never corrupt a volume.
7. **A file at or below the inline limit lives in its directory entry.** It
   costs no data block. `nvfsctz.is_inline` answers whether a given length
   does.
8. **The skip-list reserves the same number of pointers in every block.**
   littlefs varies it with the index, which packs more data into most blocks
   but makes converting an offset to a block index a loop. Here it is one
   division, at the cost of some wasted bytes per block. `nvfsctz.capacity`
   is the one function that would change.
9. **A volume written by this format will not mount under littlefs, and the
   reverse.** The format is littlefs-inspired, not littlefs-compatible.
10. **`nvfsop.actions_taken` is what makes a power cut addressable.** Run a
    machine for a given number of actions, stop feeding it, and mount what
    the array holds. Every cut in a test suite is that loop.
11. **A file handle lives in the machine, not on the device.** A host that
    drops a machine drops its handles, and there is nothing on the flash to
    become stale.
12. **Revision wraparound is undefined.** The revision is a 32-bit number and
    wraps after about 136 years at one compaction a second.
    `nvfswear.revision_near_wrap` exists so that a long-lived logger detects
    the limit rather than meeting it.

## Running on a microcontroller

The package states that its modules run on a device with no heap allocator,
and the compiler checks that claim on every build. Here it covers `nvfsgeo`,
`nvfsctz` and `nvfswear`: the layout arithmetic, the skip-list walk, and the
wear and arbitration rules. Between them they own two `@value` structs of
three integers each and no other type at all. Those three are what a firmware
runs in a loop: seek into a file, decide whether a commit fits, pick a free
block, choose between the two blocks of a pair.

`tests/embedded_probe.nv` is that claim as a program that either builds or
does not.

```bash
novo build --target=nrf52-qemu tests/embedded_probe.nv
```

That command was run against this release. It produces a Cortex-M4
executable, `embedded_probe.elf`. The probe builds; it is not run, because
every function it calls is a `todo()` that would panic on the first line.

`nvfsmeta`, `nvfsvol` and `nvfsop` are outside the claim. A record holds
`Bytes`, a directory fold builds a list of entries, and an action's payload
is a buffer. Every one of those allocates by construction.

What the probe checks in this release is narrower than what it will check.
Every body is a `todo()`, so what links today is the signatures and the
types: nothing in the shape of the surface needs an allocator or an operating
system. It does not yet prove that a seek allocates nothing. Keeping it
building once the bodies land is a cost of the implementation step.

## What is not included

- **A flash driver.** Nothing here reads or writes a part. The four actions
  are what a driver performs.
- **Cross-directory rename.** See rule 2.
- **Compatibility with littlefs on disk.** See rule 9.
- **A buffer.** The block image and the record being built are the caller's.
  [heapless-nv](https://novo-lang.org/packages/heapless-nv) is where a
  firmware's would come from, and it is deliberately not a dependency for
  that reason.
- **A second checksum implementation.** The record checksum is crc-nv's.
- **A path cache.** Every operation walks from the root.

## Related packages

- [crc-nv](https://novo-lang.org/packages/crc-nv) is the record checksum.
- [heapless-nv](https://novo-lang.org/packages/heapless-nv) is the
  fixed-capacity storage a firmware pairs this with.
- [bbqueue-nv](https://novo-lang.org/packages/bbqueue-nv) is the other
  storage-shaped package for a device, for bytes in flight rather than bytes
  at rest.
- [dfu-nv](https://novo-lang.org/packages/dfu-nv) writes a firmware image to
  flash. It erases and programs the same kind of part, with no filesystem on
  it.
- [btree-nv](https://novo-lang.org/packages/btree-nv) is the on-disk index
  for a database. It takes the same shape as this package: the caller
  performs the page read and the library asks for it.
- `std.fs` in the standard library is the operating system's filesystem. It
  is what a program on a host uses, and there is none of it on a device.

## Tests

```bash
novo test                             # 60 tests
novo test tests/nvfsgeo_tests.nv      # 11: the layout arithmetic
novo test tests/nvfsctz_tests.nv      #  8: the skip-list and the seek
novo test tests/nvfswear_tests.nv     # 10: arbitration, allocation, the power-loss rules
novo test tests/nvfsmeta_tests.nv     #  8: the record log and the commit cycle
novo test tests/nvfsvol_tests.nv      # 12: the superblock, the entries, the fold
novo test tests/nvfsop_tests.nv       # 11: a format and a mount, driven to completion
```

littlefs's `DESIGN.md` is the specification for the metadata pair, the
skip-list and the wear rules, and its `lfs_config` is the shape the four
actions invert.

No test opens a device. Every read is answered out of a byte array the test
built by hand, and the device in `tests/nvfsop_tests.nv` is a `while` loop.

The tests compile today and fail at run, each on the `not implemented` panic
that is its body. That is the expected state of an interface release. They
turn green one at a time as bodies land.

## Implementation status

| Item | Implemented |
| --- | --- |
| `nvfsgeo.NvfsError`, `nvfsctz.NvfsCtzWalk`, `nvfswear.NvfsAlloc` | declared |
| `nvfsmeta.NvfsRecord`, `.NvfsScan`, `.NvfsPair`, `.NvfsPairState` | declared |
| `nvfsvol.NvfsGeometry`, `.NvfsConfig`, `.NvfsVolume`, `.NvfsSuperblock`, `.NvfsEntryKind`, `.NvfsEntry` | declared |
| `nvfsop.NvfsAction`, `.NvfsAnswer`, `.NvfsOp`, `.NvfsResult`, `.NvfsReport`, `.NvfsStep`, `.NvfsProgress`, `.NvfsMachine` | declared |
| `nvfsgeo`: the layout arithmetic and the geometry check | no |
| `nvfsctz`: the pointer arithmetic, the walk and the seek | no |
| `nvfswear`: arbitration, allocation, sealing and the wear counters | no |
| `nvfsmeta`: the record, the scan and the three commit images | no |
| `nvfsvol`: the superblock, the entries, the fold and the path handling | no |
| `nvfsop`: `begin`, `resume_read`, `resume_ok` and the accessors | no |

## Licence

Apache-2.0. See `LICENSE`.

<!-- docs/writing-a-readme.md is the style guide for this page. -->
