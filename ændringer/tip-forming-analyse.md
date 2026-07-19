# Analyse: Hvorfor prøver den stadig at lave "tip forming"?

Dato: 2026-07-19
Repo: TridentR2 (Klipper + Happy Hare MMU)
Bemærk: Den døde mappe `mmu123123/` er bevidst holdt UDE af denne analyse.

## Kort svar

**Happy Hare selv former ikke en tip — den klipper.** Den aktive config er sat op
til toolhead-klip (`_MMU_CUT_TIP`) med tip forming slået fra
(`simple_tip_forming: False`). Ser du stadig tip forming, kommer det derfor med
stor sandsynlighed fra **sliceren**, hvis ramming/tip-shaping stadig er tændt.

## Dokumentation af fund (kun aktiv config: `mmu/base/`)

### Tip forming-parametre (`mmu/base/mmu_parameters.cfg`)

```
force_form_tip_standalone: 1     # linje 313 — Happy Hare håndterer det altid standalone (sluk sliceren!)
form_tip_macro: _MMU_CUT_TIP     # linje 314 — kalder KLIPPE-makroen, ikke forme-makroen
```

### Klippe-makroens variabler (`mmu/base/mmu_macro_vars.cfg`, `_MMU_CUT_TIP_VARS`)

```
variable_simple_tip_forming : False   # linje 291 — INGEN tip forming udføres i klippet
variable_blade_pos          : 54      # toolhead-cutter (klinge 54 mm fra nozzle-spids)
variable_pin_loc_xy         : -5, 5   # depressor-pin position (Filametrix-agtig toolhead-cutter)
```

### Konklusion om den aktive opsætning

- `form_tip_macro` = `_MMU_CUT_TIP` → **klip**, ikke form.
- `simple_tip_forming` = `False` → klippet indeholder **ikke** et tip forming-trin.
- `force_form_tip_standalone` = `1` → Happy Hare gør det altid selv, og sliceren
  SKAL være slået fra ("TURN SLICER OFF!").

Den eneste måde, den *aktive* Happy Hare-config ville forme en tip på, er hvis
`simple_tip_forming` var `True` — og det er den ikke. Altså er det ikke Happy Hare.

## Mest sandsynlige årsag: sliceren

`force_form_tip_standalone: 1` betyder, at slicerens tip forming skal være slået
fra. Er den ikke det, laver sliceren ramming/tip-shaping under print, oveni Happy
Hares klip. Tjek i slicerens filamentprofil:

- **Filament ramming / "tip shaping"** → slå FRA.
- Evt. wipe/purge-relaterede indstillinger, hvis de også kører uventet.

## Hvis du i stedet vil BEHOLDE tip forming

Så peg `form_tip_macro` på `_MMU_FORM_TIP` (linje 314) — men det giver kun mening,
hvis du ikke bruger toolhead-cutter. Bekræft venligst hvilken metode du kører.

## Hvad jeg mangler fra dig

1. Hvor ser du tip forming ske — i konsollens log, eller som fysisk bevægelse på
   printeren?
2. Bruger du toolhead-cutter (nuværende opsætning) eller vil du helt væk fra klip?

Sig til, så laver jeg selve config-ændringen (den lægges her i `ændringer/` som
aftalt, medmindre du beder mig redigere kildefilerne direkte).
