# CP/M 2.2 Architecture Cheatsheet

## CPU and Registers
- 8080/Z80-compatible 8-bit registers: `A` (accumulator), `B`, `C`, `D`, `E`, `H`, `L`; 16-bit pairs: `BC`, `DE`, `HL`. Pointers: `SP` (16-bit stack), `PC` (16-bit program counter). Flags: `Z`, `C`, `S`, `P`, `AC`.
- BDOS calling convention: `C` (8-bit) = function number, `DE` (16-bit) = pointer (FCB, DMA, buffer, etc.), returns in `A` (8-bit) and sometimes `HL`/`B` (16-bit via `aret`).

## Memory Map (44K build in this repo)
- `0000h`: warm boot vector (JMP to CCP/BIOS); rebooting user programs jump here or call BDOS func 0.
- `0003h`: I/O byte (console/list/punch/reader selection).
- `0005h`: BDOS entry (JMP bdose); user programs call here.
- `0006h`: Address field of BDOS jump (used by BIOS).
- `0009h/000Ch/000Fh`: Error handler vectors (permanent/select/RO disk) into BIOS.
- `005Ch`: Default FCB (`tfcb`).
- `0080h`: Default DMA buffer and command tail (length byte at 80h, chars from 81h).
- `0100h`: Transient Program Area (TPA) start; COM files load here.
- `9400h`: CCP (from this repo’s Makefile origin).
- `9C00h`: BDOS.
- `AA00h` and above (approx): BIOS for the 44K layout; top of RAM holds BIOS jump table.
- TPA ends just below CCP (`93FFh` in this layout), so user programs have ~0x92FF bytes minus their own stack/data.

## In-Memory Structures (Kaitai Struct style)
```yaml
meta:
  id: cpm_command_tail
  endian: le
seq:
  - id: len
    type: u1          # CP/M stores length in the first byte at 0x80
  - id: text
    type: str
    size: len
    encoding: ascii
  - id: padding
    size-eos: true    # remaining bytes up to 128 total
```
```yaml
meta:
  id: cpm_fcb
  endian: le
seq:
  - id: drive
    type: u1          # 0=current, 1=A, ...
  - id: filename
    type: str
    size: 8
    encoding: ascii   # space-padded
  - id: filetype
    type: str
    size: 3
    encoding: ascii   # space-padded
  - id: extent
    type: u1
  - id: s1_reserved
    type: u1
  - id: s2_modnum
    type: u1
  - id: record_count
    type: u1
  - id: alloc_map
    type: u1
    repeat: expr
    repeat-expr: 16   # bytes 16–31
  - id: next_record
    type: u1          # byte 32
  - id: random_record
    type: u2le        # bytes 33–34, used by random I/O
```
```yaml
meta:
  id: ccp_submit_state
  endian: le
seq:
  - id: submit_flag
    type: u1          # 0=no submit, 0xFF active
  - id: submit_fcb
    type: cpm_fcb     # starts with "$$$.SUB"
  - id: module_number
    type: u1
  - id: record_count
    type: u1
  - id: disk_map
    type: u1
    repeat: expr
    repeat-expr: 16
  - id: current_record
    type: u1
```
```yaml
meta:
  id: bdos_drive_runtime
  endian: le
seq:
  - id: sectors_per_track
    type: u2
  - id: block_shift
    type: u1
  - id: block_mask
    type: u1
  - id: extent_mask
    type: u1
  - id: max_allocation
    type: u2
  - id: dir_max_entry
    type: u2
  - id: dir_reserved_bits
    type: u2
  - id: checksum_size
    type: u2
  - id: track_offset
    type: u2
```
These definitions cover the default command tail at `0x80`, the 32-byte CP/M FCB with its random-record extension, CCP’s submit state block, and the BDOS per-drive runtime fields derived from the active DPB (`sectpt`..`offset`).

## Reset/Init Flow
- Cold/warm boot enters at `0000h`, which transfers to CCP. CCP uses BDOS `initf` to detect `$$$.SUB`, sets user/disk, then prompts.
- BDOS saves caller stack to `lstack` and restores on return; `goback` cleans up any auto drive selection before returning to the caller’s stack/PC.

## Writing/Running COM Programs
- Load address is fixed at `0100h`; no relocation or header. Return by `RET` to CCP or by `JP 0000h`/`CALL 0005h` with function 0 to warm boot.
- Default FCBs: `005Ch` (first) and `006Ch` (second). The command tail at `0080h` starts with length byte then ASCII args.
- Default DMA is `0080h`; change with BDOS func 26 if you need a different buffer.
- Use BDOS calls via `CALL 0005h`: set `C` to the function, set `DE`/`HL` as required, handle return in `A` (and `HL` for directory copies). Keep `SP` within your TPA.
