Below is an “easy-to-follow” pseudocode rewrite of **CP/M 2.2 CCP** (Console Command Processor) as represented by your `ccp.asm`, with the **BDOS ABI** and every **BDOS call** clearly annotated (CCP does **not** call BIOS directly; it goes through BDOS at `0005h`).

---

## 1) ABI / interface notes (what CCP expects and what it sets up)

### CCP ↔ BDOS ABI (used everywhere in this file)

**BDOS entry point:** `CALL 0005h`

**Calling convention:**

* `C` = BDOS function number
* `DE` = parameter (pointer or value in `E`)
* Return values:

  * many CCP-used functions return an 8-bit status in `A`
  * some return other values in `A` as well (e.g., current drive/user)
  * CCP frequently uses flags (`Z`) based on `A`

CCP defines helpers that wrap BDOS calls:

* `bdos_inr()` does: `A = BDOS(...); dcnt=A; A++; return A` so that **BDOS returning `0xFF` becomes `0x00`**, making `Z=1` an easy “error/not-found” test.
* `bdos_cond()` does: `A = BDOS(...); set flags from A; return A` so `Z=1` means success when BDOS returns 0.

### CCP low-memory conventions / transient program ABI

* `0004h` (`diska`) is used as a “current disk/user” byte for warm-boot and transient execution. CCP stores `(user<<4) | disk` here in `saveuser()`.
* Default DMA buffer is `0080h` (`buff`). CCP sets DMA there before reads/writes and before launching transient programs.
* Default FCB area for transient programs is `005Ch` (`fcb`). CCP copies a prepared FCB image there before calling the program.

### Command buffer ABI (BDOS function 10 “read buffered line”)

At **CCP base**:

* `ccp+6`: `maxlen = 127`
* `ccp+7`: `comlen` (filled by BDOS 10 or by SUBMIT)
* `ccp+8`: `combuf[128-2]` (command text)

CCP calls **BDOS 10** with `DE=&maxlen`, so BDOS writes:

* `comlen = length`
* `combuf[0..length-1] = chars`

CCP then uppercases the command and appends a `0` byte terminator for scanning.

---

## 2) BDOS functions used by CCP (numbers + meaning + parameter)

(These match your `equ` list.)

* **1** `CONIN` read char → returns `A=char`
* **2** `CONOUT` print char (`E=char`)
* **9** print `$`-terminated string (`DE=ptr`) — CCP doesn’t use `$` strings here; it uses its own 0-terminated `print`
* **10** read buffered line (`DE=&maxlen`) → fills `comlen/combuf`
* **11** console status / break (`A!=0` means char ready), then CCP clears it with function **1**
* **13** initialize disk system
* **14** select disk (`E=disk`, 0=A)
* **15** open file (`DE=FCB`)
* **16** close file (`DE=FCB`)
* **17** search first (`DE=FCB pattern`)
* **18** search next
* **19** delete file (`DE=FCB`)
* **20** sequential read (`DE=FCB`) → `A=0 ok, 1 eof, 2 error` (CCP assumes this pattern)
* **21** sequential write (`DE=FCB`) → `A=0 ok, else error`
* **22** make file (`DE=FCB`)
* **23** rename (`DE=FCB: old in first half, new in second half`)
* **25** return current drive number → `A` (CCP prints prompt from this)
* **26** set DMA address (`DE=addr`)
* **32** get/set user:

  * `E=0xFF` → get user, returns `A=user`
  * `E=user` → set user

No BIOS calls appear in this CCP source.

---

## 3) High-level structure of CCP

```pseudocode
# Globals / storage (names match intent)
byte  maxlen = 127
byte  comlen
byte  combuf[126]        # (effective) command characters
word  comaddr            # scan pointer into combuf (NUL-terminated)
word  staddr             # start pointer for current token (for errors)

byte  cdisk              # current disk (0=A, 1=B, ...)
byte  sdisk              # “selected disk” prefix (0=none, 1=A, 2=B, ...)
byte  submit             # 0=no submit, 0xFF=submit mode enabled

byte  dcnt               # scratch: stores BDOS return A for some calls
byte  bptr               # TYPE command buffer index

# FCBs / file state RAM
byte  comfcb[32]          # command fcb (also used as 2 FCBs via offset 16)
byte  comrec              # current record (FCB+32 style byte)
byte  subfcb[...]         # FCB for $$$.SUB + current record etc
```

