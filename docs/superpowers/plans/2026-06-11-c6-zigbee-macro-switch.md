# C6 Zigbee Macro Switch — Implementation Plan (Zener/pcb)

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Board Zener completa e validata (`pcb build` pulito, layout generato apribile in KiCad 10) per l'interruttore Zigbee a batteria con ESP32-C6, secondo la spec `docs/superpowers/specs/2026-06-11-c6-zigbee-macro-switch-design.md`.

**Architecture:** Workspace `pcb` (board repository) in `zener/` sul branch `dev-zener`. Componenti specifici come moduli Zener locali in `zener/components/`, riusando i `.kicad_sym`/`.kicad_mod` già verificati in `parts/` del progetto ato. Generici (R, C, LED, header) da `@stdlib`. Top-level `C6MacroSwitch.zen` con catena di potenza batteria→jumper→LDO→3V3, 6 tasti Choc su GPIO0–5, LED su GPIO8, USB nativo su IO12/13.

**Tech Stack:** `pcb` CLI (diodeinc), linguaggio Zener (Starlark), KiCad 10.0.3, stdlib `@stdlib`.

**Regole trasversali (valgono per OGNI task):**
- Prima di scrivere/modificare `.zen`: skill `zener-language` già caricata in `.claude/skills/`.
- Dopo ogni modifica agli import: `pcb sync` dentro `zener/`, poi `pcb build`.
- R/C sempre `package="0603"`.
- Ogni parte nuova: verificare LCSC stock > 0 (vedi Task 8) prima di fissarla.
- Preservare eventuali commenti `# pcb:sch`.
- Commit frequenti, messaggi conventional-commits, MAI Co-Authored-By.

---

### Task 1: Installare il tool `pcb`

**Files:** nessuno (toolchain).

- [ ] **Step 1: Installa**

```bash
curl -fsSL https://raw.githubusercontent.com/diodeinc/pcb/main/install.sh | bash
```

Installa in `$HOME/.local/bin` (già nel PATH: `ato` sta lì).

- [ ] **Step 2: Verifica**

```bash
pcb --version
```

Atteso: versione ≥ 0.3.92. Se il comando non esiste, `export PATH="$HOME/.local/bin:$PATH"`.

- [ ] **Step 3: Verifica che trovi KiCad**

```bash
kicad-cli version   # atteso: 10.0.3
```

Nessun commit (nessun file di repo toccato).

---

### Task 2: Scaffold del workspace `zener/`

**Files:**
- Create: `zener/pcb.toml`
- Create: `zener/C6MacroSwitch.zen`

- [ ] **Step 1: Genera lo scaffold con il tool** (così la struttura è quella canonica della versione installata)

```bash
cd /home/sf/src/esp32c6/esp32c6
pcb new board C6MacroSwitch https://github.com/iu3qez/esp32c6
```

Se `pcb new board` crea una directory `C6MacroSwitch/`, rinominarla in `zener/`:

```bash
mv C6MacroSwitch zener
```

Ispeziona cosa ha generato (`ls -R zener/`) e adatta i passi seguenti alla struttura reale; i contenuti sotto sono il riferimento atteso.

- [ ] **Step 2: `zener/pcb.toml`** (verifica/adatta quello generato)

```toml
[workspace]
name = "esp32c6-zener"

[board]
name = "C6MacroSwitch"
path = "C6MacroSwitch.zen"
description = "ESP32-C6 Zigbee battery-powered macro switch"
```

- [ ] **Step 3: Board minimale `zener/C6MacroSwitch.zen`**

```zen
Board(name="C6MacroSwitch", layout_path="layout/C6MacroSwitch", layers=2)

VBAT = Power("VBAT")
VDD_3V3 = Power("3V3")
GND = Ground("GND")
```

- [ ] **Step 4: Valida**

```bash
cd zener && pcb build
```

Atteso: build OK, zero errori (board vuota è legale; se `Board()` richiede argomenti diversi nella versione installata, vedere `pcb doc --package @stdlib` per `board_config`).

- [ ] **Step 5: Commit**

```bash
git add zener && git commit -m "feat(zener): scaffold pcb workspace for C6MacroSwitch"
```

---

