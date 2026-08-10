# Ændringer — drift-baseret heat soak + 11×11 bed mesh

Dette er et backup-repo, så filerne her er **forslag klar til at kopiere over**, ikke
noget der er aktivt på printeren. Mappestrukturen spejler `printer_data/config/`,
så du kan kopiere direkte oveni.

## Hvad der er ændret

| Fil | Status | Ændring |
|---|---|---|
| `macros/heat_soak.cfg` | **ny** | Selve drift-målingen: variable, målecyklus, probe og vurdering |
| `macros/print_start.cfg` | ændret | Bruger den nye soak i stedet for de faste 10 minutter |
| `configs/probe.cfg` | ændret | `probe_count: 5,5` → `11,11`, plus kortere probe-bevægelser |
| `configs/Bedfans.cfg` | ændret | Ny `BEDFAN_SOAK_MODE` — låser blæserhastigheden under soaken |

`aendringer.patch` indeholder de samme ændringer som unified diff, hvis du hellere
vil læse dem sådan.

## Installation

```bash
cd ~/printer_data/config
cp <repo>/ændringer/printer_data/config/macros/heat_soak.cfg   macros/
cp <repo>/ændringer/printer_data/config/macros/print_start.cfg macros/
cp <repo>/ændringer/printer_data/config/configs/probe.cfg      configs/
cp <repo>/ændringer/printer_data/config/configs/Bedfans.cfg    configs/
```

`printer.cfg` skal **ikke** røres — `[include macros/*.cfg]` samler `heat_soak.cfg`
op automatisk. Derefter `FIRMWARE_RESTART`.

Alle Jinja-templates er render-testet mod Klippers egne delimiters (`{%%}` / `{}`)
med mock-data for både MMU-print, SOAK=0 og slicer-overrides.

---

## 1. Heat soak der måler i stedet for at tælle ned

### Princippet

Når bed'et har nået sin temperatur og dysen står på 150 °C, probes bed-centrum
(175, 175) én gang som **baseline**. Derefter probes det samme punkt hvert 5. minut.
Forskellen mellem to på hinanden følgende målinger *er* den termiske drift i ramme,
gantry og bed.

Der homes **aldrig** Z undervejs. Alle målinger deler derfor samme nulpunkt, og
deltaet er ren drift — ikke homing-støj.

### Hvornår er soaken færdig

Alle fire skal være opfyldt samtidig:

- `|Δz| ≤ 0.0075 mm` — samme tolerance som din `z_tilt retry_tolerance`
- `|Δkammer| ≤ 0.3 °C` pr. interval — kammeret må heller ikke stadig stige
- kammeret er på eller over sit mål
- ...og det skal holde i **2 cyklusser i træk**, minimum 2 cyklusser i alt (10 min)

**Hårdt loft: 12 cyklusser = 60 minutter.** Derefter printes der alligevel, med en
advarsel i konsollen. Det er bevidst: den gamle `TEMPERATURE_WAIT` på kammeret havde
ingen timeout, så et koldt værksted eller en åben låge kunne hænge printet i det
uendelige.

### Sådan ser det ud i konsollen

```
Heat soak: baseline z=2.1234 mm, kammer 31.2c (maal 45c). Maaler hvert 5. minut,
stopper naar driften er under 0.0075 mm i 2 cyklusser i traek.
Heat soak 1 (5 min):  z=2.0981 dz=-0.0253 mm [drift] | kammer 38.4c d=+7.2c [stiger] under maal | stabil 0/2
Heat soak 2 (10 min): z=2.0870 dz=-0.0111 mm [drift] | kammer 43.1c d=+4.7c [stiger] under maal | stabil 0/2
Heat soak 3 (15 min): z=2.0829 dz=-0.0041 mm [ok]    | kammer 45.6c d=+2.5c [stiger] | stabil 0/2
Heat soak 4 (20 min): z=2.0815 dz=-0.0014 mm [ok]    | kammer 46.3c d=+0.7c [stiger] | stabil 0/2
Heat soak 5 (25 min): z=2.0809 dz=-0.0006 mm [ok]    | kammer 46.5c d=+0.2c [ok] | stabil 1/2
Heat soak 6 (30 min): z=2.0806 dz=-0.0003 mm [ok]    | kammer 46.6c d=+0.1c [ok] | stabil 2/2
Heat soak faerdig efter 30 min - termisk drift er stoppet.
```

### Dysen står på 150 °C hele soaken

Det er med vilje. Dysen er selv proben (Tap-stil, `x/y_offset: 0`), så dens
varmeudvidelse går direkte ind i målingen. Holdes den på samme temperatur ved hver
måling, er deltaet ramme-drift og ikke dyse-drift.

