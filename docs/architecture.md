# CP/M 2.2 Architecture Cheatsheet

## CPU and Registers
- 8080/Z80-compatible: `A` (accumulator), `B/C`, `D/E`, `H/L` pairs; `SP`, `PC`, and flags (`Z`, `C`, `S`, `P`, `AC`).
- BDOS calling convention: `C` = function number, `DE` = pointer (FCB, DMA, buffer, etc.), returns in `A` (and `B`/`HL` for some routines via `aret`).

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

## Reset/Init Flow
- Cold/warm boot enters at `0000h`, which transfers to CCP. CCP uses BDOS `initf` to detect `$$$.SUB`, sets user/disk, then prompts.
- BDOS saves caller stack to `lstack` and restores on return; `goback` cleans up any auto drive selection before returning to the caller’s stack/PC.

## Writing/Running COM Programs
- Load address is fixed at `0100h`; no relocation or header. Return by `RET` to CCP or by `JP 0000h`/`CALL 0005h` with function 0 to warm boot.
- Default FCBs: `005Ch` (first) and `006Ch` (second). The command tail at `0080h` starts with length byte then ASCII args.
- Default DMA is `0080h`; change with BDOS func 26 if you need a different buffer.
- Use BDOS calls via `CALL 0005h`: set `C` to the function, set `DE`/`HL` as required, handle return in `A` (and `HL` for directory copies). Keep `SP` within your TPA.
