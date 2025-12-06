Reference for the CP/M 2.2 BDOS and its BDOS-interface “front end” as implemented in this repository. The ABI is the classic CP/M one: user code calls `CALL 0005h` with the function in `C` and parameter in `DE`; BDOS returns a 16‑bit result in `HL` (mirrored into `B:A`). This module dispatches `C` through a function table; many BDOS services are thin wrappers around BIOS jump-table calls (console, disk).

---

## 1) ABI annotations (User ↔ BDOS) and BIOS call ABI

### User→BDOS ABI (what the *program* does)

**Entry point:** user program calls the low-memory BDOS gateway at `0005h` (label `bdosa equ 0006h` is the address field of `JMP BDOS`, but the classic call site is `CALL 0005h`).

**On entry (from user program):**

* `C` = BDOS function number (0..40 in this build)
* `DE` = “info” parameter:

  * sometimes a pointer (FCB, DMA address, console buffer, `$`-string)
  * sometimes a value in `E` with `D=0` (e.g., output character, disk number)

**On return to user:**

* `HL` = 16-bit return value (`aret`)
* `A`  = low byte of return value
* `B`  = high byte of return value
  (See `retmon`: loads `HL=aret`, then `A=L`, `B=H`, `RET`.)

**Important convenience done by BDOS front-end:**

* it sets `C := E` before calling the function handler, because many console/list/punch BIOS calls take the output byte in `C`.

### BDOS↔BIOS ABI (how this module calls BIOS)

CP/M BIOS is a jump table at label `bios`, with each entry 3 bytes (`JMP ...`). This code defines:

* `bootf  = bios + 3*0`  (cold boot)
* `wbootf = bios + 3*1`  (warm boot)
* `constf = bios + 3*2`  (console status)
* `coninf = bios + 3*3`  (console input)
* `conoutf= bios + 3*4`  (console output)
* … disk functions … up to `sectran = bios + 3*16`.

**Register conventions as used here (matching CP/M norms):**

* `CONST` (`constf`): returns status in `A` (this code tests bit0: `ANI 1`)
* `CONIN` (`coninf`): returns character in `A`
* `CONOUT` (`conoutf`): outputs character in `C`
* `LIST` (`listf`): outputs character in `C`
* `PUNCH` (`punchf`): outputs character in `C`
* `READER` (`readerf`): returns character in `A`
* Disk:

  * `SELDSK` (`seldskf`): input `C=disk#`, returns `HL=0` on error, else `HL` points to disk header
  * `SETTRK` (`settrkf`): input `BC=track`
  * `SETSEC` (`setsecf`): input `BC=sector`
  * `SETDMA` (`setdmaf`): input `BC=DMA address`
  * `READ` (`readf`): returns `A=0` if OK else nonzero error
  * `WRITE` (`writef`): input `C=writeType` (0 normal, 1 directory, 2 new block), returns `A=0` if OK else error
  * `SECTRAN` (`sectran`): input `BC=logical sector`, `DE=translation table`, returns `HL=physical sector`

---

## 2) Global state the code maintains (named like the assembly)

The fields below are globals in BDOS memory, kept here with the assembly names:

```pseudocode
# Console editing / echo state
byte compcol    # 0 = normal output, nonzero = "compute column only"
byte strtcol    # starting column of current input line
byte column     # current column position
byte listcp     # 0/1: also copy console output to list device (printer)
byte kbchar     # pending console char (typeahead buffer)

# Entry/exit stack handling
word entsp      # saved user SP at BDOS entry
stack lstack[...]  # local BDOS stack

# Common BDOS “ABI” scratch
byte usrcode    # current user number (0..31)
byte curdsk     # current disk (0=A, 1=B, ...)
word info       # saved DE parameter (pointer or value)
word aret       # 16-bit return value, returned in HL and also B:A
alias lret = low_byte(aret)  # many routines treat return as 8-bit

# Disk login/ro and DMA
word rodsk      # read-only disk bit vector
word dlog       # logged-in disks bit vector
word dmaad      # current DMA address (default 0080h)

# Per-selected-disk pointers copied from BIOS disk header
word cdrmaxa    # pointer to "current dir max" variable for this disk
word curtrka    # pointer to BIOS "current track" variable
word curreca    # pointer to BIOS "current record/sector index" variable
word buffa      # pointer to directory DMA buffer
word dpbaddr    # pointer to disk parameter block (DPB)
word checka     # pointer to checksum vector
word alloca     # pointer to allocation bitmap

# DPB fields copied locally when disk selected
word sectpt     # sectors per track
byte blkshf     # block shift factor
byte blkmsk     # block mask
byte extmsk     # extent mask
word maxall     # max allocation block number
word dirmax     # largest directory entry index
word dirblk     # reserved allocation bits for directory
word chksiz     # size of checksum vector (in records)
word offset     # track offset to start of disk data

# Directory scanning state
byte dptr       # offset of current directory entry within directory record
word dcnt       # directory entry counter
word drec       # directory record number (dcnt >> dskshf)

# File-operation scratch
word tranv
byte fcb_copied
byte rmf        # read-mode flag for open_reel
byte dirloc
byte seqio      # 0=random, 1=sequential, 2=rand-write-with-fill (func40 path)
byte linfo      # low(info) cached (often “disk” or “char”)
byte dminx
byte searchl
word searcha
byte single     # true if allocation map entries are 1 byte, else 2 bytes
byte resel
byte olddsk
byte fcbdsk
byte rcount
byte extval
word vrecord
word arecord
word arecord1
```

---

## 3) BDOS entry + dispatch pseudocode (label `bdose`)

