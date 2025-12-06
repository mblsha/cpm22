# CCP and BDOS Integration

- Memory map: The Makefile builds CCP at `9400h` and BDOS at `9c00h`, keeping the expected 0x800-byte gap. `bdosl` in `ccp.asm` computes the BDOS base from the CCP origin; if you move one, move both.
- Call interface: CCP invokes BDOS through the CP/M entry vector at `0005h` (`bdos` equate). User programs do the same, so BDOS must be fixed at the advertised address when binaries are linked into a bootable image.
- Startup flow: BIOS boots CCP; CCP initializes the command buffer and either executes a preset command (`combuf`/`comlen`) or prompts the console. Built-ins (e.g., `DIR`, `ERA`, `TYPE`) use BDOS functions for file search, open/close, and console I/O.
- Device path: BDOS bridges to the BIOS jump table for hardware operations (console, disk seeks, DMA addresses). CCP never calls BIOS directly; it relies on BDOS for uniform device semantics.
- Relocation considerations: If you change origins or add padding, ensure `p2bin -r` ranges in the Makefile still match the assembled addresses and that CCP’s assumptions about BDOS constants (`bdos`, `bdosl`, control character equates) remain valid.
