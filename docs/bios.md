# BIOS Function Reference (CP/M 2.2)

BIOS provides the machine-specific routines CP/M depends on. BDOS calls these via a jump table (3-byte JMPs) laid out consecutively at the top of the BIOS region.

This repository does not include a BIOS implementation; the routines below must be supplied by the target platform. Where the BDOS pseudocode shows `bios.*` calls, they refer to these externally provided entries.

## Call Conventions
- Entry via the BIOS jump table; each entry is a `JMP` to the routine.
- Registers: `A`, `B`, `C`, `D`, `E`, `H`, `L` per routine below. Return values are typically in `A` (and/or `BC`), with flags set/cleared as noted.
- Caller preserves registers unless documented otherwise (CP/M 2.2 convention keeps BDOS state intact).

## Function List (with BDOS callers)
- **BOOT (cold start)**  
  Args: none. Returns: none (transfers control to CCP/BDOS). Callers: loader/ROM only; BDOS never jumps to `bootf`.

- **WBOOT (warm start)**  
  Args: none. Returns: none (transfers control). Callers: BDOS `warm_boot` dispatch (function 0) and the `reboot` vector at 0000h used by `conbrk`/`rdech1` when Ctrl-C forces a restart.

- **CONST (console status)**  
  Args: none. Returns: `A` = 0xFF if a char is ready, 0x00 otherwise. Callers: `conbrk` stop-key handler (used by `con_output`, `print_string`, echoed console `buffered_read`, and `con_status`) and `direct_conio` (function 6) for `C=0xFE` status or `C=0xFF` input gating.

- **CONIN (console input)**  
  Args: none. Returns: `A` = ASCII character from console. Callers: `con_input` (function 1 via `conech`), `buffered_read` (function 10 line editor), `direct_conio` input branch (`C=0xFF`), and `conbrk` when it consumes a ready character (affecting `con_status` and the console output path).

- **CONOUT (console output)**  
  Args: `C` = character to print. Returns: none. Callers: `con_output` (function 2 via `tabout`/`conout`), `direct_conio` output branch (`C` other than 0xFE/0xFF), `print_string` (function 9 `print`), echoes inside `con_input`/`buffered_read` (`conech`/`ctlout`), and edit helpers such as `backup`/`crlfp`.

- **LIST (list output)**  
  Args: `C` = character. Returns: none. Callers: `list_output` (function 5 dispatch) and the console output pipeline (`con_output`, `print_string`, echoes in `buffered_read`) when the `listcp` toggle mirrors characters.

- **PUNCH (punch output)**  
  Args: `C` = character. Returns: none. Callers: `punch_output` (function 4 dispatch).

- **READER (reader input)**  
  Args: none. Returns: `A` = character from reader device. Callers: `reader_input` (function 3 dispatch).

- **HOME (disk home)**  
  Args: none. Returns: none. Callers: `home` helper invoked by `select` initialization (`reset_disks` function 13, `select_disk` function 14, and `reselect` when a new drive is logged in) and by `search` when starting directory scans (`search_first`, `open_file`, `rename_file`, `make_file`, `delete_file`, `set_attributes`, etc.).

- **SELDSK (select disk)**  
  Args: `C` = drive number (0=A, 1=B, ...). Returns: `HL` = DPH address or 0 if invalid. Callers: `selectdisk` inside `select`, reached from `reset_disks` (function 13), `select_disk` (function 14/`curselect`), and `reselect` which precedes most FCB operations (`open_file`, `close_file`, `search_first/next`, `delete_file`, `seq_read/seq_write`, `make_file`, `rename_file`, `set_attributes`, `random_read/random_write`, `file_size`, `random_write_zerofill`).

- **SETTRK (set track)**  
  Args: `BC` = track number. Returns: none. Callers: `seek` (sets track before `SETSEC`), which is used by `seek_dir` for directory I/O (`search_first/next`, `open_file`, `close_file`, `rename_file`, `delete_file`, `make_file`, `set_attributes`) and by data I/O (`diskread` for `seq_read`/`random_read`, `diskwrite` for `seq_write`/`random_write`/`random_write_zerofill`).

- **SETSEC (set sector)**  
  Args: `BC` = logical sector number (post-translation). Returns: none. Callers: `seek` after `SECTRAN`, with the same call sites as `SETTRK` (`seek_dir`, `diskread`, `diskwrite`, and `random_write_zerofill`).

- **SETDMA (set DMA address)**  
  Args: `BC` = memory address for sector transfers. Returns: none. Callers: `setdata` invoked by BDOS `set_dma` (function 26) and by `reset_disks` to restore the default DMA; `setdir`/`rd_dir`/`wrdir` swap DMA to the directory buffer around directory reads/writes used by `search_first/next`, `open_file`, `rename_file`, `delete_file`, `make_file`, `close_file`, `set_attributes`, etc.

- **READ (read sector)**  
  Args: current drive/track/sector/DMA. Returns: `A` = 0 success, 1 error. Callers: `rdbuff` wrapper used by `read_dir` (directory scans for `search`/`open`/`rename`/`delete`/`make`/`close`/`set_attributes`) and by `diskread` (powers BDOS `seq_read` and `random_read`).

- **WRITE (write sector)**  
  Args: current drive/track/sector/DMA. Returns: `A` = 0 success, 1 error. Callers: `wrbuff` wrapper used by `wrdir` (directory updates during `close`, `delete_file`, `make_file`, `rename_file`, `set_attributes`, etc.) and by `diskwrite` (used for `seq_write`, `random_write`, and `random_write_zerofill`).

- **LISTST (list status)**  
  Args: none. Returns: `A` = 0xFF if ready, 0x00 otherwise. Callers: none in this BDOS 2.2 build (no BDOS function issues `LISTST`).

- **SECTRAN (sector translate)**  
  Args: `BC` = logical sector, `DE` = translation table address. Returns: `HL` = physical sector. Callers: `seek` when `tranv` is populated by `SELDSK`; applies to directory seeks (`seek_dir` in `search`/`open`/`rename`/`delete`/`make`/`close`) and data I/O (`diskread`/`diskwrite` for sequential and random accesses).

## Notes
- The DPH returned by `SELDSK` points to the DPB, directory buffer, translation table, and allocation bitmap bases; BDOS caches these into its runtime variables.
- BDOS expects `READ/WRITE` to honor the DMA address set by `SETDMA` and the track/sector set by `SETTRK/SETSEC`.
- BIOS is free to implement buffering, skewing, or controller-specific retries as long as return codes follow the 0/1 convention.
