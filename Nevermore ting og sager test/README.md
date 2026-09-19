# Nevermore Mini – integration i PRINT_START/PRINT_END

Noter fra gennemgang af repoet og https://github.com/SanaaHamel/nevermore-controller.
De faktiske rettelser er lavet direkte i de rigtige config-filer (se nedenfor) – denne
mappe er kun til noter/valgfrie ekstra filer, som instrueret.

## Fejl der blev fundet og rettet

1. **`SET_FAN_SPEED FAN=NeverMore` fandtes ikke.**
   `printer_data/config/configs/Nevermore` (uden `.cfg`-endelse, altså ikke inkluderet
   af `[include configs/*cfg]`) er et gammelt, dødt setup fra "KlipperMacros"
   air-filter-timer, som definerede en `[fan_generic NeverMore]`. Den fil er ikke
   aktiv længere – I bruger den rigtige `[nevermore]`-controller
   (`configs/NevermoreMini.cfg`), som selv registrerer en fan ved navn
   **`nevermore_fan`** (mønster: `<sektionsnavn>_fan`).
   Fordi `print_start.cfg` kaldte `SET_FAN_SPEED FAN=NeverMore` på et fan-navn der
   ikke findes, fejlede **PRINT_START med en Klipper-fejl for enhver print med
   bed > 90°C** (dvs. ABS/ASA/PC-heatsoak-grenen kørte aldrig færdig).
   → Rettet til `FAN=nevermore_fan` i `macros/print_start.cfg`.

2. **`UPDATE_DELAYED_GCODE ID=stop_nevermore` i `print_end.cfg` pegede også på et
   objekt der ikke findes** (samme døde fil som ovenfor definerede
   `[delayed_gcode stop_nevermore]`). Det gav en fejl i slutningen af hver print.
   → Fjernet. Det var faktisk slet ikke nødvendigt: Nevermore-modulet går
   automatisk ud af "print mode" (og dermed tilbage til automatisk VOC-styret
   blæserhastighed + genoptager VOC-kalibrering) så snart extruderens
   måltemperatur går i 0 – hvilket `TURN_OFF_HEATERS` lige har gjort et par
   linjer forinden. Ingen manuel fan-off-timer nødvendig.

## Hvad "print mode" på controlleren gør automatisk

Fra dokumentationen: Nevermore-modulet går selv ind i "print mode", når en
extruder har en måltemperatur > 0. I print mode:
- Blæseren tvinges til 100%.
- VOC-kalibrering sættes på pause (så den ikke fejltolker lave emissioner som drift).

Når extruderen går tilbage til 0° (PRINT_END), går den automatisk tilbage til
automatisk/VOC-styret blæserhastighed og genoptager kalibrering.

**Konsekvens:** Så snart jeres `M104 S150` (probe-safe preheat) kører i
PRINT_START, tvinger controlleren *selv* blæseren til 100% – det er derfor der
kun var brug for en manuel `SET_FAN_SPEED ... SPEED=1` i heatsoak-fasen
*før* hotenden begynder at varme (der er extruder-target stadig 0, så
print mode er ikke aktiveret endnu, og uden manuelt override ville blæseren
kun køre i "automatic"/VOC-tilstand mens kammeret varmes op).

## Ny funktion: `NEVERMORE_TEMPERATURE_WAIT` (det I bad om)

Tilføjet lige efter jeres eksisterende
`TEMPERATURE_WAIT SENSOR="temperature_sensor chamber" MINIMUM=...` i
`macros/print_start.cfg`:

```
NEVERMORE_TEMPERATURE_WAIT MINIMUM={target_chamber_wait} UNAVAILABLE_TIMEOUT=600
```

- Bruger Nevermorens egen intake-temperatursensor (med fallback til exhaust)
  som et ekstra tjek oveni jeres rigtige kammertermistor (`temperature_sensor
  chamber`). I har allerede en fysisk kammersensor, så den er stadig
  "sandheden" – Nevermore-waitet er et sikkerhedsnet/dobbelttjek.
- `UNAVAILABLE_TIMEOUT=600` (10 min) forhindrer at PRINT_START hænger for
  evigt, hvis controlleren mister BT/USB-forbindelsen.

## Anden ændring: `print_mode_min_bed_target`

Tilføjet i `configs/NevermoreMini.cfg`:

```
print_mode_min_bed_target: 40
```

Uden denne går controlleren i "print mode" (100% fan + kalibrering sat på
pause) hver gang I varmer hotenden op til noget som helst – fx ved
dyseskift, PID-tuning eller probe-kalibrering, hvor I ikke rent faktisk
printer. Med denne skal bed-target også være ≥ 40°C, før print mode
aktiveres, så den slags værkstedsopgaver ikke udløser fuld fart/pauset
kalibrering unødigt.

## Andre idéer (ikke implementeret – kun forslag)

- **VOC gating pr. filament**: `NEVERMORE_VOC_GATING_THRESHOLD_OVERRIDE
  THRESHOLD=<175-500>` kan sættes fra slicerens start-gcode pr. filamenttype
  for filamenter der normalt lugter meget (fx nogle ASA/PC-blends), så
  controlleren ikke fejltolker den forhøjede VOC som "beskidt luft, drop
  automatisk regulering".
- **Statustjek ved print-start**: `NEVERMORE_STATUS` kan give et hurtigt
  "er controlleren overhovedet forbundet" i konsollen. Kunne udbygges til at
  RESPOND en advarsel, hvis controlleren er offline, inden et ABS-print
  startes uden filtrering – ville kræve at man slår det korrekte printer-objekt-
  felt for forbindelsesstatus op i Nevermore-modulets kildekode først, så jeg
  har bevidst ikke gættet mig til det i de rigtige macro-filer.
- **Filterlevetids-påmindelse**: Den gamle, døde `configs/Nevermore`-fil havde
  faktisk en fin idé (tæller timer blæseren har kørt, og påminder om
  filterskift). Den byggede bare på det forkerte/døde fan-objekt. Der ligger
  en opdateret, brugsklar udgave her i mappen:
  `filter_timer_optional.cfg` – peger på det rigtige `nevermore_fan`-objekt.
  Den er **ikke** inkluderet i printer.cfg (jeres regel om kun at gemme nyt
  her respekteres) – flyt filen til `printer_data/config/configs/` og tilføj
  fx `RESET_AIR_FILTER`/`QUERY_AIR_FILTER` til jeres macroer, hvis I vil bruge
  den.
- **`fan_policy_cooldown`**: styrer hvor længe (standard 900 sek) Nevermoren
  bliver ved med at filtrere efter automatik-policyen ellers ville stoppe.
  Værd at kigge på hvis I vil have mere/mindre eftertræk efter et print (den
  overtager nu selv styringen efter PRINT_END, jf. rettelse #2 ovenfor).

## Filer rørt af den faktiske integration (uden for denne mappe)

- `printer_data/config/macros/print_start.cfg`
- `printer_data/config/macros/print_end.cfg`
- `printer_data/config/configs/NevermoreMini.cfg`