Entry points at CCP base:

* `ccp:` → `ccpstart()` (normal start: may execute initial command already in buffer)
* `ccp+3:` → `ccpclear()` (start but disable auto-exec by clearing `comlen`)

---

## 4) BDOS wrapper helpers (as pseudocode)

```pseudocode
# Raw BDOS call ABI:
#   C=function, DE=param; CALL 0005h
function BDOS(funcC, paramDE) -> A:
    ...

# Print one console char (BDOS 2): E=char
function printchar(ch):
    BDOS(2, E=ch)

function crlf():
    printchar('\r')
    printchar('\n')

function print_0term(ptr):         # CCP “print”: 0-terminated, preceded by CRLF
    crlf()
    while *ptr != 0:
        printchar(*ptr)
        ptr++

function initialize_disk_system() -> A:
    # BDOS 13
    return BDOS(13, DE=ignored)

function select_disk(disk0_based): # BDOS 14: E=disk
    BDOS(14, E=disk0_based)

# “bdos_inr”: make 0xFF become 0 (easy Z-test), and save original in dcnt
function bdos_inr(funcC, fcbPtrDE) -> bool_ok:
    A = BDOS(funcC, DE=fcbPtrDE)
    dcnt = A
    A = (A + 1) & 0xFF
    return (A != 0)               # false indicates original was 0xFF

function open_file(fcbPtr)  -> bool_ok: return bdos_inr(15, fcbPtr)
function close_file(fcbPtr) -> bool_ok: return bdos_inr(16, fcbPtr)
function search_first(fcbPtr)-> bool_ok: return bdos_inr(17, fcbPtr)
function search_next(fcbPtr) -> bool_ok: return bdos_inr(18, fcbPtr)
function make_file(fcbPtr)  -> bool_ok: return bdos_inr(22, fcbPtr)

function delete_file(fcbPtr):
    BDOS(19, DE=fcbPtr)

# diskread/diskwrite return A status; Z indicates success when A==0
function diskread(fcbPtr) -> A_status:
    return BDOS(20, DE=fcbPtr)

function diskwrite(fcbPtr) -> A_status:
    return BDOS(21, DE=fcbPtr)

function get_current_drive() -> byte:
    return BDOS(25, DE=ignored)   # A holds 0-based drive

function set_dma(addr):
    BDOS(26, DE=addr)

function break_key_pressed() -> bool:
    # BDOS 11: status. If nonzero, clear by reading BDOS 1.
    A = BDOS(11, DE=ignored)
    if A == 0: return false
    BDOS(1, DE=ignored)           # consume pending char
    return true

function get_user() -> byte:
    return BDOS(32, E=0xFF)

function set_user(user):
    BDOS(32, E=user)
```

---

## 5) Command input: console vs SUBMIT (`readcom` + `del_sub`)

### Submit file logic in plain terms

If `submit != 0`, CCP tries to read commands from `A:$$$.SUB`, **from the last record backwards**, copying one record into the command buffer each time and decrementing the file’s record count to “pop” it. If anything goes wrong or the user presses a key, it deletes the submit file and returns to interactive mode.

