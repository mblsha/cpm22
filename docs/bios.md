# BIOS Function Reference (CP/M 2.2)

BIOS provides the machine-specific routines CP/M depends on. BDOS calls these via a jump table (3-byte JMPs) laid out consecutively at the top of the BIOS region.

This repository does not include a BIOS implementation; the routines below must be supplied by the target platform. Where the BDOS pseudocode shows `bios.*` calls, they refer to these externally provided entries.

## Call Conventions
- Entry via the BIOS jump table; each entry is a `JMP` to the routine.
- Registers: `A`, `B`, `C`, `D`, `E`, `H`, `L` per routine below. Return values are typically in `A` (and/or `BC`), with flags set/cleared as noted.
- Caller preserves registers unless documented otherwise (CP/M 2.2 convention keeps BDOS state intact).

## Function List
- **BOOT (cold start)**  
  Args: none. Returns: none (transfers control to CCP/BDOS). Performs full hardware init, clears buffers, builds jump table, jumps to CCP.

- **WBOOT (warm start)**  
  Args: none. Returns: none (transfers control). Reinitializes disk system and CCP without full hardware reset.

- **CONST (console status)**  
  Args: none. Returns: `A` = 0xFF if a char is ready, 0x00 otherwise. No character consumed.

- **CONIN (console input)**  
  Args: none. Returns: `A` = ASCII character from console. Blocks until available; may set parity bit per implementation.

- **CONOUT (console output)**  
  Args: `C` = character to print. Returns: none. Outputs to console device.

- **LIST (list output)**  
  Args: `C` = character. Returns: none. Sends to list device (printer).

- **PUNCH (punch output)**  
  Args: `C` = character. Returns: none. Sends to punch device (paper tape).

- **READER (reader input)**  
  Args: none. Returns: `A` = character from reader device (paper tape), typically blocks.

- **HOME (disk home)**  
  Args: none. Returns: none. Seeks to track 0 on currently selected disk.

- **SELDSK (select disk)**  
  Args: `C` = drive number (0=A, 1=B, ...). Returns: `HL` = address of Disk Parameter Header (DPH) for that drive, or `HL=0` if invalid. Also selects drive for subsequent disk ops.

- **SETTRK (set track)**  
  Args: `BC` = track number. Returns: none. Positions controller to the given track for the current drive (logical track; may be remapped by BIOS).

- **SETSEC (set sector)**  
  Args: `BC` = logical sector number (per DPB layout or translated). Returns: none. Positions to sector within current track.

- **SETDMA (set DMA address)**  
  Args: `BC` = memory address for sector transfers. Returns: none. BDOS calls this with its `dmaad`; BIOS uses it for the next READ/WRITE.

- **READ (read sector)**  
  Args: uses current drive/track/sector/DMA. Returns: `A` = 0 for success, 1 for error. Performs a single sector read into DMA.

- **WRITE (write sector)**  
  Args: uses current drive/track/sector/DMA. Returns: `A` = 0 for success, 1 for error. Writes a single sector from DMA.

- **LISTST (list status)**  
  Args: none. Returns: `A` = 0xFF if ready, 0x00 otherwise. Indicates list device availability.

- **SECTRAN (sector translate)**  
  Args: `BC` = logical sector, `DE` = address of translate table. Returns: `HL` = physical sector to use. If no translation, typically returns `BC`.

## Notes
- The DPH returned by `SELDSK` points to the DPB, directory buffer, translation table, and allocation bitmap bases; BDOS caches these into its runtime variables.
- BDOS expects `READ/WRITE` to honor the DMA address set by `SETDMA` and the track/sector set by `SETTRK/SETSEC`.
- BIOS is free to implement buffering, skewing, or controller-specific retries as long as return codes follow the 0/1 convention.
