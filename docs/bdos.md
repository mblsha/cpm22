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
    info = info_ptr; linfo = low_byte(info_ptr)
    save_user_stack_to(lstack); aret = 0
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
    if C == 0xFF: return coninf() if constf() else 0
    if C == 0xFE: return constf()
    conoutf(C); return C

def get_iobyte(info): return ioloc
def set_iobyte(info): ioloc = C
def print_string(info): print_until_dollar(BC)

def buffered_read(info):
    # info -> [maxlen, curlen, buffer...]
    editor_state = read_line_with_echo(info)
    return editor_state.length

def con_status(info): return conbrk()
def get_version(info): return dvers

def reset_disks(info):
    rodsk = 0; dlog = 0; curdsk = 0; dmaad = tbuff
    bios.select(0); setdata()

def select_disk(info):
    desired = linfo & 0x1F
    if desired != curdsk: curdsk = desired; bios.select(desired)
    return curdsk

def open_file(info):
    fcb = info; clrmodnum(fcb); reselect(fcb); return open(fcb)

def close_file(info):
    fcb = info; reselect(fcb); return close(fcb)

def search_first(info):
    fcb = info; clrmodnum_if_not_wildcard(fcb)
    reselect(fcb); status = search(fcb)
    dir_to_user(status, dmaad); return status

def search_next(info):
    reselect(info_from(searcha)); status = searchn(info_from(searcha))
    dir_to_user(status, dmaad); return status

def delete_file(info):
    fcb = info; reselect(fcb); status = delete(fcb); copy_dirloc(status); return status

def seq_read(info):
    fcb = info; reselect(fcb); return seqdiskread(fcb)

def seq_write(info):
    fcb = info; reselect(fcb); return seqdiskwrite(fcb)

def make_file(info):
    fcb = info; clrmodnum(fcb); reselect(fcb); return make(fcb)

def rename_file(info):
    fcb = info; reselect(fcb); status = rename(fcb); copy_dirloc(status); return status

def get_login_vector(info): return dlog
def get_current_disk(info): return curdsk
def set_dma(info): dmaad = info; return setdata()
def get_alloc_vector(info): return alloca

def set_read_only(info):
    set_ro(curdsk); return rodsk

def get_read_only(info): return rodsk

def set_attributes(info):
    fcb = info; reselect(fcb); status = indicators(fcb); copy_dirloc(status); return status

def get_dpb(info): return dpbaddr

def set_or_get_user(info):
    if linfo == 0xFF: return usrcode
    usrcode = linfo & 0x1F; return usrcode

def random_read(info):
    fcb = info; reselect(fcb); return randiskread(fcb)

def random_write(info):
    fcb = info; reselect(fcb); return randiskwrite(fcb)

def file_size(info):
    fcb = info; reselect(fcb); return getfilesize(fcb)  # writes ranrec

def set_random_record(info):
    fcb = info; return setrandom(fcb)

def mask_vectors(info):
    mask = ~fetch_word(info)
    dlog &= mask; rodsk &= mask; return rodsk

def func_ret(info): return aret  # placeholder for 38/39

def random_write_zerofill(info):
    fcb = info; reselect(fcb); seqio = 2
    if rseek1(fcb): diskwrite(fcb)
```

## Key Internal Helpers (what they take/do)
- `read (console editor)`: Uses `info` as pointer to (maxlen, curlen, buffer), tracks `column/strtcol`, echoes via `ctlout/tabout`, handles Ctrl-X/U/R/P/EOL and Ctrl-C reboot; writes final length byte.
- `reselect (FCB)`: Reads drive code from `info` FCB[0]; if auto-select (0–31), saves `olddsk`, sets `fcbdsk`, updates `curdsk` via `curselect`, stamps user code into FCB[0]; sets `resel` flag for cleanup in `goback`.
- `goback/retmon`: On exit, if reselection occurred, restores FCB[0] drive and previous `curdsk`, then restores user stack (`entsp`) and returns `aret` in `A`/`B`.
- Disk helpers: `seqdiskread/seqdiskwrite` manage current record (`rcount`), extent (`extval`), allocation map (`alloca`), and DMA; `randiskread/randiskwrite/rseek1` use random record field; `getfilesize` scans extents to compute `ranrec`; `dir_to_user`/`copy_dirloc` move directory entries to DMA; `set_ro` flips `rodsk` bit and sets error flags.

## In-Memory Structures
- **Global state:** `usrcode` (current user), `curdsk` (selected drive), `info` (user parameter pointer), `aret` (return value), `dmaad` (current DMA address).
- **Console editor state:** `column/strtcol/compcol/listcp/kbchar` plus `lstack` for BDOS stack.
- **Disk/login vectors:** `dlog` (logged-in drives bitmask), `rodsk` (read-only drives), `efcb` (empty dir entry template).
- **Per-select drive pointers (written by BIOS select):** `curtrka`, `curreca`, `buffa` (directory DMA), `dpbaddr` (DPB), `checka` (checksum vector), `alloca` (allocation vector), `cdrmaxa` (dir max pointer).
- **DPB-derived fields:** `sectpt` (sectors/track), `blkshf`/`blkmsk` (block geometry), `extmsk`, `maxall`, `dirmax`, `dirblk`, `chksiz`, `offset`; contiguous block referenced via `dpblist`.
- **FCB layout (at `tfcb` by default):** byte 0 drive code (0=current, 1=A,...), bytes 1–8 name, 9–11 type, 12 extent, 13 S1 (unused), 14 module number/flags, 15 record count, 16–31 allocation map, 32 next record, 33–34 random record.
- **Working variables:** `searcha/searchl` (search cursor), `dirloc` (directory position), `rcount/extval` (current record/extents), `vrecord/arecord/arecord1` (logical/physical record tracking), `seqio` (sequential vs random), `tranv` (sector translation vector), `single` (allocation map entry width), `resel/fcbdsk/olddsk/linfo` (drive-selection bookkeeping).