### Task 3: Componente ESP32-C6-WROOM-1-N8 (riuso footprint/symbol)

**Files:**
- Create: `zener/components/ESP32C6WROOM1N8/ESP32C6WROOM1N8.zen`
- Copy: `parts/Espressif_Systems_ESP32_C6_WROOM_1_N8/ESP32-C6-WROOM-1-N8.kicad_sym` → `zener/components/ESP32C6WROOM1N8/`
- Copy: `parts/Espressif_Systems_ESP32_C6_WROOM_1_N8/WIRELM-SMD_ESP32-C6-WROOM-1.kicad_mod` → `zener/components/ESP32C6WROOM1N8/`

- [ ] **Step 1: Copia symbol e footprint**

```bash
mkdir -p zener/components/ESP32C6WROOM1N8
cp parts/Espressif_Systems_ESP32_C6_WROOM_1_N8/ESP32-C6-WROOM-1-N8.kicad_sym zener/components/ESP32C6WROOM1N8/
cp parts/Espressif_Systems_ESP32_C6_WROOM_1_N8/WIRELM-SMD_ESP32-C6-WROOM-1.kicad_mod zener/components/ESP32C6WROOM1N8/
```

- [ ] **Step 2: Ispeziona i nomi pin del symbol** (servono per la mappa `pins`)

```bash
grep -oE '\(name "[^"]+"' zener/components/ESP32C6WROOM1N8/ESP32-C6-WROOM-1-N8.kicad_sym | sort -u
grep -oE '\(number "[^"]+"' zener/components/ESP32C6WROOM1N8/ESP32-C6-WROOM-1-N8.kicad_sym | sort -u
```

Pinout di riferimento (dal part ato, verificato su board precedente): EN=3, GND=1,28–37, IO0=8, IO1=9, IO2=27, IO3=26, IO4=4, IO5=5, IO6=6, IO7=7, IO8=10, IO9=15, IO10=11, IO11=12, IO12=13, IO13=14, IO15=23, IO18=16, IO19=17, IO20=18, IO21=19, IO22=20, IO23=21, P3V3=2, RXD0=24, TXD0=25.

- [ ] **Step 3: Scrivi `ESP32C6WROOM1N8.zen`**

```zen
EN = io(Net)
P3V3 = io(Power)
GND = io(Ground)
IO0 = io(Net); IO1 = io(Net); IO2 = io(Net); IO3 = io(Net)
IO4 = io(Net); IO5 = io(Net); IO6 = io(Net); IO7 = io(Net, optional=True)
IO8 = io(Net); IO9 = io(Net)
IO10 = io(Net, optional=True); IO11 = io(Net, optional=True)
IO12 = io(Net); IO13 = io(Net)
IO15 = io(Net, optional=True); IO18 = io(Net, optional=True)
IO19 = io(Net, optional=True); IO20 = io(Net, optional=True)
IO21 = io(Net, optional=True); IO22 = io(Net, optional=True)
IO23 = io(Net, optional=True)
RXD0 = io(Net, optional=True); TXD0 = io(Net, optional=True)

Component(
    name = "ESP32C6WROOM1N8",
    symbol = Symbol(library = "./ESP32-C6-WROOM-1-N8.kicad_sym"),
    footprint = "./WIRELM-SMD_ESP32-C6-WROOM-1.kicad_mod",
    part = Part(mpn = "ESP32-C6-WROOM-1-N8", manufacturer = "Espressif Systems"),
    pins = {
        "EN": EN, "P3V3": P3V3, "GND": GND,
        "IO0": IO0, "IO1": IO1, "IO2": IO2, "IO3": IO3,
        "IO4": IO4, "IO5": IO5, "IO6": IO6, "IO7": IO7,
        "IO8": IO8, "IO9": IO9, "IO10": IO10, "IO11": IO11,
        "IO12": IO12, "IO13": IO13, "IO15": IO15, "IO18": IO18,
        "IO19": IO19, "IO20": IO20, "IO21": IO21, "IO22": IO22,
        "IO23": IO23, "RXD0": RXD0, "TXD0": TXD0,
    },
)
```