```pseudocode
function BDOS_ENTRY(C_function, DE_info):
    # Save the “info” parameter for later and cache low byte.
    info  = DE_info
    linfo = low_byte(DE_info)

    # Default return is 0
    aret = 0x0000

    # Switch to local BDOS stack
    entsp = SP
    SP = &lstack

    fcbdsk = 0
    resel  = 0

    # Arrange to return through goback no matter which handler we jump to.
    push_return_address(goback)

    # If function number invalid: just return 0
    if C_function >= NFUNCS:
        goto goback

    # Convenience: many routines/BIOScalls expect an output byte in C
    C = low_byte(DE_info)   # i.e. C = E

    # Dispatch: function address = functab[C_function]
    target = functab[C_function]

    # Convention for handlers: DE receives the original info parameter
    DE = info

    jump_to(target)  # like PCHL
```

**Dispatch table meaning (by index):**

* 0 → BIOS `WBOOT` (warm boot)
* 1 → `func1` (console input w/ echo)
* 2 → `func2` = `tabout` (console output w/ tab expand)
* 3 → `func3` (reader input)
* 4 → BIOS `PUNCH`
* 5 → BIOS `LIST`
* 6..11 → direct i/o, iobyte, print string, buffered read, console status
* 12.. → disk/file functions, up to 40

---

## 4) Error handler vectors and behavior (pererr/selerr/roderr/roferr)

There are *four* low-memory vectors (relative offsets) that point at routines:

* `pererr → persub` (permanent error)
* `selerr → selsub` (select error)
* `roderr → rodsub` (write to R/O disk)
* `roferr → rofsub` (R/O file)

Core helper:

```pseudocode
function ERRFLG(msgPtr):
    CRLF()

    # Fill disk letter into "Bdos Err On X : "
    dskerr_char = 'A' + curdsk
    dskerr = dskerr_char           # dskerr is that single char in the string

    PRINT("Bdos Err On " + dskerr_char + " : ")
    PRINT(msgPtr)                  # prints until '$'

    # Return the next console char (no echo rules here)
    return CONIN()
```

Subroutines:

```pseudocode
function persub():
    ch = ERRFLG("Bad Sector$")
    if ch == CTRL_C: reboot()      # JMP 0000h
    return                         # otherwise ignore

function selsub():
    ERRFLG("Select$")
    reboot()

function rodsub():
    ERRFLG("R/O$")
    reboot()

function rofsub():
    ERRFLG("File $")
    reboot()
```

---

## 5) Console subsystem pseudocode (BDOS 1–11 support)

### BIOS-backed primitives used here

```pseudocode
function BIOS_CONST() -> byte
function BIOS_CONIN() -> byte
function BIOS_CONOUT(char in C)
function BIOS_LIST(char in C)
```

### `conin`: read one console char (with 1-byte typeahead buffer)

```pseudocode
function CONIN() -> byte:
    if kbchar != 0:
        ch = kbchar
        kbchar = 0
        return ch
    else:
        return BIOS_CONIN()
```

### `echoc`: decide if char should echo “normally”

Echo is “normal” for: CR, LF, TAB, BS, and any printable (>= ' ').
Returns “graphicOrSpecial” boolean.

```pseudocode
function IS_ECHOABLE(ch) -> bool:
    if ch == CR:      return true
    if ch == LF:      return true
    if ch == TAB:     return true
    if ch == CTRL_H:  return true
    return (ch >= ' ')
```

### `conech`: BDOS function 1: input with echo (but not caret-notation)

```pseudocode
function CONECH() -> byte:
    ch = CONIN()
    if IS_ECHOABLE(ch):
        C = ch
        TABOUT()              # does tab expansion & column tracking
    # else: control chars (except CR/LF/TAB/BS) are *not* echoed here
    return ch
```

### `conbrk`: “is a char ready?” (status + handles Ctrl-S flow, Ctrl-C reboot)

```pseudocode
function CONBRK() -> bool:
    if kbchar != 0:
        return true

    status = BIOS_CONST() & 1
    if status == 0:
        return false

    ch = BIOS_CONIN()

    if ch == CTRL_S:
        # “Stop screen”: wait for next char
        ch2 = BIOS_CONIN()
        if ch2 == CTRL_C:
            reboot()
        # otherwise, ignore the stop/start sequence
        return false

    kbchar = ch
    return true
```

### `conout`: write char in `C` to console, update column count (and maybe list-copy)

Key idea: `compcol!=0` means “do not really output, only simulate column position changes”.

```pseudocode
function CONOUT():
    if compcol == 0:
        # check for Ctrl-S stop behavior / early reboot handling
        savedC = C
        CONBRK()
        C = savedC

        BIOS_CONOUT(C)        # actual console output

        if listcp != 0:
            BIOS_LIST(C)      # optional echo to list device

    # Update column tracking regardless (unless rubout)
    UPDATE_COLUMN(C)
```

Column logic:

```pseudocode
function UPDATE_COLUMN(ch):
    if ch == RUBOUT:
        return

    column += 1

    if ch >= ' ':
        return  # printable

    # ch is control (below space): undo the +1
    column -= 1
    if column == 0:
        return

    if ch == CTRL_H:
        column -= 1
        return

    if ch == LF:
        column = 0
        return
```

### `ctlout`: print control chars as caret notation (`^C`), else print normally

This is used by line editor / buffered read, not by `conech`.

```pseudocode
function CTLOUT(ch in C):
    if IS_ECHOABLE(C):
        TABOUT()
        return

    # control char (non-echoable): print '^' then (ch | 0x40)
    saved = C
    C = '^'
    CONOUT()
    C = (saved | 0x40)
    TABOUT()
```

