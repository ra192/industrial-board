# AGENTS.md

KiCad hardware project for an STM32F103C8 industrial I/O board: 13 opto-isolated digital inputs, 10 isolated digital outputs, RS-485, ACS712 current sensing, 24 V input with isolated 5 V and 3.3 V rails, CR2032 backup. There is no software build, test suite, package manager, or CI; verification means KiCad ERC/DRC and schematic-to-PCB parity.

## Toolchain and file format

- `kicad-cli` ERC/DRC/export commands do not modify the tracked design files. Avoid `--save-board` and `upgrade` on them.
- `~*.lck` files mean this project is open in the GUI (check for a running `kicad` process). Close it before editing or regenerating files, since the GUI can overwrite disk changes. Lock files are untracked; never commit them.
- The board/schematic are UUID-heavy S-expressions with embedded symbol copies and net codes. Hand-edits are possible but produce huge diffs; prefer the GUI or a reviewed script, and keep one logical change per commit.

## Design layout

- `industrial-board.kicad_sch` is the root sheet: STM32F103C8, MAX3485 RS-485, TMA-2405D isolated DC-DC, AMS1117-3.3, battery backup, crystals, connectors. It instantiates the two child sheets.
- `input.kicad_sch` contains the 13 digital inputs (J2 pins 2-14) and the ACS712 channel (J5 `IP+`/`IP-`).
- `output.kicad_sch` contains the 10 digital outputs (J4 pins 3-12).
- Connectors: J1 6-pin SWD, J2 15-pin digital inputs, J3 3-pin RS-485, J4 14-pin digital outputs, J5 4-pin 24 V/current input.
- `Project_Library.kicad_sym` and `sym-lib-table` define only `A5970AD`, which the schematics do not use. Design symbols are embedded copies of global-library symbols.
- Board outline is 100 x 100 mm at (50, 50)-(150, 150), 2 layers, 1.6 mm thick, with M3 mounting holes. Both layers have `GND` and `GND_IN` pours; refill zones after moving copper.
- `GND` (logic side, TMA secondary) and `GND_IN` (24 V field side) are separate nets. Do not bridge them to silence DRC. Existing quirk: J3 pin 3 uses `GND_IN`, while the MAX3485 is powered from `GND`/`+3.3V`; verify intent before changing the RS-485 interface.
- Default netclass: 0.2 mm clearance and track, 0.6/0.3 mm via, 0.5 mm copper-to-edge, 0.25 mm hole clearances. Existing routing uses 0.2/0.3/0.6 mm tracks.

## Verification

Write reports to `/tmp` so generated files never enter the repo:

```sh
kicad-cli sch erc --severity-all -o /tmp/erc.rpt industrial-board.kicad_sch
kicad-cli pcb drc --severity-all --refill-zones --schematic-parity -o /tmp/drc.rpt industrial-board.kicad_pcb
```

Baseline with kicad-cli 10.0.8 (checked 2026-11):

- ERC: 0 errors, 37 warnings (34 `lib_symbol_mismatch`, 2 `same_local_global_label`, 1 `lib_symbol_issues`).
- DRC: 14 errors, 91 warnings. All errors are `starved_thermal`; all warnings are `lib_footprint_mismatch` from installed libraries differing from embedded footprints. Unconnected items: 0. Schematic parity issues: 0.

Treat new categories or counts above baseline as regressions. Do not "fix" library mismatches by updating footprints/symbols from the installed libraries, because that changes pads or pinouts. `--exit-code-violations` currently fails because of the 14 known thermals.

## Fabrication outputs

```sh
kicad-cli pcb export gerbers --board-plot-params -o gerber industrial-board.kicad_pcb
kicad-cli pcb export drill -o gerber industrial-board.kicad_pcb
kicad-cli sch export bom -o /tmp/bom.csv industrial-board.kicad_sch
```

`gerber/` is the configured plot directory, but it is neither tracked nor gitignored; do not commit generated outputs. There is no jobset file.
