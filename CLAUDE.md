# CLAUDE.md

## Progetto (branch dev-zener)

C6 Zigbee Macro Switch: interruttore Zigbee a batteria con ESP32-C6.
Design spec: docs/superpowers/specs/2026-06-11-c6-zigbee-macro-switch-design.md
Toolchain: `pcb` di Diode Inc. (linguaggio Zener, file .zen) — NON più atopile.

Il workspace Zener è la root del repo: board top-level `C6MacroSwitch.zen`,
manifest `pcb.toml`, componenti riusabili in `components/`, layout in `layout/`.
Il vecchio progetto atopile (main.ato, parts/, layouts/) è stato rimosso —
recuperabile da git history se servisse. atopile ≤0.15.7 non rilegge PCB salvati
da KiCad 10 (github.com/atopile/atopile/issues/1822): motivo della migrazione.

## Vincoli hardware non negoziabili

- Modulo: ESP32-C6-WROOM-1-N8 (LCSC C5366877) — già in casa, non sostituire
- USB-C: Korean Hroparts TYPE-C-31-M-12 (LCSC C165948)
- Tasti macro: 6× Kailh Choc CPG135001D01 (LCSC C400229), switch meccanici
- Resistenze e condensatori: package 0603
- Ogni altro componente: disponibile su LCSC con stock > 0
- LDO: very-low-dropout e low-IQ (candidato XC6220B331)

## Toolchain pcb/Zener

- Richiede KiCad 10.x (installato: 10.0.3) — il round-trip KiCad 10 funziona
- Skill di progetto in .claude/skills/: usa `zener-language` prima di toccare
  file .zen, `registry-search` per cercare parti, `datasheet-reader` per i PDF,
  `librarian` per creare componenti riusabili
- Workflow: `pcb build` (valida) → `pcb layout` (genera .kicad_pcb) → `pcb open`
- Dopo modifiche agli import: `pcb sync` poi `pcb build`
- Docs aggiornate: usa context7 (/diodeinc/pcb, /diodeinc/stdlib) quando possibile

## Tool MCP

Se un tool MCP fallisce, riporta sempre l'errore completo all'utente
(tool, parametri, messaggio) — non aggirare in silenzio.