Note d'adattamento (decidere in base all'output di Step 2 e di `pcb build`):
- Se il symbol espone i pin per NUMERO e non per nome, le chiavi di `pins` diventano i numeri (`"3": EN, "2": P3V3, ...` secondo il pinout di riferimento sopra).
- Se il symbol contiene già la property footprint corretta, rimuovere `footprint=` dal `Component()` (il symbol è source of truth).
- Aggiungere la property LCSC: se `pcb build`/BOM non la riporta, aggiungere `properties = {"LCSC": "C5366877"}` al `Component()`.

- [ ] **Step 4: Valida il componente**

```bash
cd zener && pcb build components/ESP32C6WROOM1N8/ESP32C6WROOM1N8.zen
```

Atteso: build OK. Errori tipici: pin del symbol non mappato (aggiungerlo a `pins` con `io(Net, optional=True)`), nome pin sbagliato (correggere chiave).

- [ ] **Step 5: Commit**

```bash
git add zener/components/ESP32C6WROOM1N8
git commit -m "feat(zener): ESP32-C6-WROOM-1-N8 component (reused verified sym/fp)"
```

---

### Task 4: Componente USB-C TYPE-C-31-M-12

**Files:**
- Create: `zener/components/TYPEC31M12/TYPEC31M12.zen`
- Copy: `parts/Korean_Hroparts_Elec_TYPE_C_31_M_12/TYPE-C-31-M-12.kicad_sym` e `USB-C_SMD-TYPE-C-31-M-12_1.kicad_mod` → `zener/components/TYPEC31M12/`

- [ ] **Step 1: Copia file**

```bash
mkdir -p zener/components/TYPEC31M12
cp parts/Korean_Hroparts_Elec_TYPE_C_31_M_12/TYPE-C-31-M-12.kicad_sym zener/components/TYPEC31M12/
cp parts/Korean_Hroparts_Elec_TYPE_C_31_M_12/USB-C_SMD-TYPE-C-31-M-12_1.kicad_mod zener/components/TYPEC31M12/
```

- [ ] **Step 2: Ispeziona pin del symbol** (stesso metodo grep del Task 3, Step 2).

Pinout di riferimento ato: CC1=A5, CC2=B5, D_N=A7+B7, D_P=A6+B6, EH=1..4, GND=A1B12+B1A12, SBU1=A8, SBU2=B8, VBUS=A4B9+B4A9.

- [ ] **Step 3: Scrivi `TYPEC31M12.zen`**

```zen
load("@stdlib/interfaces.zen", "Usb2")

VBUS = io(Power)
GND = io(Ground)
USB = io(Usb2)
CC1 = io(Net)
CC2 = io(Net)
SBU1 = io(Net, optional=True)
SBU2 = io(Net, optional=True)
SHIELD = io(Net)

Component(
    name = "TYPEC31M12",
    symbol = Symbol(library = "./TYPE-C-31-M-12.kicad_sym"),
    footprint = "./USB-C_SMD-TYPE-C-31-M-12_1.kicad_mod",
    part = Part(mpn = "TYPE-C-31-M-12", manufacturer = "Korean Hroparts Elec"),
    pins = {
        "VBUS": VBUS, "GND": GND,
        "D_P": USB.DP, "D_N": USB.DM,
        "CC1": CC1, "CC2": CC2,
        "SBU1": SBU1, "SBU2": SBU2,
        "EH": SHIELD,
    },
)
```

Note d'adattamento: come Task 3 (chiavi `pins` = nomi reali del symbol; se i pad multipli A4B9/B4A9 sono pin distinti nel symbol, mappare entrambi sulla stessa net: `"A4B9": VBUS, "B4A9": VBUS`). Verificare i nomi dei campi di `Usb2` con `pcb doc --package @stdlib` (`DP`/`DM` vs `D_P`/`D_N`) e usare quelli reali.

- [ ] **Step 4: Valida**

```bash
cd zener && pcb build components/TYPEC31M12/TYPEC31M12.zen
```

- [ ] **Step 5: Commit**

```bash
git add zener/components/TYPEC31M12
git commit -m "feat(zener): TYPE-C-31-M-12 USB-C connector component"
```

---

### Task 5: Componente Kailh Choc CPG135001D01

