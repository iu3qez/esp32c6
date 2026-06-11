# Design: C6 Zigbee Macro Switch (variante Zener)

Data: 2026-06-11
Branch: `dev-zener`
Stato: in attesa di approvazione spec

## Contesto e motivazione

Il progetto atopile originale (dev board ESP32-C6 con CH340G, doppio USB-C,
header verso keypad esterno) è bloccato: atopile ≤0.15.7 non rilegge i
`.kicad_pcb` salvati da KiCad 10 (issue atopile#1822). Si riparte prendendo
solo l'idea di fondo e si migra a `pcb` di Diode Inc. (linguaggio Zener,
file `.zen`), che richiede nativamente KiCad 10.x.

Il vecchio progetto ato resta nel repo come riferimento, intatto.

## Cosa si costruisce

Un interruttore/scene-controller **Zigbee a batteria** basato su ESP32-C6:
6 tasti macro, 1 LED di stato, alimentazione Li-ion 1S con LDO
very-low-dropout / low-IQ, USB-C per programmazione, charger predisposto ma
non montato (DNP) sulle prime board.

## Vincoli vincolanti (richieste esplicite)

1. **Modulo**: ESP32-C6-WROOM-1-N8, identico al progetto precedente
   (LCSC **C5366877**) — già disponibile in casa.
2. **Connettore USB-C**: identico al progetto precedente, Korean Hroparts
   TYPE-C-31-M-12 (LCSC **C165948**).
3. **Tutti gli altri componenti**: disponibili su **LCSC con stock > 0** al
   momento del part picking.
4. **Resistenze e condensatori: package 0603**.
5. Toolchain: `diodeinc/pcb` (Zener), nuovo branch `dev-zener`.

## Architettura

```
USB-C (C165948) ──VBUS──> [Charger MCP73831/TP4054, DNP] ──> BAT+
   │ D+/D-                                                    │
   │                                              JST-PH 2p ──┤
   └──> ESP32-C6 USB nativo (GPIO12/13)                       │
                                                              ▼
                                            J_IMEAS (jumper misura corrente)
                                                              │
                                                              ▼
                                       LDO 3.3V VLDO/low-IQ (XC6220B331*)
                                                              │
                                                              ▼
ESP32-C6-WROOM-1-N8 (C5366877) <── 3V3 ── partitore ADC batteria (≥1 MΩ)
   ├── GPIO0..GPIO5: 6 tasti macro (2×3), LP-GPIO, wake da deep sleep,
   │                 pull-up interni → nessuna resistenza esterna
   ├── GPIO8 (o libero): LED di stato (normalmente spento)
   ├── EN: pull-up 10k + C 1uF + tasto reset
   └── GPIO9: tasto boot (pull-up interno/10k)
```

\* Candidato primario LDO: Torex XC6220B331 (1 A, dropout ~100 mV @ 300 mA,
IQ ~8 µA). Fallback se non a stock su LCSC: TLV755P-33, ME6217C33.
La scelta finale si fa in fase di picking col vincolo stock > 0.

## Componenti

| Blocco | Parte | Note |
|---|---|---|
| MCU/radio | ESP32-C6-WROOM-1-N8 | LCSC C5366877, fissato |
| USB-C | TYPE-C-31-M-12 | LCSC C165948, fissato; CC1/CC2 pulldown 5.1k 0603 |
| LDO | XC6220B331 (candidato) | SOT-25; cap in/out ceramici 0603 (valori da datasheet, tipicamente 1–10 µF) |
| Charger (DNP) | MCP73831 o TP4054, SOT-23-5 | + R prog 0603 + LED charge; footprint montati, BOM vuota |
| Batteria | JST-PH 2 pin SMT | Li-ion/LiPo 1S |
| Misura corrente | header 2 pin 2.54 mm + shunt | in serie al positivo batteria (a monte dell'LDO): shunt rimosso → amperometro in serie; misura il consumo totale incluso IQ dell'LDO |
| Alimentazione da USB senza batteria | Schottky SOD-123 (DNP) | da VBUS all'ingresso LDO (a valle del jumper): permette flash/test senza batteria; DNP se non serve. L'LDO regge Vin 5 V |
| Monitor batteria | 2× R ≥1 MΩ 0603 + C filtro | su ADC del C6 |
| Tasti | 7× tattile SMD (6 macro + boot) + 1 reset | stesso modello per tutti, LCSC stock >0 |
| LED stato | LED 0603 + R 1k 0603 | su GPIO libero |
| EN | R 10k + C 1 µF 0603 | reset RC classico |

Decoupling: 100 nF 0603 vicino al modulo + bulk 10 µF 0603 sul rail 3V3
(il WROOM ha già i regolatori interni di decoupling principali on-module;
si segue la app note Espressif).

## Connessioni GPIO

| GPIO | Funzione | Perché |
|---|---|---|
| GPIO0–GPIO5 | 6 tasti macro verso GND | LP-GPIO: deep-sleep wake, pull-up interni |
| GPIO9 | Boot + tasto | strapping pin standard |
| GPIO12/13 | USB D-/D+ | USB nativo (flash + console) |
| EN | Reset + RC | |
| GPIO8 | LED stato | strapping ma utilizzabile come output LED |
| GPIO6 | ADC partitore batteria | ADC1_CH6, ultimo canale ADC libero (canali ADC del C6: GPIO0–6) |

Nota: se in fase di implementazione emerge un conflitto su GPIO6, si sposta
un tasto su un GPIO non-LP accettando il wake solo dagli altri 5 tasti.

## Struttura progetto Zener

```
zener/                      # nuovo workspace pcb, non tocca il progetto ato
  pcb.toml                  # manifest board + dipendenze registry
  C6MacroSwitch.zen         # board top-level
  components/               # parti locali (WROOM-1-N8, TYPE-C-31-M-12, ...)
    ...                     # .zen + .kicad_sym + .kicad_mod riusati dal vecchio repo dove possibile
  layout/                   # output pcb layout (.kicad_pcb)
```

- Generici (R, C, LED, JST, tactile) da `@stdlib` o dal registry diodeinc
  (`pcb search`), con vincolo LCSC stock > 0.
- Per WROOM-1-N8 e TYPE-C-31-M-12 si riusano simbolo/footprint KiCad già
  presenti in `parts/` del progetto ato (sono già verificati su board
  precedente), incapsulati in componenti Zener locali.
- Skill di lavoro installate in `.claude/skills/` (zener-language,
  registry-search, librarian, datasheet-reader, spice-sim): da usare in
  implementazione.

## Verifica / criteri di successo

1. `pcb build` senza errori sul board top-level.
2. `pcb layout` genera un `.kicad_pcb` che KiCad 10.0.3 apre senza warning
   di formato; round-trip salvataggio KiCad 10 → `pcb build` OK.
3. BOM: ogni parte (tranne modulo già in casa) con codice LCSC e stock > 0
   alla data del picking; R/C tutti 0603.
4. Netlist coerente con la tabella GPIO sopra (review manuale).

## Fuori scope

- Layout/sbroglio finale e forma della scheda (fase successiva).
- Firmware Zigbee.
- Ricarica montata (prime board: charger DNP).