```pseudocode
function del_sub():                       # delete $$$.SUB and clear submit flag
    if submit == 0: return
    submit = 0
    select_disk(0)                        # drive A:
    delete_file(&subfcb)
    select_disk(cdisk)                    # back to original

function readcom():
    if submit != 0:
        # switch to A: if needed and try open $$$.SUB
        if cdisk != 0: select_disk(0)
        if not open_file(&subfcb):
            goto NO_SUBMIT

        # read last record: subcr = subrc - 1, then BDOS read
        subcr = subrc - 1
        status = diskread(&subfcb)        # reads into DMA buffer 0x80 by default
        if status != 0:
            goto NO_SUBMIT

        # copy 128 bytes from buff(0x80) into comlen+combuf area
        memcopy(dst=&comlen, src=0x0080, count=128)

        # “truncate” submit file by clearing fwflag and decrementing record count, then close
        submod = 0
        subrc  = subrc - 1
        if not close_file(&subfcb):
            goto NO_SUBMIT

        # back to original disk
        if cdisk != 0: select_disk(cdisk)

        # echo submit line to console up to NUL
        print_raw_until_nul(&combuf[0])   # (their prin0 without CRLF)

        # allow abort: if break key, delete submit and restart
        if break_key_pressed():
            del_sub()
            goto CCP_MAIN_LOOP

        goto POST_READ

    NO_SUBMIT:
        del_sub()
        # then fall through to normal console read

    # normal console read via BDOS 10
    saveuser()                            # store user/disk at 0004h in case ^C/warmboot
    BDOS(10, DE=&maxlen)                  # fills comlen/combuf
    setdiska()                            # restore low-memory disk value after read

POST_READ:
    # uppercase translation and NUL-termination
    for i in 0 .. comlen-1:
        combuf[i] = to_upper(combuf[i])
    combuf[comlen] = 0                    # sentinel for scanner

    comaddr = &combuf[0]
```

(Their `prin0` prints from a pointer until a `0` byte.)

---

## 6) Token scanning and FCB filling (`fillfcb`)

Key idea: `comaddr` points into the NUL-terminated command line. `fillfcb(offset)` parses the next token into `comfcb[offset..]`, handling:

* optional drive prefix `A:`
* filename `8` chars + extension `3` chars
* `*` expands to repeated `?` wildcards
* sets `sdisk` if a drive prefix is present
* advances `comaddr` past this token
* returns number of `?` wildcards in name+type (Z flag in assembly corresponds to “no ?”)

Delimiter set includes: blank, `=`, left-arrow (`_` in your listing via `la=0x5F`), `.`, `:`, `;`, `<`, `>`, NUL.

```pseudocode
function is_delim(ch) -> bool:
    if ch == 0: return true
    if ch < ' ': command_error()
    if ch == ' ': return true
    if ch in ['=', LA, '.', ':', ';', '<', '>']: return true
    return false

function skip_blanks(ptr) -> ptr:
    while *ptr == ' ': ptr++
    return ptr

function fillfcb(offset):   # offset is 0 (first) or 16 (second)
    f = &comfcb[offset]
    sdisk = 0

    p = skip_blanks(comaddr)
    staddr = p                  # remember for comerr printing

    # drive prefix?
    if *p != 0:
        maybe = *p
        if 'A' <= maybe <= 'P' and p[1] == ':':
            sdisk = (maybe - 'A' + 1)      # 1=A, 2=B, ...
            f[0] = sdisk                   # (drive code in FCB byte 0)
            p += 2
        else:
            # no explicit drive; CCP later often forces f[0]=0 for “use current”
            f[0] = cdisk   # (as in assembly; many callers ignore/overwrite this)
    else:
        f[0] = cdisk

    # filename (8)
    for i in 0..7:
        ch = *p
        if is_delim(ch): goto PAD_NAME
        if ch == '*':
            f[1+i] = '?'
            # NOTE: do NOT advance p -> repeats '?' for remaining slots
        else:
            f[1+i] = ch
            p++
    # if name longer than 8, skip until delimiter
    while not is_delim(*p): p++
    goto TYPE_FIELD

PAD_NAME:
    for j from current_i..7:
        f[1+j] = ' '

TYPE_FIELD:
    # type (3), only if a '.' is present
    if *p == '.':
        p++
        for i in 0..2:
            ch = *p
            if is_delim(ch): break
            if ch == '*':
                f[9+i] = '?'
            else:
                f[9+i] = ch
                p++
        while not is_delim(*p): p++    # truncate remainder
    else:
        for i in 0..2: f[9+i] = ' '

    # clear extent/s1/s2 (3 bytes following type)
    f[12] = 0; f[13] = 0; f[14] = 0

    # advance comaddr
    comaddr = p

    # count '?' in 11 bytes (name+type)
    qcount = 0
    for i in 1..11:
        if f[i] == '?': qcount++
    return qcount
```

---