### `tabout`: expand tabs to spaces up to next 8-column boundary

```pseudocode
function TABOUT():
    if C != TAB:
        CONOUT()
        return

    do:
        C = ' '
        CONOUT()
    while (column & 0b111) != 0
```

### “Erase one screen position” helper used by line editing

```pseudocode
function PCTHL():               # output BS without affecting column count
    BIOS_CONOUT(CTRL_H in C)

function BACKUP_ONE_POSITION():
    PCTHL()
    BIOS_CONOUT(' ' in C)
    PCTHL()
```

### `crlf` and `print` ($-terminated string at BC)

```pseudocode
function CRLF():
    C = CR; CONOUT()
    C = LF; CONOUT()

function PRINT(ptrBC):
    while true:
        ch = *ptrBC
        if ch == '$': return
        ptrBC++
        C = ch
        TABOUT()
```

### Buffered console line input (BDOS function 10) — label `read`

This implements CP/M’s classic line editor:

* backspace (^H) and rubout delete previous character
* ^X deletes current line by erasing back to `strtcol`
* ^U deletes line with a visible “#<CRLF>” and restarts
* ^R repeats the line (“#<CRLF>” then re-echo)
* ^E forces a physical end-of-line (prints CRLF but continues line logically)
* ^P toggles list-copy output
* ^C as first char reboots

The buffer format at `info` is the standard CP/M one:

* `buf[0]` = max length
* `buf[1]` = current length (written by BDOS)
* `buf[2..]` = characters

```pseudocode
function BDOS_READ_BUFFERED_LINE(infoPtr):
    strtcol = column

    maxLen = mem8[infoPtr + 0]
    len    = 0
    # “cursor” is stored as pointer to buf[1] initially; characters go at ++cursor
    cursor = infoPtr + 1

    while true:
        ch = CONIN() & 0x7F     # strip parity

        if ch == CR or ch == LF:
            break  # end of line

        if ch == CTRL_H:        # backspace
            if len == 0:
                continue
            len -= 1
            # Column-safe erase: redraw to recompute and then back up.
            COLUMN_SAFE_DELETE(cursor, len)
            continue

        if ch == RUBOUT:        # delete previous (same as backspace, but echoes deleted char)
            if len == 0:
                continue
            # fetch last char (at cursor + len), then delete it
            last = mem8[cursor + len]
            len -= 1
            # The assembly echoes using the deleted char path; net effect is deletion.
            COLUMN_SAFE_DELETE(cursor, len)
            continue

        if ch == CTRL_E:        # physical eol: print CRLF and reset start column
            CRLF()
            strtcol = 0
            continue

        if ch == CTRL_P:        # toggle list-copy
            listcp = 1 - listcp
            continue

        if ch == CTRL_X:        # delete line back to strtcol (visual erase)
            while column > strtcol:
                column -= 1
                BACKUP_ONE_POSITION()
            # restart whole read from scratch
            return BDOS_READ_BUFFERED_LINE(infoPtr)

        if ch == CTRL_U:        # delete line and restart, with "#\r\n" marker
            PRINT_HASH_CRLF_AND_ALIGN()
            return BDOS_READ_BUFFERED_LINE(infoPtr)

        if ch == CTRL_R:        # repeat line: "#\r\n" then re-echo buffer
            PRINT_HASH_CRLF_AND_ALIGN()
            for i in 0 .. len-1:
                C = mem8[cursor + 1 + i]   # actual chars start at buf[2]
                CTLOUT(C)
            continue

        # normal character insertion
        if len >= maxLen:
            break  # buffer full

        len += 1
        mem8[cursor + len] = ch

        # echo it with caret-notation handling
        C = ch
        CTLOUT(C)

        # reboot if first char is Ctrl-C
        if ch == CTRL_C and len == 1:
            reboot()

    # Store final length into buf[1]
    mem8[infoPtr + 1] = len

    # CP/M convention: print CR on completion
    C = CR
    CONOUT()

    # return value is not explicitly set; BDOS typically returns 0
```

The “column-safe delete” in the assembly is implemented by reusing the `CTRL_R` path plus `compcol` to compute cursor position and backspace the difference. In pseudocode:

```pseudocode
function COLUMN_SAFE_DELETE(cursorToLenByte, newLen):
    oldCol = column

    # Repaint line to recompute column; then move cursor back to new position.
    PRINT_HASH_CRLF_AND_ALIGN()

    # Temporarily compute column-only while “echoing” the buffer,
    # to discover new cursor column after deletion.
    compcol = oldCol              # nonzero => conout won’t actually output in assembly
    column  = 0                   # effectively recomputed by replay

    for i in 0 .. newLen-1:
        C = mem8[(cursorToLenByte + 1) + i]
        CTLOUT(C)                 # advances column, but in assembly may be simulated

    # Now backup the number of positions needed to match new cursor placement.
    # Assembly computes: compcol := (oldCol - column) then BACKUP repeatedly.
    diff = oldCol - column
    compcol = 0                   # return to normal output mode
    for k in 1 .. diff:
        BACKUP_ONE_POSITION()
```

(Exact flag semantics differ, but user-visible behavior is: delete last char reliably even with tabs/control expansions.)

### BDOS functions 1–11

```pseudocode
# BDOS 0: warm boot
function BDOS_FUNC0(): BIOS_WBOOT()   # does not return

# BDOS 1:
function BDOS_FUNC1(): aret.low = CONECH()

# BDOS 2: console output with tab expansion
function BDOS_FUNC2(): TABOUT()

# BDOS 3: reader input
function BDOS_FUNC3(): aret.low = BIOS_READER()

# BDOS 4: punch output (BIOS)
# BDOS 5: list output (BIOS)

# BDOS 6: direct console I/O
#   if E==0xFF: input (if ready) else return 0
#   if E==0xFE: status
#   else: output E
function BDOS_FUNC6(E):
    if E == 0xFF:
        if (BIOS_CONST() == 0): aret.low = 0
        else aret.low = BIOS_CONIN()
    elif E == 0xFE:
        aret.low = BIOS_CONST()
    else:
        C = E
        BIOS_CONOUT(C)

# BDOS 7: get I/O byte at 0003h
function BDOS_FUNC7(): aret.low = mem8[0x0003]

# BDOS 8: set I/O byte at 0003h
function BDOS_FUNC8(E): mem8[0x0003] = E

# BDOS 9: print $-terminated string at DE
function BDOS_FUNC9(DE):
    PRINT(DE)

# BDOS 10: buffered console read
function BDOS_FUNC10(DE):
    BDOS_READ_BUFFERED_LINE(DE)

# BDOS 11: console status
function BDOS_FUNC11():
    aret.low = 1 if CONBRK() else 0
```

---

## 6) Disk / filesystem subsystem pseudocode

### Low-level helpers: `move`, error dispatch (`goerr`), disk I/O check

```pseudocode
function MEMMOVE(dstHL, srcDE, countC):
    for i in 0 .. countC-1:
        mem8[dstHL+i] = mem8[srcDE+i]

function GOERR(vectorPtrHL):
    # vectorPtr points to a word that contains an error routine address
    routine = mem16[vectorPtrHL]
    jump_to(routine)

function DIOCOMP(statusA):
    if statusA == 0:
        return
    GOERR(pererr)      # permanent error
```

Disk-buffer I/O:

```pseudocode
function RDBUFF():
    status = BIOS_READ()
    DIOCOMP(status)

function WRBUFF(writeType in C):
    status = BIOS_WRITE(writeType)
    DIOCOMP(status)
```

### Disk select (`selectdisk`): calls BIOS SELDSK and copies disk header + DPB

```pseudocode
function SELECTDISK() -> bool:
    # Inputs: curdsk
    C = curdsk
    header = BIOS_SELDSK(C)      # HL in assembly
    if header == 0:
        return false

    # Disk header layout (as used here):
    #   0: word tran (translate table)
    #   2: word cdrmax pointer
    #   4: word curtrk pointer
    #   6: word currec pointer
    #   8: word buffa pointer
    #  10: word dpbaddr pointer
    #  12: word checka pointer
    #  14: word alloca pointer
    tranv = mem16[header + 0]
    cdrmaxa = mem16[header + 2]
    curtrka = mem16[header + 4]
    curreca = mem16[header + 6]
    buffa   = mem16[header + 8]
    dpbaddr = mem16[header + 10]
    checka  = mem16[header + 12]
    alloca  = mem16[header + 14]

    # Copy DPB into local variables
    dpb = dpbaddr
    sectpt = mem16[dpb + 0]
    blkshf = mem8 [dpb + 2]
    blkmsk = mem8 [dpb + 3]
    extmsk = mem8 [dpb + 4]
    maxall = mem16[dpb + 5]
    dirmax = mem16[dpb + 7]
    dirblk = mem16[dpb + 9]
    chksiz = mem16[dpb + 11]
    offset = mem16[dpb + 13]

    # Determine allocation-entry size
    # single=true if maxall < 256 (i.e., maxall.high == 0)
    single = (high_byte(maxall) == 0)

    return true
```

### Home (`home`): BIOS HOME + reset disk-position vars

```pseudocode
function HOME():
    BIOS_HOME()
    mem16[curtrka] = 0
    mem16[curreca] = 0
```

### Seek logic (`seek_dir` and `seek`)

Directory record for an entry:

* `dcnt` is directory entry index
* `drec = dcnt >> dskshf` (`dskshf=2`, because 4 entries/record)

```pseudocode
function SEEK_DIR():
    drec = dcnt >> dskshf
    arecord = drec
```

The main seek (`seek`) moves BIOS “current track” and “current record within track” so that `arecord` becomes the current logical record, then selects track+sector via BIOS:

```pseudocode
function SEEK_TO_ARECORD():
    desired = arecord

    currec = mem16[curreca)
    curtrk = mem16(curtrka)

    # Adjust curtrk/currec so currec <= desired < currec+sectpt
    while desired < currec:
        currec -= sectpt
        curtrk -= 1

    while desired >= currec + sectpt:
        currec += sectpt
        curtrk += 1

    # Set physical track = curtrk + offset
    trackToSet = curtrk + offset
    BIOS_SETTRK(trackToSet in BC)

    # Write back updated “current” values
    mem16[curtrka] = curtrk
    mem16[curreca] = currec

    # Sector within track = desired - currec, then translate
    logicalSector = desired - currec
    physSector = BIOS_SECTRAN(logicalSector in BC, tranv in DE)  # returns HL
    BIOS_SETSEC(physSector in BC)
```

### DMA selection helpers

```pseudocode
function SETDMA(addr):
    BIOS_SETDMA(addr in BC)

function SETDATA_DMA():
    SETDMA(dmaad)

function SETDIR_DMA():
    SETDMA(buffa)
```

Directory record read/write:

```pseudocode
function RD_DIR_RECORD():
    SETDIR_DMA()
    RDBUFF()
    # (caller may SETDATA_DMA after)

function WR_DIR_RECORD():
    NEWCHECKSUM()       # initialize checksum slot for current record
    SETDIR_DMA()
    WRBUFF(writeType=1) # directory write
    SETDATA_DMA()
```

---

## 7) Directory entry access + allocation bitmap

### Address current directory entry within the directory buffer

Each directory record is 128 bytes and contains 4 entries of 32 bytes. `dptr` is the byte offset (0, 32, 64, 96).

```pseudocode
function GETDPTR_ADDR() -> ptr:
    return buffa + dptr
```

### `read_dir`: iterate directory entries one by one

```pseudocode
function END_OF_DIR() -> bool:
    return (dcnt == 0xFFFF)

function SET_END_DIR():
    dcnt = 0xFFFF

function READ_DIR_ENTRY(initializingChecksumFlag in C_bool):
    # increment directory counter
    dcnt += 1

    if dcnt > dirmax:
        SET_END_DIR()
        return

    # compute dptr = (dcnt & dskmsk) * 32
    # where dskmsk = dirrec-1 = 3
    entryIndexWithinRecord = dcnt & 0x3
    dptr = entryIndexWithinRecord * 32

    # if new record (entryIndexWithinRecord==0), seek and read record
    if entryIndexWithinRecord == 0:
        SEEK_DIR()
        RD_DIR_RECORD()
        CHECKSUM(initializingChecksumFlag)
```

### Allocation vector bit helpers (`getallocbit`, `set_alloc_bit`, `scandm`)

Conceptually:

* allocation bitmap is a bitset of blocks
* a directory entry’s disk map lists allocated blocks
* scanning the disk map sets/clears those bits

```pseudocode
function ALLOC_TEST_AND_ADDR(blockIndex) -> (byteValue, bytePtr, bitShiftCount):
    # blockIndex is BC in assembly
    byteIndex = blockIndex >> 3
    bitPos    = blockIndex & 7      # 0..7

    ptr = alloca + byteIndex
    v   = mem8[ptr]

    # Assembly rotates so the target bit ends up in bit0; shiftCount is for undo
    # We’ll represent it directly:
    return (v, ptr, bitPos)

function SET_ALLOC_BIT(blockIndex, bitValue0or1):
    (v, ptr, bitPos) = ALLOC_TEST_AND_ADDR(blockIndex)
    if bitValue0or1 == 1: v |=  (1 << bitPos)
    else:                 v &= ~(1 << bitPos)
    mem8[ptr] = v

function SCAN_DISK_MAP_AND_MARK_ALLOC(dirEntryPtr, markBit):
    # markBit: 1 = allocated, 0 = free
    # disk map starts at offset dskmap (16) and extends to end of entry
    if single:
        for i in 0 .. (32-16)-1:
            block = mem8[dirEntryPtr + 16 + i]
            if block != 0 and block <= maxall:
                SET_ALLOC_BIT(block, markBit)
    else:
        for i in 0 .. ((32-16)/2)-1:
            block = mem16[dirEntryPtr + 16 + 2*i]
            if block != 0 and block <= maxall:
                SET_ALLOC_BIT(block, markBit)
```

---

## 8) File Control Block (FCB) helpers used by read/write/open/close

### Extract current “record in extent” etc from FCB (`getfcb`, `setfcb`)

```pseudocode
function GETFCB_FIELDS(fcbPtr):
    vrecord = mem8[fcbPtr + nxtrec]      # next record number (0..127)
    rcount  = mem8[fcbPtr + reccnt]      # records in this extent
    extval  = mem8[fcbPtr + extnum] & extmsk

function SETFCB_FIELDS(fcbPtr):
    # seqio==1 (sequential) increments nxtrec; seqio==0 or 2 does not
    inc = 1 if seqio == 1 else 0

    mem8[fcbPtr + nxtrec] = (low_byte(vrecord) + inc)
    mem8[fcbPtr + reccnt] = rcount
```

### Disk map indexing (`dm_position`, `getdm`, `index`, `allocated`, `atran`)

Conceptually:

* disk map slot = (extval << (7 - blkshf)) + (vrecord >> blkshf)
* read block number from that slot (1 or 2 bytes depending on `single`)
* `atran` converts block# + vrecord into absolute record (`arecord`)

```pseudocode
function DM_POSITION() -> int:
    return (extval << (7 - blkshf)) + (low_byte(vrecord) >> blkshf)

function GETDM(fcbPtr, slot) -> int:
    base = fcbPtr + dskmap
    if single:
        return mem8[base + slot]
    else:
        return mem16[base + 2*slot]

function INDEX(fcbPtr):
    slot = DM_POSITION()
    arecord = GETDM(fcbPtr, slot)

function ALLOCATED() -> bool:
    return (arecord != 0)

function ATRAN():
    # arecord is a block number
    arecord1 = arecord << blkshf
    withinBlock = low_byte(vrecord) & blkmsk
    arecord = arecord1 | withinBlock
```

### File write flag in modnum (`setfwf`, and clearing it on write)

In CP/M 2.2, directory entry has:

* `modnum` field contains module number plus high bit as “file write flag”
* This BDOS sets the flag when opening/making; clears it when data is written

```pseudocode
const FWF_MASK = 0x80

function SET_FWF(fcbPtr):
    mem8[fcbPtr + modnum] |= FWF_MASK

function CLEAR_FWF(fcbPtr):
    mem8[fcbPtr + modnum] &= ~FWF_MASK

function CLRMODNUM(fcbPtr):
    mem8[fcbPtr + modnum] = 0
```

---

## 9) High-level directory operations: search, delete, rename, open, close, make

### `search` / `searchn`

Search compares the name/type fields with wildcards `?`, and compares extents using `compext`.

```pseudocode
function SEARCH_FIRST(fcbPtr, lengthC):
    dirloc = 0xFF
    searchl = lengthC
    searcha = fcbPtr

    SET_END_DIR()
    HOME()
    return SEARCH_NEXT()

function SEARCH_NEXT():
    READ_DIR_ENTRY(initializingChecksumFlag=false)

    if END_OF_DIR():
        SET_END_DIR()
        aret.low = 0xFF
        return NOT_FOUND

    # stop at logical end (cdrmax) if applicable
    if DIR_ENTRY_IS_EMPTY_OR_BEYOND_CDRMAX():
        SET_END_DIR()
        aret.low = 0xFF
        return NOT_FOUND

    dirEntry = GETDPTR_ADDR()

    # Compare first searchl bytes
    for i in 0 .. searchl-1:
        want = mem8[searcha + i]
        have = mem8[dirEntry + i]

        if want == '?': continue
        if i == ubytes: continue  # ubytes field isn't compared

        if i == extnum:
            if EXTENT_MISMATCH(want, have): goto SEARCH_NEXT
        else:
            if ((want - have) & 0x7F) != 0: goto SEARCH_NEXT

    # Match: return position low bits
    lret = dcnt & dskmsk     # 0..3
    if dirloc == 0xFF:
        dirloc = 0
    return FOUND
```

### Delete

```pseudocode
function DELETE_FILE(fcbPtr):
    CHECK_WRITE_ALLOWED()
    SEARCH_FIRST(fcbPtr, length=extnum)

    while not END_OF_DIR():
        CHECK_RO_DIR_ENTRY()                 # error if r/o file
        dirEntry = GETDPTR_ADDR()
        dirEntry[0] = EMPTY (0xE5)          # mark deleted
        SCAN_DISK_MAP_AND_MARK_ALLOC(dirEntry, markBit=0)  # free blocks
        WR_DIR_RECORD()
        SEARCH_NEXT()
```

### Rename / set indicators

Both loop over all matching extents and update directory entries.

```pseudocode
function RENAME_FILE(fcbPtr):
    CHECK_WRITE_ALLOWED()
    SEARCH_FIRST(fcbPtr, length=extnum)

    # Copy fcb[0] (drive/user code byte) into fcb[dskmap] as temp storage
    fcbPtr[dskmap] = fcbPtr[0]

    while not END_OF_DIR():
        CHECK_RO_DIR_ENTRY()
        # Copy name+type from the “new name” half into directory entry
        COPY_DIR(from=fcbPtr, srcOffset=dskmap, length=(extnum - dskmap))
        SEARCH_NEXT()

function SET_FILE_INDICATORS(fcbPtr):
    SEARCH_FIRST(fcbPtr, length=extnum)
    while not END_OF_DIR():
        COPY_DIR(from=fcbPtr, srcOffset=0, length=extnum)
        SEARCH_NEXT()
```

### Open

```pseudocode
function OPEN_FILE(fcbPtr):
    SEARCH_FIRST(fcbPtr, length=namlen)
    if END_OF_DIR():
        return  # lret=0xFF already

    # Copy directory entry -> fcb (first nxtrec bytes)
    dirEntry = GETDPTR_ADDR()
    memcopy(dst=fcbPtr, src=dirEntry, count=nxtrec)

    SET_FWF(fcbPtr)

    # Adjust fcb reccnt based on extent comparison:
    # if user_ext < dir_ext: reccnt=128
    # if equal:             reccnt=dir_reccnt
    # if user_ext > dir_ext:reccnt=0
    user_ext = fcbPtr[extnum]
    dir_ext  = dirEntry[extnum]
    dir_rcnt = dirEntry[reccnt]

    if user_ext == dir_ext: fcbPtr[reccnt] = dir_rcnt
    elif user_ext < dir_ext: fcbPtr[reccnt] = 128
    else: fcbPtr[reccnt] = 0
```

### Close (merge disk maps, update extent/record count, write directory entry)

Close is skipped if:

* disk is read-only (`nowrite` true), or
* file write flag still set (meaning “not written”), or
* file not found in directory

Merge rule:

* any 0 entries in either map get filled from the other
* if after that they mismatch: error (lret=0xFF)
* if ok, update directory extent/reccnt if fcb’s extent is newer/larger
* write directory record back

```pseudocode
function CLOSE_FILE(fcbPtr):
    lret = 0
    dcnt = 0

    if DISK_IS_READONLY(): return
    if (fcbPtr[modnum] & FWF_MASK) != 0: return  # not written, nothing to flush

    SEARCH_FIRST(fcbPtr, length=namlen)
    if END_OF_DIR(): return

    dirEntry = GETDPTR_ADDR()

    if single:
        for i in 0 .. (32-dskmap)-1:
            a = fcbPtr[dskmap + i]
            b = dirEntry[dskmap + i]
            if a == 0: fcbPtr[dskmap + i] = b
            if b == 0: dirEntry[dskmap + i] = a
            if dirEntry[dskmap + i] != fcbPtr[dskmap + i]: return MERGE_ERROR()
    else:
        for i in 0 .. ((32-dskmap)/2)-1:
            a = mem16[fcbPtr + dskmap + 2*i]
            b = mem16[dirEntry + dskmap + 2*i]
            if a == 0: a = b
            if b == 0: b = a
            if a != b: return MERGE_ERROR()
            mem16[fcbPtr + dskmap + 2*i] = a
            mem16[dirEntry + dskmap + 2*i] = b

    # If fcb extent >= directory extent: overwrite dir extent and record count
    if fcbPtr[extnum] >= dirEntry[extnum]:
        dirEntry[extnum] = fcbPtr[extnum]
        dirEntry[reccnt] = fcbPtr[reccnt]

    fcb_copied = true
    SEEK_DIR()
    WR_DIR_RECORD()
```

### Make (create new directory entry)

```pseudocode
function MAKE_FILE(fcbPtr):
    CHECK_WRITE_ALLOWED()

    # Find an empty directory entry (first byte == 0xE5)
    # Temporarily set info to a fake FCB containing only EMPTY
    if not FIND_EMPTY_DIR_ENTRY():
        aret.low = 0xFF
        return

    # Clear remainder of fcb beyond name/type fields
    for i in namlen .. 31:
        fcbPtr[i] = 0
    fcbPtr[ubytes] = 0

    EXTEND_CDRMAX_IF_NEEDED()

    # Copy full fcb to directory entry and set write flag
    COPY_FCB_TO_DIR_ENTRY(fcbPtr)
    SET_FWF(fcbPtr)
```

---

## 10) Sequential & random disk I/O (BDOS 20/21 and 33/34), plus func40

### Sequential read (`diskread` via `seqdiskread`)

```pseudocode
function SEQDISKREAD(fcbPtr):
    seqio = 1
    return DISKREAD(fcbPtr)

function DISKREAD(fcbPtr):
    rmf = true
    GETFCB_FIELDS(fcbPtr)

    # EOF if vrecord >= rcount, unless vrecord==128 and we can open next extent
    if low_byte(vrecord) >= rcount:
        if low_byte(vrecord) != 128:
            lret = 1; return
        OPEN_REEL(fcbPtr)             # next extent
        vrecord = 0
        if lret != 0: return          # can't open -> EOF

    INDEX(fcbPtr)
    if not ALLOCATED():
        lret = 1; return              # reading unwritten data treated as EOF

    ATRAN()
    SEEK_TO_ARECORD()
    RDBUFF()                          # reads to current DMA
    SETFCB_FIELDS(fcbPtr)             # updates nxtrec if sequential
```

### Sequential write (`diskwrite` via `seqdiskwrite`)

```pseudocode
function SEQDISKWRITE(fcbPtr):
    seqio = 1
    return DISKWRITE(fcbPtr)

function DISKWRITE(fcbPtr):
    rmf = false
    CHECK_WRITE_ALLOWED()
    CHECK_RO_FILE(fcbPtr)

    GETFCB_FIELDS(fcbPtr)
    if low_byte(vrecord) > 127:
        lret = 1; return

    INDEX(fcbPtr)

    writeType = 0
    if not ALLOCATED():
        # allocate a new block to cover this vrecord
        slot = DM_POSITION()
        dminx = slot

        prevBlock = PREVIOUS_BLOCK_FOR_FILE(fcbPtr, slot)  # 0 if none
        newBlock = GET_BLOCK_NEAR(prevBlock)
        if newBlock == 0:
            lret = 2; return               # disk full

        STORE_BLOCK_IN_FCB_MAP(fcbPtr, slot, newBlock)
        arecord = newBlock
        writeType = 2                       # “start of new block” for BIOS WRITE

    if lret != 0:
        return

    ATRAN()

    # Special “rand write with fill” path (func40 sets seqio=2):
    # the assembly may zero-fill the whole cluster before writing the record.
    if seqio == 2 and writeType == 2:
        ZERO_FILL_NEW_CLUSTER_BEFORE_WRITE()

    SEEK_TO_ARECORD()
    WRBUFF(writeType)

    # Update rcount if we wrote past end of extent
    if low_byte(vrecord) >= rcount:
        rcount = low_byte(vrecord) + 1
        # Mark that metadata changed so we clear FWF
        metadataChanged = true
    else:
        metadataChanged = (writeType == 2)

    # If metadata changed, clear file-write-flag bit in modnum (“file has been written”)
    if metadataChanged:
        CLEAR_FWF(fcbPtr)

    # If last record in extent and sequential, prepare next extent
    if low_byte(vrecord) == 127 and seqio == 1:
        SETFCB_FIELDS(fcbPtr)        # commit current
        OPEN_REEL(fcbPtr)            # open/create next extent
        if lret == 0:
            vrecord = 0xFF           # assembly uses 0xFF so next +1 wraps to 0

        lret = 0                     # returned value on success

    SETFCB_FIELDS(fcbPtr)
```

### Random seek (`rseek`) + random read/write

Random record (`ranrec`) is a 3-byte value. The seek logic:

* decodes ranrec into: record-within-extent, extent number, module number
* if current fcb extent/module doesn’t match, it closes current, updates fields, opens the correct extent (or creates it if writing), else errors
* on failures, it writes `modnum=1100_0000b` to mark the FCB invalid but “closeable”

```pseudocode
function RSEEK(fcbPtr, isReadModeBool):
    seqio = 0                 # mark random I/O

    (wantRec, wantExt, wantMod) = DECODE_RANREC(fcbPtr)

    # If ranrec high byte != 0 => error 6 (seek past physical EOD)
    if ranrec_high(fcbPtr) != 0:
        return SEEK_ERROR(6)

    fcbPtr[nxtrec] = wantRec

    if (fcbPtr[extnum] != wantExt) or ((fcbPtr[modnum] & 0x7F) != wantMod):
        # Must move to different extent/module
        CLOSE_FILE(fcbPtr)
        if lret == 0xFF: return SEEK_ERROR(3)   # cannot close

        fcbPtr[extnum] = wantExt
        fcbPtr[modnum] = wantMod

        OPEN_FILE(fcbPtr)
        if lret == 0xFF:
            # not found: if read => error 4; if write => create => error 5 if fail
            if isReadModeBool:
                return SEEK_ERROR(4)
            MAKE_FILE(fcbPtr)
            if lret == 0xFF:
                return SEEK_ERROR(5)

    aret.low = 0
    return OK

function RANDISKREAD(fcbPtr):
    if RSEEK(fcbPtr, isReadMode=true) == OK:
        DISKREAD(fcbPtr)

function RANDISKWRITE(fcbPtr):
    if RSEEK(fcbPtr, isReadMode=false) == OK:
        DISKWRITE(fcbPtr)
```

### BDOS function 40 (random write with zero-fill of unallocated block)

This code:

* sets `seqio=2`
* does `rseek` (write mode)
* then `diskwrite`, which uses the “fill new cluster” path

```pseudocode
function BDOS_FUNC40(fcbPtr):
    RESELECT_IF_NEEDED()
    seqio = 2
    if RSEEK(fcbPtr, isReadMode=false) == OK:
        DISKWRITE(fcbPtr)
```

---

## 11) Disk selection + “reselect” logic (why `goback` mutates FCB[0])

CP/M allows an FCB to encode “auto drive select” in fcb[0] (1=A, 2=B, …, 0=current).
This BDOS implements that with:

```pseudocode
function CURSELECT():
    # if linfo already equals curdsk, do nothing
    if linfo == curdsk: return
    curdsk = linfo
    SELECT()            # logs in and maybe initializes disk

function SELECT():
    # call BIOS + load DPB, etc
    ok = SELECTDISK()
    if not ok: GOERR(selerr)

    # if disk not logged in yet: mark logged-in and INITIALIZE()
    if (dlog bit for curdsk) == 0:
        set bit in dlog for curdsk
        INITIALIZE_DISK()

function RESELECT_IF_NEEDED():
    resel = true

    driveCode = (mem8[info + 0] & 0x1F) - 1     # normalized, or 0xFF if none
    linfo = driveCode

    if driveCode <= 29:                         # 30+ => no auto select
        olddsk = curdsk
        fcbdsk = mem8[info + 0]                 # save original drive byte
        mem8[info + 0] &= 0xE0                  # clear low 5 bits (drive select)
        CURSELECT()

    # restore user code bits in fcb[0]
    mem8[info + 0] = (mem8[info + 0] | usrcode)
```

And at end (`goback`), if reselection happened, it cleans up:

```pseudocode
function GOBACK():
    if resel:
        mem8[info + 0] = 0            # fcb(0)=0 (clear drive select)
        if fcbdsk != 0:
            mem8[info + 0] = fcbdsk   # restore original
            linfo = olddsk
            CURSELECT()               # restore prior current disk

    SP = entsp
    HL = aret
    A  = low_byte(aret)
    B  = high_byte(aret)
    return
```

---

## 12) BDOS function handlers 12–37 (high-level mapping)

No tables—just a compact rundown with the key ABI notes:

* **12**: return version
  `aret = 0x0022` (CP/M 2.2).

* **13**: reset disk system
  clears `rodsk` and `dlog`, sets `curdsk=0`, sets `dmaad=0080h`, calls `SETDATA_DMA()`, then `SELECT()` (which initializes disk A: if needed).

* **14**: select disk
  `curselect()` using `linfo` (low byte of DE parameter).

* **15**: open file
  clears fcb module number, reselection (auto-drive/user), then `OPEN_FILE`.

* **16**: close file
  reselection, then `CLOSE_FILE`.

* **17/18**: search first/next
  perform name matching (with special-case “don’t reselect if first char is '?'”), then copy the *directory record* to user DMA (`dir_to_user`).

* **19**: delete file
  reselection, delete, return `dirloc`.

* **20**: seq read
  reselection, sequential read.

* **21**: seq write
  reselection, sequential write.

* **22**: make file
  clear modnum, reselection, make.

* **23**: rename
  reselection + rename + return `dirloc`.

* **24**: return login vector (`dlog`) in HL/B:A.

* **25**: return current disk (`curdsk`) in A.

* **26**: set DMA
  sets `dmaad = DE`, calls BIOS `SETDMA`.

* **27**: return allocation vector address (`alloca`) in HL/B:A.

* **28**: write-protect current disk
  `set_ro()`.

* **29**: return read-only vector (`rodsk`) in HL/B:A.

* **30**: set file indicators
  reselection, update attributes, return `dirloc`.

* **31**: return DPB address (`dpbaddr`) in HL/B:A.

* **32**: get/set user code
  if `linfo == 0xFF` return `usrcode`, else set `usrcode = linfo & 0x1F`.

* **33**: random read
  reselection, random read.

* **34**: random write
  reselection, random write.

* **35**: compute file size
  reselection, `getfilesize` (walks all extents, returns max ranrec in fcb).

* **36**: set random record
  computes ranrec from current FCB fields.

* **37**: “reset drive(s)” using a bitmask passed in `DE`
  clears bits in `dlog` for drives selected by the mask and clears corresponding bits in `rodsk` (also masking by new `dlog`).
  (As decoded from the bitwise/compliment sequence in `func37`.)

* **38, 39**: present but implemented as `func_ret` (no-op return).

* **40**: random write with zero fill of newly allocated block (described above).

---

This walkthrough follows the original control flow closely so the assembly labels map to the pseudocode. It is intended to stand alone as a reference for the BDOS entry stack, ABI, and per-function responsibilities.
