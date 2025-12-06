# Repository Guidelines

## Project Structure & Module Organization
- Root contains `ccp.asm` (Console Command Processor), `bdos.asm` (BDOS kernel), `Makefile`, and `README.md`.
- Build artifacts (`*.p`, `*.lst`, `*.bin`) are generated in the root; keep the repo clean by running `make clean` before commits.
- No nested src/test directories; add new tooling or scripts alongside the Makefile with clear names.

## Build, Test, and Development Commands
- `make` or `make all`: assemble both modules using Macro Assembler AS (`asl`) and convert to binaries with `p2bin`; origins default to 0x9400 (CCP) and 0x9C00 (BDOS).
- `make clean`: remove generated `.p`, `.lst`, `.bin` files.
- To experiment with other load addresses, pass `origin` to `asl`, e.g. `asl -D origin=9000h -o bdos.p -L bdos.asm`, then `p2bin -r '$9000-$a9ff' bdos.p`.

## Coding Style & Naming Conventions
- 8080 assembly with one instruction per line; keep the existing lower-case opcodes/directives and tab-indented operands for alignment.
- Labels start at column 1; constants use `equ` with `h` for hex; prefer underscores over `$` in symbols.
- Comments start with `;` and align in a column for readability; use double quotes for strings. Avoid the old ASM80 `!` multi-instruction syntax.

## Testing Guidelines
- Successful assembly without warnings is the primary check. Run `make` and ensure both binaries are produced.
- If altering memory layout, verify segment ranges in `p2bin -r` and sanity-check binary lengths with `ls -l` or `cmp` against known images.
- Keep manual test notes (commands run, warnings addressed) in PR descriptions.

## Commit & Pull Request Guidelines
- Use short, imperative commit subjects (e.g., `adjust bdos origin guard`) with optional bodies describing rationale and impacts.
- Keep commits focused; avoid mixing formatting-only changes with logic changes.
- For PRs, include a concise summary, rationale, and build results (`make`/`make clean` as applicable). Mention toolchain versions if relevant.
- After opening a PR, watch required CI via `gh pr checks --watch --interval 5 --required` to confirm status.
