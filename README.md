# C6MacroSwitch

Interruttore/scene-controller Zigbee a batteria basato su ESP32-C6-WROOM-1-N8:
6 tasti meccanici Kailh Choc, LED di stato, alimentazione Li-ion 1S con LDO
very-low-dropout (XC6220B331), USB-C per programmazione via USB nativo,
charger MCP73831 predisposto ma non montato (DNP).

Progetto [Zener](https://github.com/diodeinc/pcb) (`pcb` CLI, richiede KiCad 10.x).

- Spec: `../docs/superpowers/specs/2026-06-11-c6-zigbee-macro-switch-design.md`
- Sourcing LCSC: `SOURCING.md`

## Comandi

```bash
pcb build C6MacroSwitch.zen    # valida il design
pcb layout C6MacroSwitch.zen   # sincronizza layout/C6MacroSwitch/layout.kicad_pcb
pcb open C6MacroSwitch.zen     # apre il layout in KiCad
pcb bom C6MacroSwitch.zen      # BOM con sourcing
```

Il round-trip KiCad 10 (salvataggio nativo → `pcb build`/`pcb layout`) è
verificato e funziona.

## Mappa GPIO

| GPIO | Funzione |
|---|---|
| IO0–IO5 | Tasti macro KEY1–KEY6 (LP-GPIO: wake da deep sleep, pull-up interni) |
| IO6 | ADC partitore batteria (1M/1M + 100nF) |
| IO8 | strapping, pullup 10k (download mode) |
| IO9 | Boot (pull-up 10k + tasto) |
| IO10 | LED di stato (verde, attivo alto) |
| IO12/IO13 | USB D−/D+ (USB nativo) |
| EN | Reset (pull-up 10k + 1µF + tasto) |

## Alimentazione

```
J_BAT (JST-PH) → Q_RVP/D_RVP (protezione inversione) → J_IMEAS (jumper misura corrente) → XC6220 → 3V3
USB VBUS → MCP73831 (DNP) → BAT+   |   USB VBUS → 1N5817W (DNP) → ingresso LDO
```

- Rimuovendo lo shunt da J_IMEAS si misura l'assorbimento totale a batteria.
- Protezione inversione batteria: montare ESATTAMENTE UNO tra Q_RVP
  (AO3401A, basse perdite) e D_RVP (1N5817W, semplice, ~300 mV di caduta);
  mai entrambi.
- D_USB_PWR (DNP) permette flash/test da USB senza batteria.
  **Attenzione:** mai montarlo con batteria collegata e jumper J_IMEAS
  inserito (bypassa il charger).
- Charger e relativi R/LED tutti DNP sulle prime board.
- Cutoff batteria firmware consigliato ~3,4 V (sotto, il rail esce di
  regolazione durante i burst TX).

## Note di popolamento

- SW1–SW6 (Kailh Choc CPG135001D01): esauriti su LCSC, montaggio in proprio.
- Tutti i passivi 0603 (LED 0402, unica taglia a catalogo house).