**Files:**
- Create: `zener/components/KailhChoc/KailhChoc.zen`
- Copy: `parts/Kailh_CPG135001D01/CPG135001D01.kicad_sym` e `KEY-TH_CPG135001D0X.kicad_mod` → `zener/components/KailhChoc/`

- [ ] **Step 1: Copia file**

```bash
mkdir -p zener/components/KailhChoc
cp parts/Kailh_CPG135001D01/CPG135001D01.kicad_sym zener/components/KailhChoc/
cp parts/Kailh_CPG135001D01/KEY-TH_CPG135001D0X.kicad_mod zener/components/KailhChoc/
```

- [ ] **Step 2: Scrivi `KailhChoc.zen`** (2 pin: 1 e 2)

```zen
P1 = io(Net)
P2 = io(Net)

Component(
    name = "KailhChoc",
    symbol = Symbol(library = "./CPG135001D01.kicad_sym"),
    footprint = "./KEY-TH_CPG135001D0X.kicad_mod",
    part = Part(mpn = "CPG135001D01", manufacturer = "Kailh"),
    pins = {"1": P1, "2": P2},
)
```

(Adattare chiavi `pins` ai nomi reali del symbol come nei task precedenti.)

- [ ] **Step 3: Valida e committa**

```bash
cd zener && pcb build components/KailhChoc/KailhChoc.zen
git add zener/components/KailhChoc
git commit -m "feat(zener): Kailh Choc CPG135001D01 switch component"
```

---

### Task 6: Componenti riusati minori — tactile button e Schottky

**Files:**
- Create: `zener/components/TactileTSA061B/TactileTSA061B.zen` (+ sym/fp copiati da `parts/BZCN_TSA061B2808B/`)
- Create: `zener/components/SS1N5817W/SS1N5817W.zen` (+ sym/fp copiati da `parts/Slkor_1N5817W/`)

- [ ] **Step 1: Tactile (reset/boot), pinout ato: 1,2 = lato A; 3,4 = lato B; 5 = ground/frame**

```bash
mkdir -p zener/components/TactileTSA061B
cp parts/BZCN_TSA061B2808B/*.kicad_sym parts/BZCN_TSA061B2808B/*.kicad_mod zener/components/TactileTSA061B/
```

`TactileTSA061B.zen` (verificare i nomi file copiati e i pin reali col grep):

```zen
A = io(Net)
B = io(Net)

Component(
    name = "TactileTSA061B",
    symbol = Symbol(library = "./TSA061B2808B.kicad_sym"),
    footprint = "./KEY-SMD_4P-L6.0-W6.0-LS8.0.kicad_mod",
    part = Part(mpn = "TSA061B2808B", manufacturer = "BZCN"),
    pins = {"1": A, "2": A, "3": B, "4": B, "5": B},
)
```

- [ ] **Step 2: Schottky 1N5817W (per VBUS→LDO, DNP), pinout ato: 2=anodo, 1=catodo**

```bash
mkdir -p zener/components/SS1N5817W
cp parts/Slkor_1N5817W/*.kicad_sym parts/Slkor_1N5817W/*.kicad_mod zener/components/SS1N5817W/
```

```zen
A = io(Net)
K = io(Net)

Component(
    name = "SS1N5817W",
    symbol = Symbol(library = "./1N5817W.kicad_sym"),
    footprint = "./SOD-123_L2.8-W1.8-LS3.7-RD.kicad_mod",
    part = Part(mpn = "1N5817W", manufacturer = "Slkor"),
    pins = {"1": K, "2": A},
)
```

(I nomi esatti di sym/fp si leggono con `ls parts/BZCN_TSA061B2808B/ parts/Slkor_1N5817W/` e dai trait `is_atomic_part` nei rispettivi `.ato`; adattare.)

- [ ] **Step 3: Valida entrambi e committa**

```bash
cd zener && pcb build components/TactileTSA061B/TactileTSA061B.zen components/SS1N5817W/SS1N5817W.zen
git add zener/components/TactileTSA061B zener/components/SS1N5817W
git commit -m "feat(zener): tactile button and 1N5817W Schottky components"
```

---

### Task 7: Parti nuove — LDO, charger (DNP), JST-PH

