# Pandemonium! (Sega Saturn, 1997) reverse-engineering notes and symbols

Hand-named function map, review records, study notes and tools for the Sega Saturn port of
**Pandemonium!** (Crystal Dynamics / Toys for Bob, 1996; Saturn port 1997). Four programs are covered:

- `KERN.BIN`, the Saturn main program at 0x06004000: 1095 functions, all hand-named (33 still marked provisional);
- `DUCK.BIN`, the Saturn FMV player module at 0x060f0000: 67 functions, all hand-named (2 provisional);
- `PANDY3.EXE`, the Windows 95 3Dfx Glide build (MSVC 4.2): 958 functions, 0 placeholders left (764 hand names, 184 provisional);
- `PANDY.EXE`, the Windows 95 software/DirectDraw build: 867 functions, 0 placeholders left (710 hand names, 89 provisional).

Same method as [sonicr-decomp](https://github.com/indieindie7/sonicr-decomp),
[scorcher-decomp](https://github.com/indieindie7/scorcher-decomp) and
[amok-decomp](https://github.com/indieindie7/amok-decomp): a first naming pass, BSim library matching,
a second review round in which every provisional name was re-read with its callers, then study passes on
the rail, movement and enemy code. Non-commercial, done for study and preservation.

**No game data is included**: no disc image, no program files, no unpacked data, no decompiled C,
no game text, no Ghidra database. You need your own copy of the game to use any of this.

## What is here

| Path | Content |
|---|---|
| `NOTES.md` | the working log: disc layout, Ghidra setup, BSim matching, both naming rounds, the study summary |
| `study_rail.md` | rail pieces, links, mirror twins, branches, how distance along a piece becomes a 3D position |
| `study_move.md` | player movement along the rail, jump buffer, knockback, impulse timers, character swap |
| `study_cam_enemy.md` | enemy state handlers, hitbox slots, swept contacts, triggers |
| `ghidra/hand_KERN.txt`, `ghidra/hand_DUCK.txt` | hand-named Saturn functions (`0x<addr> func <name> hand <comment>`) |
| `ghidra/pc/hand_PANDY3.txt`, `ghidra/pc/hand_PANDY.txt`, `ghidra/pc/funcinfo_*.tsv`, `ghidra/pc/n1_prompt_pc.txt` | the PC builds: hand names, per-function export (string-derived names replaced by `fn_<addr>`), the naming prompt |
| `ghidra/funcinfo_*.tsv` | per-function export: `F` address name source size, `C` callees (string rows removed) |
| `ghidra/naming/` | first-pass batches (`h1-h8`), the provisional list, review prompt and verdicts (`<prog> 0x<addr> <old> CONFIRM/RENAME/UNSURE <new> <reason>`) |
| `ghidra/scripts/` | `FindSh2FunctionsMem.java` (SH-2 function discovery), `DumpLong.java` |
| `ghidra/pmrun.sh` | headless Ghidra wrapper (set `GHIDRA_HEADLESS`, `PROJ_DIR`, `SP`) |
| `tools/` | gap and pointer scans, provisional review context and apply scripts |

`ApplyNames.java` and `ExportFuncInfo.java` come from the Scorcher / AMOK repos.

## Licence

Notes, names and tools: MIT (see `LICENSE`). Pandemonium! belongs to its rights holders; nothing from the
game is distributed here. Takedown requests will be honoured.

`ghidra/pc/review/` holds the second-round Sonnet verdicts (CONFIRM / RENAME / UNSURE per provisional name) for the two PC builds; after them PANDY3 keeps 6 and PANDY 7 provisional names.