Bivirkning: ASA/ABS sidder i hotenden ved 150 °C i op til en time. Det er et godt
stykke under flydepunktet, så ooze er minimal — men det er værd at kende til.
`M104 S150` er rykket op *før* `M190`, så dysen varmer op sideløbende med bed'et i
stedet for bagefter.

### Justering

Standardværdierne står i `_HEATSOAK_VARS` øverst i `heat_soak.cfg`. Alt kan også
overrules pr. print fra sliceren:

| Parameter | Default | Betydning |
|---|---|---|
| `SOAK=0` | 1 | Slår drift-målingen fra, bruger fast tid i stedet |
| `SOAK_MINUTES` | 5 | Minutter ved `SOAK=0` |
| `SOAK_INTERVAL` | 300 | Sekunder mellem målinger |
| `SOAK_DRIFT` | 0.0075 | mm drift der regnes som "stoppet" |
| `SOAK_CHAMBER_DRIFT` | 0.3 | °C kammerdrift pr. interval |
| `SOAK_MIN` | 2 | Minimum antal cyklusser |
| `SOAK_STABLE` | 2 | Stabile cyklusser i træk |
| `SOAK_MAX` | 12 | Hårdt loft på antal cyklusser |

Eksempel i sliceren for et hurtigt PLA-print: `PRINT_START BED=60 EXTRUDER=215 CHAMBER=0 SOAK=0`

Slicer-overrides skrives til `_HEATSOAK_STATE` og rører ikke dine defaults i
`_HEATSOAK_VARS`, så de gælder kun det ene print.

Der er også en manuel `HEAT_SOAK BED=110 CHAMBER=45` til at varme maskinen op uden
at starte et print.

### Hvorfor koden ser ud som den gør

To ting i Klipper styrer strukturen:

1. **Der findes ingen `while`-løkke.** `PRINT_START` kalder derfor `_HEATSOAK_STEP` i
   en `{% for %}`-løkke op til `SOAK_MAX` gange. Hvert kald er en frisk render, så det
   kan læse `done`-flaget og springe over med det samme, når soaken er færdig. De
   overskydende kald koster ingenting.
2. **En makros template renderes *før* dens kommandoer udføres.** Derfor kan
   `printer.probe.last_z_result` ikke læses i den samme makro, som kalder `PROBE` —
   man ville få resultatet fra forrige måling. Målingen ligger i `_HEATSOAK_PROBE`
   og vurderingen i `_HEATSOAK_EVAL`, som først renderes, når proben rent faktisk er kørt.

Ventetiden deles i 1-minuts `G4`-bidder, dels for nedtællingen på displayet, dels
fordi det holder lookahead-køen kort — samme mønster som din gamle makro.

`PROBE` kaldes med `SAMPLES_TOLERANCE=0.02` i stedet for de 0.006 fra `probe.cfg`.
Tidligt i soaken bevæger bed'et sig måleligt mellem to samples, og en
tolerance-fejl der ville afbryde printet. Median af 3 er stadig rigeligt præcist til
at måle drift på 0.0075 mm-niveau. Din normale `probe.cfg`-tolerance gælder uændret
for `BED_MESH_CALIBRATE` og `Z_TILT_ADJUST`.

---

## 2. Bed mesh: 11×11, stadig adaptivt

`probe_count: 5,5` → `11,11`. 121 punkter over 10–320 mm giver 31 mm punktafstand.
`ADAPTIVE=1` er uændret i `PRINT_START`. Klipper skalerer punktantallet ned efter
printets areal og bevarer punkttætheden, så små prints ikke koster 121 punkter.

`bicubic` fungerer fint med 11 punkter.

## 3. Kortere probe-bevægelser

Nedturen mod bed'et er **uændret** på 5,0 mm/s. Det er kun de tomme bevægelser der er
skåret ned:

| Parameter | Før | Nu |
|---|---|---|
| `sample_retract_dist` | 3,0 mm | **1,0 mm** |
| `[bed_mesh] horizontal_move_z` | 5 mm | **2 mm** |

`sample_retract_dist` koster dobbelt: den bestemmer både hvor langt der løftes ved
15 mm/s **og** hvor langt der køres ned igen ved 5 mm/s. Nedturen er den dyre. Med
`samples: 3` og `drop_first_result: True` er der 4 nedture pr. punkt, altså 3 gentagne
løft-og-ned-cyklusser der hver sparer godt en halv sekund.