## 7) Intrinsic dispatch (`DIR`, `ERA`, `TYPE`, `SAVE`, `REN`, `USER`)

CCP recognizes intrinsics by comparing first 4 chars of `comfcb+1` against:

```text
0: "DIR "
1: "ERA "
2: "TYPE"
3: "SAVE"
4: "REN "
5: "USER"
else: userfunc (external .COM)
```

Pseudocode:

```pseudocode
function intrinsic_index() -> int:
    name4 = comfcb[1..4]
    for i, key in enumerate(["DIR ", "ERA ", "TYPE", "SAVE", "REN ", "USER"]):
        if name4 == key and comfcb[5] == ' ':
            return i
    return 6   # userfunc
```

---

## 8) CCP start / main loop / command execution

```pseudocode
function ccpclear():
    comlen = 0
    ccpstart()

function ccpstart():
    SP = &stack

    # Boot passes C = (user<<4) | disk (disk is low nibble, 0=A)
    boot_user = (C >> 4) & 0x0F
    set_user(boot_user)

    # BDOS init; returns 0xFF if a '$' file exists (used as “submit present” hint)
    submit = initialize_disk_system()

    cdisk = C & 0x0F
    select_disk(cdisk)

    # If comlen nonzero, execute initial command already in buffer; else prompt.
    if comlen != 0:
        goto EXECUTE_CURRENT_BUFFER
    else:
        goto CCP_MAIN_LOOP

CCP_MAIN_LOOP:
    SP = &stack
    crlf()
    drive = get_current_drive()          # BDOS 25, 0-based
    printchar('A' + drive)
    printchar('>')
    readcom()                            # fills combuf, sets comaddr

EXECUTE_CURRENT_BUFFER:
    set_dma(0x0080)                      # default DMA is buff
    cdisk = get_current_drive()

    # parse command word into comfcb
    q = fillfcb(offset=0)
    if q != 0: command_error()           # command itself cannot be ambiguous (“???”)

    # If command had an explicit drive prefix (sdisk!=0), treat it as external/disk switch
    if sdisk != 0:
        userfunc()
        return

    idx = intrinsic_index()
    dispatch_intrinsic(idx)
```

`dispatch_intrinsic` maps 0..5 to handlers, 6 to `userfunc`.

---

## 9) Error reporting (`comerr`) – “print offending token + ?”

This prints from `staddr` until blank or NUL, then prints `?`, CRLF, deletes submit file, and restarts CCP.

```pseudocode
function command_error():
    crlf()
    p = staddr
    while *p != 0 and *p != ' ':
        printchar(*p)
        p++
    printchar('?')
    crlf()
    del_sub()
    goto CCP_MAIN_LOOP
```

---

## 10) Intrinsics (easy pseudocode)

### (0) DIR — directory listing

* Parses an optional filespec after `DIR`
* If no filespec, uses `????????.???`
* Calls BDOS search first/next on `comfcb`
* Uses `dcnt` (saved BDOS return) to locate the directory entry within DMA buffer `buff`:

  * `dirPos = dcnt & 3`
  * `entryPtr = buff + dirPos*32`
* Skips entries marked “system” (sys bit in type flags)

```pseudocode
function DIR():
    fillfcb(0)                 # parse optional filespec
    setdisk_for_command()

    if comfcb[1] == ' ':
        for i in 1..11: comfcb[i] = '?'

    colCount = 0
    if not search_first(&comfcb):
        print_0term("NO FILE\0")
        goto RETCOM

    while true:
        dirPos   = dcnt & 3                    # 0..3 saved from BDOS
        entryPtr = 0x0080 + dirPos*32          # DMA buffer

        if is_system_file(entryPtr):           # checks sysfile flag bit
            goto NEXT

        if (colCount % 4) == 0:
            crlf()
            printchar('A' + get_current_drive())
            printchar(':')
        else:
            printchar(' ')
            printchar(':')

        printchar(' ')
        print_filename_8_3_from_dir_entry(entryPtr)

        colCount++

    NEXT:
        if break_key_pressed(): break
        if not search_next(&comfcb): break

    RETCOM:
        resetdisk_after_command()
        endcom_check_garbage()
```