Per ognuna: **prima** cercare nel registry (skill `registry-search`: `pcb search -m registry:components <query>`, poi `pcb search -m registry:modules <query>`); usare il package registry se esiste. Solo se assente, creare componente locale (skill `librarian` per il flusso symbol/footprint: scaricare da LCSC/EasyEDA con lo stesso metodo usato per le parti ato, o disegnare da datasheet con `datasheet-reader`).

- [ ] **Step 1: LDO XC6220B331MR-G**

```bash
cd zener && pcb search -m registry:components XC6220
pcb search XC6220B331MR-G
```

Verifica LCSC stock >0 per XC6220B331MR-G (pagina LCSC; vedi Task 8 Step 1 per il metodo). Se stock = 0, fallback nell'ordine: TLV755P-33 (`TLV75533PDBVR`), ME6217C33M5G — stesso giro di ricerca registry + stock. Il componente espone almeno: `VIN = io(Power)`, `VOUT = io(Power)`, `GND = io(Ground)` (+ `CE`/`EN` se presente nel package scelto: legarlo a VIN nel top-level).

- [ ] **Step 2: Charger MCP73831 (o TP4054)**

```bash
pcb search -m registry:components MCP73831
pcb search MCP73831
```

Stock check su `MCP73831T-2ACI/OT`; fallback `TP4054`. Servono: `VDD`, `VBAT`, `STAT`, `PROG`, `VSS`. La R di PROG (0603) fissa la corrente di carica: scegliere `2kohm` (≈500 mA) — verrà montata solo col charger, quindi `dnp=True` insieme a esso.

- [ ] **Step 3: Connettore batteria JST-PH 2 pin**

```bash
pcb search -m registry:components "JST PH"
pcb search S2B-PH-K-S
```

Candidato THT side-entry `S2B-PH-K-S` (JST) o equivalente SMT; vincolo: LCSC stock >0. Se assente nel registry, creare `zener/components/JstPh2/` col flusso librarian.

- [ ] **Step 4: Valida e committa** (file effettivi dipendono da registry vs locale)

```bash
cd zener && pcb sync && pcb build
git add zener && git commit -m "feat(zener): add LDO, charger (DNP) and battery connector parts"
```

---

### Task 8: Verifica sourcing LCSC (stock > 0)

**Files:**
- Create: `zener/SOURCING.md`

- [ ] **Step 1: Per OGNI MPN scelto nei Task 3–7, verifica stock su LCSC**

Metodo: `WebFetch` su `https://www.lcsc.com/search?q=<MPN>` oppure API pubblica `https://wmsc.lcsc.com/ftps/wm/search/global-search?keyword=<MPN>` e leggere il campo stock. In alternativa la pagina prodotto diretta se il codice C è noto (es. `https://www.lcsc.com/product-detail/C400229.html`).

- [ ] **Step 2: Compila `zener/SOURCING.md`**

```markdown
# Sourcing (LCSC) — verificato YYYY-MM-DD

| Ref | Parte | LCSC | Stock | Note |
|---|---|---|---|---|
| U1 | ESP32-C6-WROOM-1-N8 | C5366877 | n/a | già in casa |
| J1 | TYPE-C-31-M-12 | C165948 | <stock> | |
| SW1–6 | CPG135001D01 | C400229 | <stock> | |
| U2 | <LDO scelto> | <C...> | <stock> | |
| U3 | <charger> | <C...> | <stock> | DNP |
| ... | | | | |
```

Regola: se una parte ha stock 0, tornare al task corrispondente e usare il fallback. R/C 0603 generici: verificare che il valore esista su LCSC in 0603 (sono basic part, di norma sì) e annotare un codice C di riferimento.

- [ ] **Step 3: Commit**

```bash
git add zener/SOURCING.md && git commit -m "docs(zener): LCSC sourcing table with stock verification"
```

---

### Task 9: Top-level `C6MacroSwitch.zen` — wiring completo

**Files:**
- Modify: `zener/C6MacroSwitch.zen`

- [ ] **Step 1: Scrivi il top-level completo**

