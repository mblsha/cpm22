# BDOS Listing Guide

- Purpose: Implements the CP/M 2.2 BDOS call layer, mediating user programs and the BIOS for console, disk, and file services.
- Entry and dispatch: Calls arrive at `bdose` with function number in `C` and parameters in `DE`; the routine saves state, swaps to a local stack, and jumps via `functab` to handlers (`func1`..). Return values are written through `aret`.
- BIOS linkage: Uses the BIOS jump table (`bios` symbol near the end) with offsets like `bootf`, `constf`, `readf`, and `writef`; these are 3-byte jumps laid out consecutively.
- Build location: Origin can be set with `origin`; Makefile builds at `9c00h`. The default in-source origin is `0800h` unless `test` is enabled. Keep BDOS aligned with CCP expectations (BDOS assumed 0x800 bytes above CCP origin).
- Serial number: Six-byte serial block appears early and is currently zeroed; adjust only if you need OEM tagging.
- Patch level: Includes Digital Research CP/M V2.2 Patch 01 (`patch1 equ 1`).

### Editing Tips
- Preserve stack sizing (`ssize` and `lstack`) and vector addresses used by error handlers (`pererr`, `selerr`).
- When adding or reordering BDOS functions, update `nfuncs`/`functab` consistently and consider impacts on CCP and applications expecting stable numbers.

## BDOS Functions (Python-style pseudocode)
```python
def bdos_dispatch(func, info_ptr):
    # args: C=function code, DE=info/FCB pointer; returns aret (A/B)
    info = info_ptr
    linfo = low_byte(info_ptr)
    save_user_stack_to(lstack)
    aret = 0
    return [
        warm_boot, con_input, con_output, reader_input,
        punch_output, list_output, direct_conio, get_iobyte,
        set_iobyte, print_string, buffered_read, con_status,
        get_version, reset_disks, select_disk, open_file,
        close_file, search_first, search_next, delete_file,
        seq_read, seq_write, make_file, rename_file,
        get_login_vector, get_current_disk, set_dma, get_alloc_vector,
        set_read_only, get_read_only, set_attributes, get_dpb,
        set_or_get_user, random_read, random_write, file_size,
        set_random_record, mask_vectors, func_ret, func_ret,
        random_write_zerofill
    ][func](info)

def warm_boot(info): bios.wbootf()  # no return
def con_input(info): return conech()
def con_output(info): return tabout(C)
def reader_input(info): return readerf()
def punch_output(info): return punchf(C)
def list_output(info): return listf(C)

def direct_conio(info):
    # args: C holds mode/value; returns char/status
    # returns: A (char or status)
    if C == 0xFF: return coninf() if constf() else 0
    if C == 0xFE: return constf()
    conoutf(C); return C

def get_iobyte(info): return ioloc  # returns: A
def set_iobyte(info): ioloc = C     # returns: A unchanged
def print_string(info): print_until_dollar(BC)  # returns: none

def buffered_read(info):
    # args: info -> [maxlen, curlen, buffer...]; returns new length
    # returns: A (length)
    editor_state = read_line_with_echo(info)
    return editor_state.length

def con_status(info): return conbrk()
def get_version(info): return dvers

def reset_disks(info):
    # args: none; clears vectors and selects drive 0
    # returns: A=curdsk after select
    rodsk = 0
    dlog = 0
    curdsk = 0
    dmaad = tbuff
    bios.select(0)
    setdata()

def select_disk(info):
    # args: linfo=desired drive; returns current drive
    # returns: A=curdsk
    desired = linfo & 0x1F
    if desired != curdsk:
        curdsk = desired
        bios.select(desired)
    return curdsk

def open_file(info):
    # args: info -> FCB address
    # returns: A status (0=ok, FFh=not found)
    fcb = info
    clrmodnum(fcb)
    reselect(fcb)
    return open(fcb)

def close_file(info):
    # args: info -> FCB address
    # returns: A status
    fcb = info
    reselect(fcb)
    return close(fcb)

def search_first(info):
    # args: info -> FCB address; returns search status
    # returns: A status (FFh if none), and fills DMA with dir entry
    fcb = info
    clrmodnum_if_not_wildcard(fcb)
    reselect(fcb)
    status = search(fcb)
    dir_to_user(status, dmaad)
    return status

def search_next(info):
    # args: searcha saved cursor; returns search status
    # returns: A status, DMA filled
    reselect(info_from(searcha))
    status = searchn(info_from(searcha))
    dir_to_user(status, dmaad)
    return status

def delete_file(info):
    # args: info -> FCB address
    # returns: A status
    fcb = info
    reselect(fcb)
    status = delete(fcb)
    copy_dirloc(status)
    return status

def seq_read(info):
    # args: info -> FCB address
    # returns: A status (0=ok, 1=EOF, others=errors)
    fcb = info
    reselect(fcb)
    return seqdiskread(fcb)

def seq_write(info):
    # args: info -> FCB address
    # returns: A status
    fcb = info
    reselect(fcb)
    return seqdiskwrite(fcb)

def make_file(info):
    # args: info -> FCB address
    # returns: A status
    fcb = info
    clrmodnum(fcb)
    reselect(fcb)
    return make(fcb)

def rename_file(info):
    # args: info -> FCB with old/new halves
    # returns: A status
    fcb = info
    reselect(fcb)
    status = rename(fcb)
    copy_dirloc(status)
    return status

def get_login_vector(info): return dlog          # returns: HL (aret)
def get_current_disk(info): return curdsk        # returns: A
def set_dma(info): dmaad = info; return setdata()# returns: none (A unchanged)
def get_alloc_vector(info): return alloca        # returns: HL (aret)

def set_read_only(info):
    # args: uses curdsk; marks rodsk and BIOS state
    # returns: A status
    set_ro(curdsk); return rodsk

def get_read_only(info): return rodsk            # returns: HL (aret)

def set_attributes(info):
    # args: info -> FCB address
    # returns: A status; dirloc in HL
    fcb = info
    reselect(fcb)
    status = indicators(fcb)
    copy_dirloc(status)
    return status

def get_dpb(info): return dpbaddr

def set_or_get_user(info):
    # args: linfo=0xFF to query else new user code
    # returns: A (user code)
    if linfo == 0xFF:
        return usrcode
    usrcode = linfo & 0x1F
    return usrcode

def random_read(info):
    # args: info -> FCB with ranrec
    # returns: A status
    fcb = info
    reselect(fcb)
    return randiskread(fcb)

def random_write(info):
    # args: info -> FCB with ranrec
    # returns: A status
    fcb = info
    reselect(fcb)
    return randiskwrite(fcb)

def file_size(info):
    # args: info -> FCB; writes ranrec
    # returns: A status; ranrec in FCB
    fcb = info
    reselect(fcb)
    return getfilesize(fcb)  # writes ranrec

def set_random_record(info):
    # args: info -> FCB ranrec
    # returns: none (A unaffected)
    fcb = info
    return setrandom(fcb)

def mask_vectors(info):
    # args: info points to word mask; clears bits in dlog/rodsk
    # returns: HL (aret) containing updated vectors
    mask = ~fetch_word(info)
    dlog &= mask
    rodsk &= mask
    return rodsk

def func_ret(info): return aret  # placeholder for 38/39

def random_write_zerofill(info):
    # args: info -> FCB with ranrec
    # returns: A status
    fcb = info
    reselect(fcb)
    seqio = 2
    if rseek1(fcb):
        diskwrite(fcb)
```