### (1) ERA — erase

If filespec is all wildcards (`qcount == 11`), asks `ALL (Y/N)?` and requires `Y`.

```pseudocode
function ERA():
    q = fillfcb(0)
    if q == 11:
        print_0term("ALL (Y/N)?\0")
        readcom()
        if comlen != 1 or combuf[0] != 'Y':
            goto CCP_MAIN_LOOP
        # consume remaining input by adjusting comaddr
        comaddr = &combuf[1]

    setdisk_for_command()
    delete_file(&comfcb)

    # BDOS delete returns 0xFF if not found; CCP commonly does inr-test
    if last_BDOS_A_was_0xFF():
        print_0term("NO FILE\0")

    goto RETCOM
```

### (2) TYPE — print file to console until ^Z or break

Reads 128-byte records via BDOS 20, prints until `0x1A`, stops on break.

```pseudocode
function TYPE():
    if fillfcb(0) != 0: command_error()   # no '?' allowed
    setdisk_for_command()

    if not open_file(&comfcb):
        resetdisk_after_command()
        command_error()

    crlf()
    bptr = 255

    while true:
        if bptr >= 127:
            status = diskread(&comfcb)    # fills DMA=0x80
            if status != 0:
                if status == 1: break     # EOF ok
                print_0term("READ ERROR\0")
                break
            bptr = 0
        else:
            bptr++

        ch = mem8[0x0080 + bptr]
        if ch == 0x1A: break              # ^Z
        printchar(ch)
        if break_key_pressed(): break

    goto RETCOM
```

### (3) SAVE — “SAVE n filename”

* `n` is pages (256 bytes). CCP converts to sectors (128 bytes) by `sectors = pages*2`.
* Writes sequential 128-byte chunks starting at `0x100` to the file.

```pseudocode
function SAVE():
    pages = getnumber()                   # parse decimal from line
    sectors = pages * 2

    if fillfcb(0) != 0: command_error()
    setdisk_for_command()

    delete_file(&comfcb)                  # remove existing
    if not make_file(&comfcb):
        print_0term("NO SPACE\0")
        goto RETCOM

    comrec = 0
    addr = 0x0100

    for i in 0 .. sectors-1:
        set_dma(addr)
        status = diskwrite(&comfcb)
        if status != 0:
            print_0term("NO SPACE\0")
            goto RETSAVE
        addr += 128

    # close (directory update)
    close_file(&comfcb)                   # CCP tries to detect failure, but mostly errors are caught above

RETSAVE:
    set_dma(0x0080)
    goto RETCOM
```

### (4) REN — rename: `REN newname=oldname`

* New name parsed first into comfcb, checked not already existing.
* Then copied into comfcb+16 (second half).
* Old name parsed into comfcb (first half).
* BDOS 23 called with `DE=&comfcb`.

```pseudocode
function REN():
    if fillfcb(0) != 0: command_error()
    newDrive = sdisk
    setdisk_for_command()

    if search_first(&comfcb):
        print_0term("FILE EXISTS\0")
        goto RETCOM

    # move first 16 bytes to second half (new name -> comfcb+16)
    memcopy(dst=&comfcb[16], src=&comfcb[0], count=16)

    # require '=' or left-arrow delimiter
    p = skip_blanks(comaddr)
    if *p not in ['=', LA]:
        resetdisk_after_command()
        command_error()
    comaddr = p + 1

    if fillfcb(0) != 0:
        resetdisk_after_command()
        command_error()

    # drive conflict check (if old drive specified, must match newDrive)
    if sdisk != 0 and newDrive != 0 and sdisk != newDrive:
        resetdisk_after_command()
        command_error()
    if newDrive != 0:
        sdisk = newDrive                  # keep consistent

    # verify old file exists
    comfcb[0] = 0                         # search on current selected disk
    if not search_first(&comfcb):
        print_0term("NO FILE\0")
        goto RETCOM

    BDOS(23, DE=&comfcb)                  # rename
    goto RETCOM
```

### (5) USER — set user number 0..15

```pseudocode
function USER():
    n = getnumber()
    if n >= 16: command_error()
    if comfcb[1] == ' ': command_error()  # no digits typed
    set_user(n)
    goto ENDCOM
```