```zen
load("@stdlib/interfaces.zen", "Usb2")

Resistor  = Module("@stdlib/generics/Resistor.zen")
Capacitor = Module("@stdlib/generics/Capacitor.zen")
Led       = Module("@stdlib/generics/Led.zen")
PinHeader = Module("@stdlib/generics/PinHeader.zen")

Esp32   = Module("./components/ESP32C6WROOM1N8/ESP32C6WROOM1N8.zen")
UsbC    = Module("./components/TYPEC31M12/TYPEC31M12.zen")
Choc    = Module("./components/KailhChoc/KailhChoc.zen")
Tactile = Module("./components/TactileTSA061B/TactileTSA061B.zen")
Schottky = Module("./components/SS1N5817W/SS1N5817W.zen")
# + LDO / charger / JST dai package del Task 7 (registry URL o ./components/...)

Board(name="C6MacroSwitch", layout_path="layout/C6MacroSwitch", layers=2)

GND      = Ground("GND")
VBAT     = Power("VBAT")          # dal connettore batteria
VBAT_SW  = Power("VBAT_SW")       # a valle del jumper misura corrente
VDD_3V3  = Power(voltage="3.3V", name="3V3")
VBUS     = Power("VBUS")

# Batteria -> jumper misura corrente -> LDO
# J_BAT (JST-PH): pin 1 = VBAT, pin 2 = GND  (istanza dal package Task 7)
PinHeader(name="J_IMEAS", pins=2, pitch="2.54mm", P1=VBAT, P2=VBAT_SW)

# LDO (XC6220B331): VIN=VBAT_SW, VOUT=VDD_3V3, GND=GND (+ CE a VBAT_SW se presente)
Capacitor(name="C_LDO_IN",  value="1uF",  package="0603", P1=VBAT_SW, P2=GND)
Capacitor(name="C_LDO_OUT", value="1uF",  package="0603", P1=VDD_3V3, P2=GND)
# (valori da datasheet del LDO scelto; XC6220: 1uF in/out ceramici minimi)

# USB-C: dati al C6, VBUS al charger (DNP) e alla Schottky (DNP)
usb_bus = Usb2("USB")
UsbC(name="J_USB", VBUS=VBUS, GND=GND, USB=usb_bus, CC1=Net("CC1"), CC2=Net("CC2"), SHIELD=GND)
Resistor(name="R_CC1", value="5.1kohm", package="0603", P1=Net("CC1"), P2=GND)
Resistor(name="R_CC2", value="5.1kohm", package="0603", P1=Net("CC2"), P2=GND)

# Alimentazione da USB senza batteria (DNP): VBUS -> Schottky -> VBAT_SW
Schottky(name="D_USB_PWR", A=VBUS, K=VBAT_SW, dnp=True)

# Charger (DNP): VDD=VBUS, VBAT=VBAT (a monte del jumper), VSS=GND
# R_PROG 2k (DNP), LED_CHG + R 1k (DNP) su STAT
# (istanze dal package Task 7, tutte con dnp=True)

# ESP32-C6
esp = Esp32(
    name="U1",
    P3V3=VDD_3V3, GND=GND,
    EN=Net("EN"),
    IO0=Net("KEY1"), IO1=Net("KEY2"), IO2=Net("KEY3"),
    IO3=Net("KEY4"), IO4=Net("KEY5"), IO5=Net("KEY6"),
    IO6=Net("VBAT_SENSE"),
    IO8=Net("LED"),
    IO9=Net("BOOT"),
    IO12=usb_bus.DM, IO13=usb_bus.DP,
)
Capacitor(name="C_ESP_BULK", value="10uF",  package="0603", P1=VDD_3V3, P2=GND)
Capacitor(name="C_ESP_DEC",  value="100nF", package="0603", P1=VDD_3V3, P2=GND)

# EN: pull-up + RC + tasto reset
Resistor(name="R_EN", value="10kohm", package="0603", P1=VDD_3V3, P2=Net("EN"))
Capacitor(name="C_EN", value="1uF", package="0603", P1=Net("EN"), P2=GND)
Tactile(name="SW_RST", A=Net("EN"), B=GND)

# Boot: tasto su GPIO9 + pull-up
Resistor(name="R_BOOT", value="10kohm", package="0603", P1=VDD_3V3, P2=Net("BOOT"))
Tactile(name="SW_BOOT", A=Net("BOOT"), B=GND)

# 6 tasti macro Choc: GPIO -> switch -> GND (pull-up interni, LP-GPIO wake)
for i in range(6):
    Choc(name="SW%d" % (i + 1), P1=Net("KEY%d" % (i + 1)), P2=GND)

# LED di stato
Resistor(name="R_LED", value="1kohm", package="0603", P1=Net("LED"), P2=Net("LED_A"))
Led(name="D_LED", color="green", package="0603", A=Net("LED_A"), K=GND)

# Partitore batteria su ADC (GPIO6) — alta impedenza + filtro
Resistor(name="R_VBAT_TOP", value="1Mohm", package="0603", P1=VBAT_SW, P2=Net("VBAT_SENSE"))
Resistor(name="R_VBAT_BOT", value="1Mohm", package="0603", P1=Net("VBAT_SENSE"), P2=GND)
Capacitor(name="C_VBAT_SENSE", value="100nF", package="0603", P1=Net("VBAT_SENSE"), P2=GND)
```

