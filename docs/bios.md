# BIOS Function Reference (CP/M 2.2)

BIOS provides the machine-specific routines CP/M depends on. BDOS calls these via a jump table (3-byte JMPs) laid out consecutively at the top of the BIOS region.

This repository does not include a BIOS implementation; the routines below must be supplied by the target platform. Where the BDOS pseudocode shows `bios.*` calls, they refer to these externally provided entries.

## Call Conventions
- Entry via the BIOS jump table; each entry is a `JMP` to the routine.
- Registers: `A`, `B`, `C`, `D`, `E`, `H`, `L` per routine below. Return values are typically in `A` (and/or `BC`), with flags set/cleared as noted.
- Caller preserves registers unless documented otherwise (CP/M 2.2 convention keeps BDOS state intact).

## Function List (with typical callers)
- **BOOT (cold start)**  
  Args: none. Returns: none (transfers control to CCP/BDOS). Called only by the loader/ROM.

- **WBOOT (warm start)**  
  Args: none. Returns: none (transfers control). Called by BDOS function 0 and CCP on fatal errors.

- **CONST (console status)**  
  Args: none. Returns: `A` = 0xFF if a char is ready, 0x00 otherwise. Called by BDOS console status and direct console I/O.

- **CONIN (console input)**  
  Args: none. Returns: `A` = ASCII character from console. Called by BDOS buffered/direct console reads.

- **CONOUT (console output)**  
  Args: `C` = character to print. Returns: none. Called by BDOS console output and line editor echo.

- **LIST (list output)**  
  Args: `C` = character. Returns: none. Called by BDOS list output.

- **PUNCH (punch output)**  
  Args: `C` = character. Returns: none. Called by BDOS punch output.

- **READER (reader input)**  
  Args: none. Returns: `A` = character from reader device. Called by BDOS reader input.

- **HOME (disk home)**  
  Args: none. Returns: none. Called inside BDOS disk routines during reset/format (not shown explicitly in pseudocode).

- **SELDSK (select disk)**  
  Args: `C` = drive number (0=A, 1=B, ...). Returns: `HL` = DPH address or 0 if invalid. Called by BDOS `select_disk` and `reselect`.

- **SETTRK (set track)**  
  Args: `BC` = track number. Returns: none. Called by BDOS disk I/O helpers (`seqdiskread/write`, `randiskread/write`) before `read/write`.

- **SETSEC (set sector)**  
  Args: `BC` = logical sector number (post-translation). Returns: none. Called by BDOS disk I/O helpers before `read/write`.

- **SETDMA (set DMA address)**  
  Args: `BC` = memory address for sector transfers. Returns: none. Called by BDOS `setdata` (func 26) and before disk I/O.

- **READ (read sector)**  
  Args: current drive/track/sector/DMA. Returns: `A` = 0 success, 1 error. Called by BDOS disk readers.

- **WRITE (write sector)**  
  Args: current drive/track/sector/DMA. Returns: `A` = 0 success, 1 error. Called by BDOS disk writers.

- **LISTST (list status)**  
  Args: none. Returns: `A` = 0xFF if ready, 0x00 otherwise. Called by BDOS list status queries.

- **SECTRAN (sector translate)**  
  Args: `BC` = logical sector, `DE` = translation table address. Returns: `HL` = physical sector. Called by BDOS when DPB provides a skew table (see `sectran` usage in disk helpers).

## Notes
- The DPH returned by `SELDSK` points to the DPB, directory buffer, translation table, and allocation bitmap bases; BDOS caches these into its runtime variables.
- BDOS expects `READ/WRITE` to honor the DMA address set by `SETDMA` and the track/sector set by `SETTRK/SETSEC`.
- BIOS is free to implement buffering, skewing, or controller-specific retries as long as return codes follow the 0/1 convention.
