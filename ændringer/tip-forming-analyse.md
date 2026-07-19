# Analyse: Hvorfor prøver den stadig at lave "tip forming"?

Dato: 2026-07-19
Repo: TridentR2 (Klipper + Happy Hare MMU)

## Kort svar

Der er **to årsager**, der begge kan forklare det:

1. **Du redigerer sandsynligvis den forkerte mappe.** Printeren læser KUN fra
   `printer_data/config/mmu/`. Mappen `printer_data/config/mmu123123/` (og filen
   `mmu_vars.cfgasdasd`) er **døde kopier, der ikke inkluderes af noget**.
   Ændringer lavet der har ingen effekt — det er den klassiske grund til at noget
   "stadig" sker.

2. **Slicerens tip forming / ramming er sandsynligvis stadig tændt.** Den aktive
   config tvinger standalone-håndtering (`force_form_tip_standalone: 1`), som
   udtrykkeligt kræver at sliceren slås fra ("TURN SLICER OFF!"). Er slicerens
   ramming/tip-shaping stadig aktiv i filament-/printerprofilen, laver den tip
   forming under print — uafhængigt af Happy Hare.

## Dokumentation af fund

### Hvad er faktisk aktivt?

`printer_data/config/printer.cfg` inkluderer:

```
[include mmu/base/*.cfg]
[include mmu/optional/client_macros.cfg]
...
[include MMU.cfg]        # <-- denne fil er TOM
```

Søgning efter `mmu123123` i alle konfigurationsfiler: **ingen referencer**.
→ `mmu123123/` er en død mappe. `MMU.cfg` i roden er tom.

### Aktiv tip forming-opsætning (`mmu/base/mmu_parameters.cfg`)

```
force_form_tip_standalone: 1        # linje 313 — altid standalone (sluk sliceren!)
form_tip_macro: _MMU_CUT_TIP        # linje 314 — kalder CUT-makroen (toolhead-cutter)
extruder_form_tip_current: 100
slicer_tip_park_pos: 0
```

### Aktiv makro-variabel (`mmu/base/mmu_macro_vars.cfg`)

```
variable_simple_tip_forming : False   # linje 291 — springer tip forming over i cut-makroen
```

**Vigtig konklusion:** I den *aktive* config former Happy Hare ikke selv en tip.
`form_tip_macro` peger på `_MMU_CUT_TIP` (klip), og `simple_tip_forming` er
`False`. Happy Hare *klipper* altså — den *former* ikke. Ser du stadig tip
forming-*bevægelse*, kommer den derfor med stor sandsynlighed fra sliceren.

### Forskel mellem aktiv mappe og den døde kopi

| Indstilling                   | `mmu/` (AKTIV)   | `mmu123123/` (DØD) |
|-------------------------------|------------------|--------------------|
| `form_tip_macro`              | `_MMU_CUT_TIP`   | `_MMU_FORM_TIP`    |
| `variable_simple_tip_forming` | `False`          | `True`             |

Bemærk: den døde mappe er sat op til at *forme* tip. Hvis du har lavet dine
seneste ændringer der, har de aldrig ramt printeren.

## Foreslåede handlinger (endnu ikke udført)

Vælg alt efter hvad du faktisk vil opnå:

### A) Du vil helt af med tip forming (fordi du klipper ved toolhead)
Den aktive config er allerede korrekt til dette. Kontrollér i stedet **sliceren**:
- Sluk "Filament ramming" / "tip shaping" i filamentprofilen.
- Sluk evt. wipe tower hvis du ikke bruger den.

### B) Du bruger EREC-cutter (klip ved MMU, ikke toolhead)
Så skal det aktive `form_tip_macro` ifølge Happy Hares egen note ændres til
`_MMU_FORM_TIP` (form en pæn tip før udtræk, klip så ved MMU efter unload), og
EREC-addon skal aktiveres. Bekræft venligst hvilken cutter du har, før dette
ændres.

### C) Ryd op i de døde filer (anbefales uanset)
Fjern det, der kun skaber forvirring:
- `printer_data/config/mmu123123/` (hele mappen)
- `printer_data/config/mmu123123/mmu_vars.cfgasdasd`
- Evt. den tomme `printer_data/config/MMU.cfg`

Dette er destruktive sletninger — bekræft venligst, før jeg udfører dem.

## Hvad jeg mangler fra dig for at gå videre

1. Hvilken cutter/tip-metode vil du reelt bruge? (toolhead-klip / EREC / ingen)
2. Hvor ser du tip forming ske — i konsollens log, eller som fysisk bevægelse?
3. Må jeg slette de døde filer (afsnit C)?
