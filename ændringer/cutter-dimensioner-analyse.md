# Analyse: Cutter-dimensioner (Crossbow + A4T + WWBMG)

Dato: 2026-07-19
Hardware oplyst af bruger: WWBMG (gate/MMU), Crossbow toolhead-cutter, A4T-toolhead.
Kilde i repo: `mmu/base/mmu_macro_vars.cfg` (`_MMU_CUT_TIP_VARS`) og
`mmu/base/mmu_parameters.cfg`.

## Vigtig forudsætning

Blade-til-nozzle-afstanden og pin-koordinaterne er **byggespecifikke** — de
afhænger af din fysiske Crossbow-montering, extruder-adapter og hvor pin'en
sidder på din ramme. Jeg kan ikke måle din printer eksternt. Nedenfor
dokumenterer jeg de **nuværende værdier**, flager dem der ligner placeholders
eller er indbyrdes inkonsistente, og giver den korrekte kalibreringsmetode og
reference-intervaller for Crossbow/A4T.

## Nuværende konfigurerede dimensioner

| Variabel | Nuværende værdi | Vurdering |
|---|---|---|
| `cutting_axis` | `"y"` | ✅ **Korrekt for A4T** (A4T klipper langs Y mod Ymax) |
| `pin_park_dist` | `-5.0` | ✅ Negativt kræves for A4T — konsistent |
| `blade_pos` (nozzle→klinge) | `54` | ⚠️ Skal kalibreres. Crossbow-builds ligger typisk højere (rapporterede ~58–71). Ingen toolhead-sensor → **kan ikke auto-kalibreres**, må måles manuelt |
| `retract_length` | `35` | ⚠️ Vejledning: ≈ `blade_pos − 5` ≈ **49**. 35 efterlader ~14 mm ekstra filament → mere spild og dybere "nail head". Gennemgå efter blade_pos er sat |
| `pin_loc_xy` | `-5, 5` | ⚠️ Ligner placeholder. Skal matche hvor Crossbow-pin'en fysisk lige rører cutter-armen på din ramme |
| `pin_loc_compressed_xy` | `-5, 19` | ⚠️ Placeholder. Klippe-slag = 19−5 = **14 mm i Y**. Skal måles på maskinen (fuldt komprimeret) |
| `rip_length` | `1.0` | OK som udgangspunkt (frigør armen efter klip) |
| `pushback_length` | `20` | ⚠️ "PTFE-længde + 3 mm" — verificér mod din faktiske PTFE mellem extruder og hotend |
| `cut_fast_move_speed` | `32` | OK typisk startværdi |
| `cut_slow_move_speed` | `8` | OK typisk startværdi |
| `gantry_servo_enabled` | `False` | OK (Crossbow bruger toolhead-pin, ikke gantry-servo) |

Relaterede i `mmu_parameters.cfg`:

| Variabel | Nuværende værdi | Vurdering |
|---|---|---|
| `toolhead_extruder_to_nozzle` | `75` | ⚠️ Byggespecifik. Skal matche A4T + din extruder (NEMA14-pancake iflg. `configs/nh36.cfg`/`autotune_tmc.cfg`). Verificér via CAD eller `MMU_TEST_LOAD` |
| `toolhead_sensor_to_nozzle` | `62` | Ignoreres — **ingen toolhead-sensor monteret** |
| `toolhead_ooze_reduction` | `0` | Trækkes fra effektiv retract. Sæt korrekt før finjustering af `retract_length` |
| toolhead-sensor (`toolhead_switch_pin`) | tom | ❗ Ingen sensor → `blade_pos` og `toolhead_extruder_to_nozzle` skal sættes **manuelt** |

Printerens X-grænse (`configs/XY.cfg`): `position_min: -10`, så `pin_loc` X = `-5`
er indenfor rækkevidde (5 mm inde fra min).

## Konkrete punkter der bør handles på

1. **`blade_pos: 54` skal verificeres/kalibreres.** Uden toolhead-sensor gøres
   det manuelt: mål CAD-afstand fra intern nozzle-spids til klingen på din
   Crossbow + A4T-extruder-adapter, og finjustér med testklip. Bemærk: nogle
   builds skal sætte værdien lavere end den målte, fordi der stadig sker en
   retract før klippet (se Happy Hare issue #163 om nozzle→klinge-afstand).
   Med `force_form_tip_standalone: 1` og sliceren helt slukket burde den
   effekt være mindre, men bekræft med et par testklip.

2. **`pin_loc_xy` / `pin_loc_compressed_xy` er placeholders.** De skal måles på
   din maskine:
   - Jog toolhead til pin'en lige rører cutter-armen → det er `pin_loc_xy`.
   - Jog videre til fuldt komprimeret (med lille margin fra rammen) →
     `pin_loc_compressed_xy`.

3. **`retract_length: 35` vs `blade_pos: 54`.** Vejledningen anbefaler ≈ 49.
   Juster først efter `blade_pos` er fastlagt, og sæt `toolhead_ooze_reduction`
   korrekt inden.

4. **`toolhead_extruder_to_nozzle: 75`** bør bekræftes mod A4T + din extruder.

## Hvad jeg IKKE kan gøre uden dig

De endelige tal for `blade_pos` og pin-positionerne kræver måling på din
fysiske printer (eller Crossbow-justeringsjiggen). Giver du mig:
- den målte nozzle→klinge-afstand,
- pin-positionerne aflæst fra jog,
- extruder-model + PTFE-længde,

…så udfylder jeg de nøjagtige værdier og leverer en færdig config-patch her i
`ændringer/`.

## Kilder

- Crossbow Filament Cutter (DW-Tas): https://github.com/DW-Tas/Crossbow-Filament-Cutter
- A4T-toolhead (Armchair Heavy Industries): https://github.com/Armchair-Heavy-Industries/A4T
- A4T-konfigurator: https://a4t.dwtas.net/
- Happy Hare – nozzle→klinge-afstand (issue #163): https://github.com/moggieuk/Happy-Hare/issues/163
- Happy Hare – Blobbing and Stringing (wiki): https://github.com/moggieuk/Happy-Hare/wiki/Blobbing-and-Stringing