---

## 11) External command execution (`userfunc`) — load and run `COMMAND.COM`

This is the heart of CCP for transient programs.

Rules implemented:

* If command name is blank **but** a drive prefix exists, treat as “disk switch” (`B:`).
* Otherwise, require that the command has no explicit filetype; force `.COM`.
* Load the `.COM` file into memory starting at `0x100`, reading 128-byte records until EOF.
* Abort if load would overwrite CCP (`addr >= tranm`).
* Before calling program:

  * build default FCB(s) at `005Ch` from remaining command tail (two parses: offset 0 and 16)
  * build command tail at `0080h`: length byte + characters after first blank
  * set DMA to `0080h`
  * store user/disk at `0004h` (`saveuser`)
  * `CALL 0100h`

```pseudocode
function userfunc():
    # optional serialization check compares CCP serial bytes to BDOS serial bytes;
    # on failure it patches CCP start with DI;HLT and jumps there.

    if comfcb[1] == ' ':
        # No command word. If there was a drive prefix, just switch disks.
        if sdisk == 0:
            goto ENDCOM
        cdisk = sdisk - 1            # convert 1-based to 0-based
        setdiska()
        select_disk(cdisk)
        goto ENDCOM

    # command exists; require blank type field, then force type = "COM"
    if comfcb[9] != ' ': command_error()

    setdisk_for_command()            # may select sdisk drive for loading the .COM
    comfcb[9..11] = ['C','O','M']

    if not open_file(&comfcb):
        resetdisk_after_command()
        command_error()

    # load into memory 0x100 using DMA
    addr = 0x0100
    while true:
        set_dma(addr)
        status = diskread(&comfcb)
        if status == 0:
            addr += 128
            if addr >= tranm:        # would overwrite CCP/BDOS area
                print_0term("BAD LOAD\0")
                goto RETCOM
            continue
        if status == 1:              # EOF
            break
        print_0term("BAD LOAD\0")
        goto RETCOM

    resetdisk_after_command()        # restore original disk before argument parsing

    # Build FCB(s) for transient program at 005Ch
    fillfcb(0)                       # first argument
    comfcb[0] = sdisk                # override: 0 if none, else 1=A etc
    fillfcb(16)                      # second argument
    comfcb[16] = sdisk               # store drive for second half too

    comrec = 0
    memcopy(dst=0x005C, src=&comfcb[0], count=33)  # includes “current record” byte

    # Build command tail at 0080h: everything from first blank onward
    p = &combuf[0]
    while *p != 0 and *p != ' ': p++
    tailLen = 0
    dst = 0x0080 + 1
    while true:
        *dst = *p
        if *p == 0: break
        tailLen++
        p++; dst++
    mem8[0x0080] = tailLen

    crlf()

    set_dma(0x0080)
    saveuser()
    call 0x0100                      # execute transient

    # when it returns:
    SP = &stack
    setdiska()
    select_disk(cdisk)
    goto CCP_MAIN_LOOP
```

Disk-switch helpers used above:

```pseudocode
function setdisk_for_command():
    comfcb[0] = 0                    # “use current” for BDOS ops
    if sdisk == 0: return
    target = sdisk - 1               # 0-based
    if target == cdisk: return
    select_disk(target)

function resetdisk_after_command():
    if sdisk == 0: return
    target = sdisk - 1
    if target == cdisk: return
    select_disk(cdisk)
```

---

## 12) End-of-command garbage check (`endcom`)

After any command, CCP ensures nothing else remains on the line (no stray tokens), otherwise `comerr`.

```pseudocode
function endcom_check_garbage():
    fillfcb(0)
    # valid end means: blank “name” and no drive prefix
    if comfcb[1] != ' ' or sdisk != 0:
        command_error()
    goto CCP_MAIN_LOOP
```

---

If you paste the rest of `ccp.asm` (or confirm there isn’t more), I can also produce a second version where the pseudocode is organized as a neat “CCP state machine” (PROMPT → READ → PARSE → DISPATCH → DONE), but the logic above already matches each labeled routine one-for-one.