Adattamenti previsti: nomi dei campi `Usb2` (`DP`/`DM`) e firma esatta di `PinHeader`/`Led` da verificare con `pcb doc --package @stdlib`; istanze LDO/charger/JST secondo l'API reale dei package scelti nel Task 7. Il loop `for` Starlark è legale a top-level di modulo; se la versione installata richiede istanze esplicite, srotolare in 6 righe.

- [ ] **Step 2: Sync e build**

```bash
cd zener && pcb sync && pcb build
```

Atteso: zero errori. Iterare sugli errori (pin mismatch, firme generics) finché pulito.

- [ ] **Step 3: Review netlist contro la tabella GPIO della spec**

```bash
pcb build 2>&1 | tail -5   # conferma OK
```

Confronto manuale: KEY1–6→IO0–5, VBAT_SENSE→IO6, LED→IO8, BOOT→IO9, USB DM/DP→IO12/IO13, EN con RC.

- [ ] **Step 4: Commit**

```bash
git add zener && git commit -m "feat(zener): complete C6MacroSwitch top-level wiring"
```

---

### Task 10: Layout e round-trip KiCad 10

- [ ] **Step 1: Genera il layout**

```bash
cd zener && pcb layout
```

Atteso: crea `zener/layout/C6MacroSwitch/*.kicad_pcb` senza errori.

- [ ] **Step 2: Verifica apertura KiCad 10 (headless)**

```bash
kicad-cli pcb export svg zener/layout/C6MacroSwitch/*.kicad_pcb -o /tmp/c6ms_check.svg --layers F.Cu 2>&1 | tail -3
```

Atteso: export senza errori = KiCad 10 legge il file.

- [ ] **Step 3: Round-trip: tocca e risalva con KiCad 10, poi ricompila**

Aprire con `pcb open` (o KiCad GUI), salvare (Ctrl+S, il file viene riscritto in formato KiCad 10), poi:

```bash
cd zener && pcb build && pcb layout
```

Atteso: nessun errore — è il test che con atopile falliva (motivo della migrazione).

- [ ] **Step 4: Commit**

```bash
git add zener/layout && git commit -m "feat(zener): initial generated layout, KiCad 10 round-trip verified"
```

---

### Task 11: Chiusura — README, push, memoria

- [ ] **Step 1: `zener/README.md`** breve: cosa è la board, comandi (`pcb build` / `pcb layout` / `pcb open`), link alla spec, tabella GPIO.

- [ ] **Step 2: Push**

```bash
git push
```

- [ ] **Step 3: Aggiorna la memoria di progetto** (`/home/sf/.claude/projects/-home-sf-src-esp32c6-esp32c6/memory/`): nuova nota `zener-migration` con stato raggiunto e prossimi passi (placement, sbroglio), e link `[[atopile-kicad10-incompatibility]]`.

---

## Fuori piano (fasi successive, non in questo piano)

- Placement e sbroglio finale in KiCad (interattivo con l'utente).
- Forma scheda / meccanica / keycap.
- Ordine JLCPCB/LCSC e firmware Zigbee.