## Key Internal Helpers (what they take/do)
- `read (console editor)`: Uses `info` as pointer to (maxlen, curlen, buffer), tracks `column/strtcol`, echoes via `ctlout/tabout`, handles Ctrl-X/U/R/P/EOL and Ctrl-C reboot; writes final length byte.
- `reselect (FCB)`: Reads drive code from `info` FCB[0]; if auto-select (0–31), saves `olddsk`, sets `fcbdsk`, updates `curdsk` via `curselect`, stamps user code into FCB[0]; sets `resel` flag for cleanup in `goback`.
- `goback/retmon`: On exit, if reselection occurred, restores FCB[0] drive and previous `curdsk`, then restores user stack (`entsp`) and returns `aret` in `A`/`B`.
- Disk helpers: `seqdiskread/seqdiskwrite` manage current record (`rcount`), extent (`extval`), allocation map (`alloca`), and DMA; `randiskread/randiskwrite/rseek1` use random record field; `getfilesize` scans extents to compute `ranrec`; `dir_to_user`/`copy_dirloc` move directory entries to DMA; `set_ro` flips `rodsk` bit and sets error flags.

## In-Memory Structures (Kaitai Struct style)
```yaml
meta:
  id: bdos_globals
  endian: le
seq:
  - id: efcb
    type: u1          # empty dir entry template (0xE5)
  - id: rodsk
    type: u2le        # read-only drive bitmask
  - id: dlog
    type: u2le        # logged-in drive bitmask
  - id: dmaad
    type: u2le        # current DMA address (default 0x0080)
```
```yaml
meta:
  id: bdos_drive_select_block
  endian: le
seq:
  - id: cdrmaxa
    type: u2le        # pointer to current dir max
  - id: curtrka
    type: u2le        # current track
  - id: curreca
    type: u2le        # current record
  - id: buffa
    type: u2le        # directory DMA pointer
  - id: dpbaddr
    type: u2le        # disk parameter block pointer
  - id: checka
    type: u2le        # checksum vector pointer
  - id: alloca
    type: u2le        # allocation bitmap pointer
```
```yaml
meta:
  id: bdos_dpb_runtime
  endian: le
seq:
  - id: sectors_per_track
    type: u2le
  - id: block_shift
    type: u1
  - id: block_mask
    type: u1
  - id: extent_mask
    type: u1
  - id: max_allocation
    type: u2le
  - id: dir_max_entry
    type: u2le
  - id: dir_reserved_bits
    type: u2le
  - id: checksum_size
    type: u2le
  - id: track_offset
    type: u2le
```
```yaml
meta:
  id: bdos_working_vars
  endian: le
seq:
  - id: tranv
    type: u2le        # translate vector address (if any)
  - id: fcb_copied
    type: u1
  - id: rmf
    type: u1
  - id: dirloc
    type: u1
  - id: seqio
    type: u1
  - id: linfo
    type: u1
  - id: dminx
    type: u1
  - id: searchl
    type: u1
  - id: searcha
    type: u2le
  - id: tinfo
    type: u2le
  - id: single
    type: u1
  - id: resel
    type: u1
  - id: olddsk
    type: u1
  - id: fcbdsk
    type: u1
  - id: rcount
    type: u1
  - id: extval
    type: u1
  - id: vrecord
    type: u2le
  - id: arecord
    type: u2le
  - id: arecord1
    type: u2le
  - id: dptr
    type: u1
  - id: dcnt
    type: u2le
  - id: drec
    type: u2le
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
    repeat-expr: 16
  - id: next_record
    type: u1
  - id: random_record
    type: u2le
```
Use these structures to map BDOS data areas when inspecting memory dumps or debugging the running system. Addresses vary with origin; the order shown matches the layout in `bdos.asm`.
