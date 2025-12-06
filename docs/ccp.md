# CCP Listing Guide

- Purpose: Console Command Processor handling command parsing, built-in commands (`DIR`, `ERA`, `TYPE`, etc.), and program loading. It starts at `ccpstart`, with a warm-start path at `ccpclear`.
- Buffering: Command buffer lives at `combuf` with length tracked via `comlen`. Warm starts clear this buffer; cold starts may execute a preloaded command.
- BDOS interface: Calls the BDOS entry at `0005h` (`bdos` equate) for console I/O, file search, open/close, and disk selection. BDOS address is fixed via the `bdosl` equate computed from `origin`.
- Build location: Default origin is set via `origin` (Makefile uses `9400h`), with BDOS assumed 0x800 bytes higher. Adjust `origin` only if you also update BDOS placement.
- Style: 8080 assembly with one instruction per line, lower-case directives, labels at column 1. Strings use double quotes; comments begin with `;`. Serial number notes and debugging flag (`testing`) are documented near the top.

### Editing Tips
- Keep built-in command tables and jump tables aligned; many routines depend on fixed offsets.
- When changing console behavior, update shared equates for control characters to keep CCP and BDOS expectations in sync.

## Routines (Python-style pseudocode)
```python
def ccpstart(c_user_disk):
    comlen = 0
    diska = (c_user_disk << 4) | (c_user_disk & 0x0F)
    submit = initialize()          # BDOS initf; returns FFh if $$$.SUB
    cdisk = c_user_disk & 0x0F
    select(cdisk)
    return ccp_loop()

def readcom():
    if submit:
        select(0)
        open(subfcb); subcr = subrc - 1
        diskread(subfcb)            # last record into buff
        copy(buff, combuf, 128); close(subfcb)
        delete(subfcb); select(cdisk)
    else:
        bdos_rbuff(maxlen, comlen, combuf)
    uppercase(combuf[:comlen]); combuf[comlen] = 0
    comaddr = combuf; return combuf

def fillfcb(offset=0):
    fcb = comfcb + offset; staddr = comaddr
    sdisk = parse_drive_prefix(comaddr) or 0
    name, type, comaddr = parse_name_and_type(comaddr)  # pads with ' ' or '?'
    fcb.drive = sdisk or cdisk; fcb.name = name; fcb.type = type
    zero(fcb[12:16]); zero(fcb[16:32]); return count_qmarks(name+type)

def intrinsic_index():
    for i, token in enumerate(["DIR ", "ERA ", "TYPE", "SAVE", "REN ", "USER"]):
        if comfcb.name_type_prefix_matches(token): return i
    return None

def ccp_loop():
    while True:
        prompt(current_drive_letter())
        readcom()
        setdma(buff); cdisk = current_drive()
        qmarks = fillfcb()
        if qmarks > 0: error_from(staddr); continue
        if sdisk: return userfunc()      # transient with drive override
        idx = intrinsic_index()
        dispatch_intrinsic(idx) if idx is not None else userfunc()

def direct():  # DIR
    if comfcb.name_is_blank(): comfcb.name_type = "???????????"
    setdisk(); entries = []
    if searchcom(): entries.append(dir_entry_from_dma())
    while searchn(): entries.append(dir_entry_from_dma())
    print_dir(entries, four_per_line=True, skip_system=True)
    resetdisk(); return

def erase():   # ERA
    setdisk()
    if fillfcb() == 11 and confirm("ALL (Y/N)?") is False: return
    status = delete(comfcb); print("NO FILE") if status == 0xFF else None
    resetdisk()

def type():    # TYPE
    setdisk()
    if not open(comfcb): print("NO FILE"); resetdisk(); return
    while True:
        if not diskread(comfcb): print("READ ERROR"); break
        for ch in buff:
            if ch == 0x1A: resetdisk(); return
            printchar(ch)
            if break_key(): resetdisk(); return

def save():    # SAVE n,<file>
    sectors = parse_number_from_comfcb()
    if fillfcb(): error_from(staddr); return
    setdisk(); delete(comfcb); make(comfcb)
    comrec = 0; dma = tran
    for _ in range(sectors * 16):
        setdma(dma); dma += 128
        if not diskwrite(comfcb): print("NO SPACE"); break
    close(comfcb); setdmabuff(); resetdisk()

def rename():  # REN old=new
    if fillfcb(): error_from(staddr); return
    left_drive = sdisk; left = comfcb.clone()
    right_q = fillfcb(offset=16); right_drive = sdisk
    if left_drive and right_drive and left_drive != right_drive: error()
    if not search(left): print("NO FILE"); return
    if search(right): print("FILE EXISTS"); return
    renam(comfcb); resetdisk()

def user():    # USER n
    n = parse_number_from_comfcb()
    if n >= 16 or comfcb.name_not_blank(): error_from(staddr); return
    setuser(n)

def userfunc():  # load transient
    if comfcb.name_blank() and sdisk: select(sdisk-1); return
    ensure_type(comfcb, "COM"); setdisk()
    if not open(comfcb): resetdisk(); error_from(staddr); return
    dma = tran
    while True:
        setdma(dma)
        if not diskread(comfcb): break
        dma += 128
        if dma > tranm: load_error()
    if last_read_return != 1: load_error()
    resetdisk()
    copy_fcb_to_default(comfcb)
    copy_command_tail_to_buff(combuf, buff)
    setdmabuff(); saveuser()
    jump(tran); restore_and_select(cdisk)
```

## In-Memory Structures
- `maxlen`/`comlen`/`combuf`/`comaddr`: Command buffer length byte, current length, 128-byte text area, and scan pointer.
- `staddr`: Points to the start of the token currently being parsed (used for `comerr` display).
- `submit/subfcb/submod/subrc/subcr`: Tracks `$$$.SUB` processing; FCB plus module/record counters and disk map.
- `comfcb/comrec`: Working FCB for current command and its record counter; reused for transient loads and intrinsic file ops.
- `buff` (external BDOS address)/`bptr`: DMA buffer for file I/O and pointer within it (TYPE).
- `dcnt/cdisk/sdisk`: Directory index from last BDOS search, current drive, and drive override specified in command (0 means none).
- Stack at `stack` (16 words reserved) initialized in `ccpstart`.
