Here’s the **second version** of the CCP rewrite, organized as a **single explicit state machine** (PROMPT → READ → PARSE → DISPATCH → CLEANUP), with **BDOS ABI annotations** embedded at each call site.

---

## CCP as a state machine (one main loop, explicit states)

### BDOS ABI (CCP-only reminder)

* **CALL 0005h** (BDOS)
* **C = function number**
* **DE = parameter** (pointer or value in `E`)
* **Returns in A** (often: `A=0 OK`, `A=0xFF not found`, or `A=1 EOF`, etc., depending on function)

CCP never calls BIOS directly.

---

## Core CCP memory model

```pseudocode
# CCP-resident command buffer (at CCP base)
byte maxlen = 127
byte comlen = 0
byte combuf[128-2]          # command characters (later NUL-terminated by CCP)
word comaddr                # scanner cursor into combuf (points at next token)
word staddr                 # start of current token (for comerr printing)

# Disk/user/session
byte cdisk                  # current disk: 0=A,1=B,...
byte sdisk                  # disk prefix for this command: 0=none, 1=A,2=B,...
byte submit                 # 0=no submit, 0xFF=submit mode

# Scratch saved from some BDOS calls
byte dcnt                   # stores BDOS “directory position” / code (via wrapper)

# FCB working area
byte comfcb[32]             # active command FCB (first half)
byte comrec                 # “current record” byte (used during loads/saves)
# submit FCB ($$$.SUB) and fields
byte subfcb[...]            # $$$.SUB FCB (on A:)
byte submod
byte subrc
byte subcr

# Default addresses
const word BDOS = 0x0005
const word DISKA = 0x0004   # “user/disk” byte in low memory
const word BUFF = 0x0080    # default DMA / command tail buffer
const word FCB_DEFAULT = 0x005C
const word TPA_BASE = 0x0100  # transient load base
const word CCP_LIMIT = tranm  # “top” address for loading (from asm)
```

---

## BDOS wrapper API (with function numbers)

```pseudocode
function BDOS_CALL(C, DE) -> byte A

function BDOS_CONOUT(ch):
    # BDOS 2: E=char
    BDOS_CALL(C=2, DE=(0<<8 | ch))

function BDOS_CONSTAT() -> byte:
    # BDOS 11
    return BDOS_CALL(C=11, DE=0)

function BDOS_CONIN() -> byte:
    # BDOS 1
    return BDOS_CALL(C=1, DE=0)

function BDOS_READLINE(addrMaxLen):
    # BDOS 10: DE = &maxlen (buffer descriptor)
    BDOS_CALL(C=10, DE=addrMaxLen)

function BDOS_INIT() -> byte:
    # BDOS 13: returns 0xFF if '$' file “present” in DRI CCP convention
    return BDOS_CALL(C=13, DE=0)

function BDOS_SELDSK(disk0based):
    # BDOS 14: E=disk
    BDOS_CALL(C=14, DE=(0<<8 | disk0based))

function BDOS_CURDSK() -> byte:
    # BDOS 25: returns 0-based current disk in A
    return BDOS_CALL(C=25, DE=0)

function BDOS_SETDMA(addr):
    # BDOS 26: DE=addr
    BDOS_CALL(C=26, DE=addr)

function BDOS_USER_GET() -> byte:
    # BDOS 32: E=0xFF => get
    return BDOS_CALL(C=32, DE=0x00FF)

function BDOS_USER_SET(user):
    # BDOS 32: E=user => set
    BDOS_CALL(C=32, DE=(0<<8 | user))

# Helper: “bdos_inr” pattern used in ccp.asm:
# store A in dcnt, and return (A+1)!=0 so that A=0xFF becomes false.
function BDOS_INR(C, fcbPtr) -> bool ok:
    A = BDOS_CALL(C=C, DE=fcbPtr)
    dcnt = A
    return ((A + 1) & 0xFF) != 0

function BDOS_OPEN(fcbPtr)  -> bool ok: return BDOS_INR(15, fcbPtr)
function BDOS_CLOSE(fcbPtr) -> bool ok: return BDOS_INR(16, fcbPtr)
function BDOS_SFIRST(fcbPtr)-> bool ok: return BDOS_INR(17, fcbPtr)
function BDOS_SNEXT(fcbPtr) -> bool ok: return BDOS_INR(18, fcbPtr)
function BDOS_MAKE(fcbPtr)  -> bool ok: return BDOS_INR(22, fcbPtr)

function BDOS_DELETE(fcbPtr):
    # BDOS 19
    BDOS_CALL(C=19, DE=fcbPtr)

function BDOS_READ(fcbPtr) -> byte:
    # BDOS 20: A=0 OK, 1 EOF, 2+ error (as used by CCP)
    return BDOS_CALL(C=20, DE=fcbPtr)

function BDOS_WRITE(fcbPtr) -> byte:
    # BDOS 21
    return BDOS_CALL(C=21, DE=fcbPtr)

function BDOS_RENAME(fcbPtr):
    # BDOS 23
    BDOS_CALL(C=23, DE=fcbPtr)
```