| Pr. mesh-punkt | Før | Nu |
|---|---|---|
| Nedtur til første sample | 1,00 s | 0,40 s |
| 3 × (løft + nedtur) | 2,40 s | 0,80 s |
| Afsluttende løft | 0,40 s | 0,13 s |
| XY-flyt (31 mm) | 0,18 s | 0,18 s |
| **I alt** | **~4,0 s** | **~1,5 s** |

Med overhead pr. probe lander en fuld 11×11-plade på **omkring 4 minutter i stedet for
8–9**. To tredjedele af besparelsen kommer fra retract-distancen, en tredjedel fra
travel-højden. `Z_TILT_ADJUST` og heat soak-målingerne bliver også lidt hurtigere.

### Grænserne for de to tal

**1,0 mm retract:** den bindende grænse er ikke clearance til pladen, men at Tap'ens
flexure skal nå at **de-trigge** før næste nedtur. Optisk Tap udløser typisk 0,2–0,5 mm
inde i vandringen, og decelerationen ved 5 mm/s med `max_z_accel: 350` lægger kun
0,036 mm oveni — så 1,0 mm er cirka 2× margin. Marginen afhænger dog af, hvor din
optiske flag sidder, og fejlmoden er hård: `Probe triggered prior to movement` afbryder
mesh'et og dermed printet.

Verificér før du printer noget vigtigt:

```
PROBE_ACCURACY SAMPLES=10 SAMPLE_RETRACT_DIST=1
```

Kører den 10 samples igennem uden fejl, og er spredningen på niveau med den samme test
ved `SAMPLE_RETRACT_DIST=3`, er 1,0 mm bevist på netop din maskine. Ser du fejlen, så
sæt `sample_retract_dist: 1.5` — det koster kun omkring 35 sekunder på en fuld plade.

**2 mm travel-højde:** mesh'et kører først efter `Z_TILT_ADJUST` og `G28 Z`, så
gantryet er rettet ind og nulpunktet friskt. En Voron 350-plade varierer typisk
0,2–0,5 mm over 10–320, så der er halvanden millimeter reel luft tilbage.
Blobifier-bakken (X=10, Y=350) og børsten (X=50–80, Y=350) ligger begge uden for
mesh-området, der stopper ved Y=320.

To ting jeg **ikke** har rørt:

- `[z_tilt] horizontal_move_z: 5` er død config — `z-tilt.cfg` sætter selv
  `HORIZONTAL_MOVE_Z=10` på pass 1 og `=2` på pass 2, så konfigurationsværdien bruges
  aldrig.
- `probe_z: 5` i `_HEATSOAK_VARS` står stadig på 5 mm. Der er én måling hvert 5. minut,
  så de 0,6 sparede sekunder er ligegyldige, og den ekstra clearance er rar mens
  pladen stadig bevæger sig.

---

## 4. Bedfans under soaken

Ny makro `BEDFAN_SOAK_MODE` i `Bedfans.cfg`, som kaldes af `_HEATSOAK_BEGIN`.

Din bedfan-statemaskine går slow → (60 s) → ramp → fast efter at bed'et har ramt
temperaturen. Det skift ville lande midt i måleserien og flytte rundt på varmen i
kammeret — og dukke op som "drift" i målingen. `BEDFAN_SOAK_MODE` springer
ramp-forsinkelsen over og sætter `ramp_stage` direkte til 2, så hastigheden ligger
fast fra og med baseline-målingen. I praksis betyder det, at blæserne når fuld fart
60 sekunder tidligere end før.

Chamber-beskyttelsen (`chamber_limit` 60 °C / `chamber_resume` 50 °C) er bevidst
**ikke** sat ud af kraft — den skal stadig kunne skrue ned, hvis kammeret bliver for
varmt. Sker det midt i soaken, tæller den cyklus bare ikke som stabil.

---

## Ting jeg fandt undervejs

**Rettet:** `target_chamber_wait` blev beregnet i linje 12 af den gamle
`print_start.cfg`, men brugt aldrig — der blev ventet på `target_chamber` i stedet.
`CHAMBER_MINIMAL` fra sliceren var altså død kode. Den bruges nu som kammermål i
soaken.

**Rettet:** `TEMPERATURE_WAIT` på kammeret havde ingen timeout. Erstattet af soakens
hårde loft.

**Ikke rørt, men værd at vide:** dit carbon-filter står på 208.870 sekunder ≈ 58 timer
af de 70, `_AIR_FILTER_VARIABLES` er sat til. `QUERY_AIR_FILTER` viser status,
`RESET_AIR_FILTER` nulstiller efter skift.
