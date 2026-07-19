# Forslag: Blødere nozzle-kontakt ved homing/probing (Tap)

Dato: 2026-07-19
Problem: Dysen "banker" for hårdt i bedet ved homing.
Årsag: Kontaktkraften styres af homing-hastigheden. Setup kører Tap
(`endstop_pin: probe:z_virtual_endstop`), så dysen rører bedet ved både G28 Z,
z_tilt og bed_mesh.

## Patch 1 — `configs/Z.cfg`, sektion `[stepper_z]`

Selve homing-nedturen. Første berøring sker ved `homing_speed`.

```diff
-homing_speed: 15
-second_homing_speed: 3
+homing_speed: 8          # blødere første berøring (var 15)
+second_homing_speed: 2   # blødere bekræftende touch (var 3)
 homing_retract_dist: 3
```

Bemærk: `homing_retract_dist: 3` beholdes (blødt dobbelt-tap). `probe.cfg`'s
kommentar nævner `0` for Tap; det er valgfrit og ikke årsag til det hårde slag.

## Patch 2 — `configs/probe.cfg`, sektion `[probe]`

Gælder tap i hvert punkt under z_tilt (3 pkt) og bed_mesh (25 pkt).

```diff
-speed: 10.0
+speed: 5.0               # blødere tap pr. probe-punkt (var 10.0)
 lift_speed: 15.0
```

## Effekt

- G28 Z: første berøring 15 → 8 mm/s (~halv kontaktkraft).
- z_tilt/bed_mesh: hver tap 10 → 5 mm/s.
- Ingen ændring i nøjagtighed forventet; Tap er repeterbar ved lavere hastigheder.

## Ikke ændret (bevidst)

- `homing_retract_dist: 3` — behold, medmindre du ønsker enkelt-tap (sæt 0).
- `stealthchop_threshold: 0` — korrekt (spreadcycle giver bedst probe-følsomhed).
- `run_current: 0.65` — uændret; ikke relateet til kontaktkraft.