---

## Console helpers (CCP’s own)

```pseudocode
function CRLF():
    BDOS_CONOUT('\r')
    BDOS_CONOUT('\n')

function PRINT0Z(ptr0term):
    # CCP’s internal messages are 0-terminated, and it prints CRLF first
    CRLF()
    while *ptr0term != 0:
        BDOS_CONOUT(*ptr0term)
        ptr0term++

function BREAK_KEY_PRESSED() -> bool:
    # BDOS 11 status; if nonzero, read one char (BDOS 1) to clear it
    if BDOS_CONSTAT() == 0: return false
    BDOS_CONIN()
    return true

function TO_UPPER(ch) -> ch:
    if 'a' <= ch <= 'z': return ch & 0x5F
    return ch

function SAVEUSER_DISKA():
    # store (user<<4)|(disk) into low memory 0004h so transients / ^C can recover it
    user = BDOS_USER_GET()
    mem[DISKA] = ((user & 0x0F) << 4) | (cdisk & 0x0F)

function SETDISKA_FROM_CDISK():
    mem[DISKA] = (mem[DISKA] & 0xF0) | (cdisk & 0x0F)
```

---

## Tokenization / FCB fill (single “scanner” interface)

Think of the command line as a stream:

* `comaddr` points at next unread character in combuf (NUL-terminated by CCP)
* `fillfcb(offset)` consumes one token (filespec), writes into `comfcb[offset]...`, updates `sdisk`, and advances `comaddr`
* it returns `qcount` (# of `?` wildcards) — CCP uses “qcount==0” to mean **unambiguous**

(Exact delimiter set matches asm: blank, NUL, `=`, left-arrow, `.`, `:`, `;`, `<`, `>`; control chars error.)

```pseudocode
function COMMAND_ERROR():
    CRLF()
    p = staddr
    while *p != 0 and *p != ' ':
        BDOS_CONOUT(*p)
        p++
    BDOS_CONOUT('?')
    CRLF()
    DEL_SUBMIT_FILE()
    goto STATE_PROMPT

function IS_DELIM(ch) -> bool:
    if ch == 0: return true
    if ch < ' ': COMMAND_ERROR()
    if ch == ' ': return true
    if ch in ['=', LA, '.', ':', ';', '<', '>']: return true
    return false

function SKIP_BLANKS(p) -> p:
    while *p == ' ': p++
    return p

function FILLFCB(offset) -> int qcount:
    f = &comfcb[offset]
    sdisk = 0

    p = SKIP_BLANKS(comaddr)
    staddr = p

    # optional drive prefix A:
    if *p != 0 and ('A' <= *p <= 'P') and p[1] == ':':
        sdisk = (*p - 'A' + 1)      # 1=A,2=B,...
        f[0] = sdisk
        p += 2
    else:
        # default: “current” (CCP later may overwrite comfcb[0]=0 for BDOS calls)
        f[0] = cdisk

    # filename 8
    i = 0
    while i < 8 and not IS_DELIM(*p):
        if *p == '*':
            f[1+i] = '?'
            # do not advance p: repeats '?' in remaining slots (matching asm behavior)
        else:
            f[1+i] = *p
            p++
        i++
    while i < 8:
        f[1+i] = ' '
        i++

    # if name longer than 8, truncate until delimiter
    while not IS_DELIM(*p):
        p++

    # type 3 if '.' present
    if *p == '.':
        p++
        j = 0
        while j < 3 and not IS_DELIM(*p):
            if *p == '*':
                f[9+j] = '?'
            else:
                f[9+j] = *p
                p++
            j++
        while j < 3:
            f[9+j] = ' '
            j++
        while not IS_DELIM(*p):
            p++
    else:
        f[9] = ' '; f[10] = ' '; f[11] = ' '

    # clear EX/S1/S2 (3 bytes)
    f[12] = 0; f[13] = 0; f[14] = 0

    comaddr = p

    # count '?' in 11 bytes name+type
    qcount = 0
    for k in 1..11:
        if f[k] == '?': qcount++
    return qcount
```

---

## Disk switching policy (explicit states)

CCP uses:

* `sdisk` = 0 means “no prefix”, stay on `cdisk`
* `sdisk` != 0 means a command-prefixed disk was requested for *this* operation

```pseudocode
function SETDISK_FOR_COMMAND():
    comfcb[0] = 0                 # “use current” for BDOS file operations
    if sdisk == 0: return
    wanted = sdisk - 1            # to 0-based
    if wanted == cdisk: return
    BDOS_SELDSK(wanted)

function RESETDISK_AFTER_COMMAND():
    if sdisk == 0: return
    wanted = sdisk - 1
    if wanted == cdisk: return
    BDOS_SELDSK(cdisk)
```

---

## Submit file handling (state-friendly)

```pseudocode
function DEL_SUBMIT_FILE():
    if submit == 0: return
    submit = 0
    BDOS_SELDSK(0)                 # A:
    BDOS_DELETE(&subfcb)
    BDOS_SELDSK(cdisk)

function READ_COMMAND_INTO_BUFFER():
    if submit != 0:
        # Attempt to pop one record from A:$$$.SUB
        if cdisk != 0: BDOS_SELDSK(0)
        if not BDOS_OPEN(&subfcb):
            goto FALLBACK_CONSOLE

        subcr = subrc - 1
        if BDOS_READ(&subfcb) != 0:
            goto FALLBACK_CONSOLE

        memcopy(dst=&comlen, src=BUFF, count=128)

        submod = 0
        subrc = subrc - 1
        if not BDOS_CLOSE(&subfcb):
            goto FALLBACK_CONSOLE

        if cdisk != 0: BDOS_SELDSK(cdisk)

        # echo submit line until NUL
        p = &combuf[0]
        while *p != 0:
            BDOS_CONOUT(*p)
            p++

        if BREAK_KEY_PRESSED():
            DEL_SUBMIT_FILE()
            goto STATE_PROMPT

        goto AFTER_READ

    FALLBACK_CONSOLE:
        DEL_SUBMIT_FILE()
        # Interactive console read (BDOS 10)
        SAVEUSER_DISKA()
        BDOS_READLINE(&maxlen)           # BDOS 10
        SETDISKA_FROM_CDISK()

    AFTER_READ:
        # uppercase and NUL-terminate
        for i in 0 .. comlen-1:
            combuf[i] = TO_UPPER(combuf[i])
        combuf[comlen] = 0
        comaddr = &combuf[0]
```

---

## Intrinsic recognition (no tables; just a clean matcher)

```pseudocode
function MATCH4(ptr, s4) -> bool:
    return ptr[0]==s4[0] and ptr[1]==s4[1] and ptr[2]==s4[2] and ptr[3]==s4[3]

function INTRINSIC_KIND() -> enum:
    name = &comfcb[1]     # first 4 chars of command in FCB
    if MATCH4(name,"DIR "):  return DIR
    if MATCH4(name,"ERA "):  return ERA
    if MATCH4(name,"TYPE"):  return TYPE
    if MATCH4(name,"SAVE"):  return SAVE
    if MATCH4(name,"REN "):  return REN
    if MATCH4(name,"USER"):  return USER
    return TRANSIENT
```

---

## The CCP state machine

### State enumeration

```pseudocode
STATE_BOOT
STATE_PROMPT
STATE_READ
STATE_PARSE
STATE_DISPATCH
STATE_DONE
STATE_ERROR
```

### Boot/start logic

```pseudocode
state = STATE_BOOT

STATE_BOOT:
    SP = &stack

    # Boot passes C = (user<<4)|(disk)
    boot_user = (C >> 4) & 0x0F
    BDOS_USER_SET(boot_user)           # BDOS 32

    submit = BDOS_INIT()               # BDOS 13 (CCP uses A as “submit hint”)

    cdisk = C & 0x0F
    BDOS_SELDSK(cdisk)                 # BDOS 14

    if comlen != 0:
        state = STATE_PARSE            # execute preloaded command line
    else:
        state = STATE_PROMPT
```

### Prompt

```pseudocode
STATE_PROMPT:
    SP = &stack
    CRLF()
    d = BDOS_CURDSK()                  # BDOS 25
    BDOS_CONOUT('A' + d)
    BDOS_CONOUT('>')
    state = STATE_READ
```

### Read

```pseudocode
STATE_READ:
    BDOS_SETDMA(BUFF)                  # BDOS 26 (default DMA for searches/reads)
    cdisk = BDOS_CURDSK()              # remember “home” disk for reset
    READ_COMMAND_INTO_BUFFER()
    state = STATE_PARSE
```

### Parse + dispatch decision

```pseudocode
STATE_PARSE:
    BDOS_SETDMA(BUFF)                  # keep DMA sane

    q = FILLFCB(offset=0)              # parse command word into comfcb
    if q != 0:
        state = STATE_ERROR            # ambiguous command name not allowed
        break

    if sdisk != 0:
        # command had a drive prefix like "B:FOO" or "B:"
        state = STATE_DISPATCH         # will go through TRANSIENT handler which also does disk-switch-only
        break

    kind = INTRINSIC_KIND()
    state = STATE_DISPATCH
```

### Dispatch

```pseudocode
STATE_DISPATCH:
    if kind == DIR:  DO_DIR()
    elif kind == ERA: DO_ERA()
    elif kind == TYPE: DO_TYPE()
    elif kind == SAVE: DO_SAVE()
    elif kind == REN: DO_REN()
    elif kind == USER: DO_USER()
    else: DO_TRANSIENT_OR_DISKSWITCH()

    state = STATE_DONE
```

### Done (cleanup + “garbage” check)

```pseudocode
STATE_DONE:
    RESETDISK_AFTER_COMMAND()

    # “endcom”: nothing left on line except blanks/NUL; also no stray drive prefix
    _ = FILLFCB(offset=0)
    if comfcb[1] != ' ' or sdisk != 0:
        state = STATE_ERROR
    else:
        state = STATE_PROMPT
```

### Error

```pseudocode
STATE_ERROR:
    COMMAND_ERROR()
    # COMMAND_ERROR jumps to STATE_PROMPT
```

---

## Intrinsic handlers (state-oriented summaries)

### DIR

```pseudocode
function DO_DIR():
    _ = FILLFCB(0)                  # optional filespec
    SETDISK_FOR_COMMAND()

    if comfcb[1] == ' ':
        for i in 1..11: comfcb[i] = '?'

    if not BDOS_SFIRST(&comfcb):
        PRINT0Z("NO FILE\0")
        return

    shown = 0
    while true:
        # directory entry lives in DMA buffer; dcnt encodes which of 4 entries
        entryIndex = dcnt & 3
        entryPtr = BUFF + entryIndex*32

        if not IS_SYSTEM_FILE(entryPtr):
            if (shown % 4) == 0:
                CRLF()
                BDOS_CONOUT('A' + BDOS_CURDSK())
                BDOS_CONOUT(':')
            else:
                BDOS_CONOUT(' ')
                BDOS_CONOUT(':')

            BDOS_CONOUT(' ')
            PRINT_8DOT3_FROM_DIR(entryPtr)
            shown++

        if BREAK_KEY_PRESSED(): return
        if not BDOS_SNEXT(&comfcb): return
```

### ERA

```pseudocode
function DO_ERA():
    q = FILLFCB(0)

    if q == 11:                      # ????????.???  -> “ALL?”
        PRINT0Z("ALL (Y/N)?\0")
        READ_COMMAND_INTO_BUFFER()
        if comlen != 1 or combuf[0] != 'Y': return
        comaddr = &combuf[1]         # consume

    SETDISK_FOR_COMMAND()
    BDOS_DELETE(&comfcb)
    # CCP treats 0xFF-return as “not found” in callers; message if so:
    if last_delete_returned_0xFF(): PRINT0Z("NO FILE\0")
```

### TYPE

```pseudocode
function DO_TYPE():
    if FILLFCB(0) != 0: COMMAND_ERROR()
    SETDISK_FOR_COMMAND()

    if not BDOS_OPEN(&comfcb):
        RESETDISK_AFTER_COMMAND()
        COMMAND_ERROR()

    CRLF()
    bptr = 255

    while true:
        if bptr >= 127:
            status = BDOS_READ(&comfcb)     # BDOS 20 -> DMA=BUFF
            if status == 1: return          # EOF
            if status != 0:
                PRINT0Z("READ ERROR\0")
                return
            bptr = 0
        else:
            bptr++

        ch = mem[BUFF + bptr]
        if ch == 0x1A: return               # ^Z
        BDOS_CONOUT(ch)
        if BREAK_KEY_PRESSED(): return
```

### SAVE

```pseudocode
function DO_SAVE():
    pages = GETNUMBER_FROM_LINE()           # via FILLFCB(0) digits parse
    sectors = pages * 2

    if FILLFCB(0) != 0: COMMAND_ERROR()
    SETDISK_FOR_COMMAND()

    BDOS_DELETE(&comfcb)
    if not BDOS_MAKE(&comfcb):
        PRINT0Z("NO SPACE\0")
        return

    comrec = 0
    addr = TPA_BASE

    for s in 0 .. sectors-1:
        BDOS_SETDMA(addr)
        if BDOS_WRITE(&comfcb) != 0:
            PRINT0Z("NO SPACE\0")
            break
        addr += 128

    BDOS_SETDMA(BUFF)
```

### REN

```pseudocode
function DO_REN():
    if FILLFCB(0) != 0: COMMAND_ERROR()
    savedNewDrive = sdisk

    SETDISK_FOR_COMMAND()
    if BDOS_SFIRST(&comfcb):
        PRINT0Z("FILE EXISTS\0")
        return

    memcopy(dst=&comfcb[16], src=&comfcb[0], count=16)

    p = SKIP_BLANKS(comaddr)
    if *p not in ['=', LA]:
        RESETDISK_AFTER_COMMAND()
        COMMAND_ERROR()
    comaddr = p + 1

    if FILLFCB(0) != 0:
        RESETDISK_AFTER_COMMAND()
        COMMAND_ERROR()

    # drive conflict: old and new must match if both specified
    if (sdisk != 0) and (savedNewDrive != 0) and (sdisk != savedNewDrive):
        RESETDISK_AFTER_COMMAND()
        COMMAND_ERROR()
    sdisk = (savedNewDrive != 0) ? savedNewDrive : sdisk

    comfcb[0] = 0
    if not BDOS_SFIRST(&comfcb):
        PRINT0Z("NO FILE\0")
        return

    BDOS_RENAME(&comfcb)
```

### USER

```pseudocode
function DO_USER():
    n = GETNUMBER_FROM_LINE()
    if n >= 16: COMMAND_ERROR()
    BDOS_USER_SET(n)
```

---

## Transient program / disk-switch handler

```pseudocode
function DO_TRANSIENT_OR_DISKSWITCH():
    # If command name blank, allow “B:” disk switch
    if comfcb[1] == ' ':
        if sdisk == 0: return
        cdisk = sdisk - 1
        SETDISKA_FROM_CDISK()
        BDOS_SELDSK(cdisk)
        return

    # Must have blank type; CCP forces .COM
    if comfcb[9] != ' ': COMMAND_ERROR()
    SETDISK_FOR_COMMAND()
    comfcb[9]='C'; comfcb[10]='O'; comfcb[11]='M'

    if not BDOS_OPEN(&comfcb):
        RESETDISK_AFTER_COMMAND()
        COMMAND_ERROR()

    # Load into memory from 0100h upward
    addr = TPA_BASE
    while true:
        BDOS_SETDMA(addr)
        status = BDOS_READ(&comfcb)
        if status == 0:
            addr += 128
            if addr >= CCP_LIMIT:
                PRINT0Z("BAD LOAD\0")
                return
            continue
        if status == 1: break     # EOF
        PRINT0Z("BAD LOAD\0")
        return

    RESETDISK_AFTER_COMMAND()

    # Parse up to two FCB arguments from the remainder of the line
    FILLFCB(0)
    comfcb[0] = sdisk
    FILLFCB(16)
    comfcb[16] = sdisk
    comrec = 0

    memcopy(dst=FCB_DEFAULT, src=&comfcb[0], count=33)

    # Build command tail in BUFF: [len][chars...][0]
    p = &combuf[0]
    while *p != 0 and *p != ' ': p++
    tailLen = 0
    dst = BUFF + 1
    while true:
        mem[dst] = *p
        if *p == 0: break
        tailLen++
        p++; dst++
    mem[BUFF] = tailLen

    CRLF()
    BDOS_SETDMA(BUFF)
    SAVEUSER_DISKA()

    call TPA_BASE      # execute transient (.COM)

    # on return:
    SP = &stack
    SETDISKA_FROM_CDISK()
    BDOS_SELDSK(cdisk)
    # state machine will go back to PROMPT
```

---

If you want a third style, I can also emit the same state machine as a **single continuous “switch(state)” loop** (C-like), but the version above keeps the CCP’s original subroutine boundaries readable while still making the global control flow totally explicit.

