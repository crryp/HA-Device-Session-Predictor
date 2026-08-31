# Apparaat Sessie Voorspeller

Schat hoe lang een "dom" apparaat (wasmachine, vaatwasser, droger, ...) nog
moet draaien — puur op basis van een vermogenssensor (Watt). Geen slimme
aansluiting op het apparaat zelf nodig, alleen een energiemeter/slimme
stekker die het huidige vermogen rapporteert.

Gebouwd en getest op een wasmachine met een HomeWizard Energy Socket, maar
werkt met elke sensor die een vermogenswaarde in Watt teruggeeft.

## Hoe het werkt

1. **Sessiedetectie**: vermogen boven een drempel = "aan", onder een andere
   drempel (lager) = "uit". Beide met een minimale aaneengesloten duur, zodat
   korte pieken/dalen tussen programmafases niet per ongeluk als start/einde
   worden gezien.
2. **Checkpoints tijdens de sessie**: cumulatief energieverbruik wordt op
   twee snelheden bijgehouden:
   - **Fijn**: elke 10 minuten, tot 6x (dekt het eerste uur) — vangt meestal
     de meest onderscheidende fase van een programma (bv. de opwarmfase).
   - **Grof**: elke 20 minuten, tot 9x (dekt tot 3 uur) — nodig om lange en
     middellange programma's uit elkaar te houden, iets waar de fijne curve
     alleen niet goed in is (die stopt na het eerste uur).
3. **Bij sessie-einde** wordt de duur, het energieverbruik en de curve
   opgeslagen in een kleine geschiedenis (standaard: laatste 6 curves, laatste
   20 duren/verbruikswaarden).
4. **Tijdens een nieuwe sessie** vergelijkt de sensor de curve-tot-nu-toe met
   alle opgeslagen curves, en gebruikt de dichtstbijzijnde match om de
   resterende tijd te schatten. Loopt de sessie langer door dan de beste
   match voorspelde (met 5 minuten coulance)? Dan valt de schatting terug op
   de een-na-beste match.
5. **Duplicaat-detectie**: een sessie die qua duur (±5 min) én energieverbruik
   (±10%, of ±0,05 kWh absoluut) sterk lijkt op een al opgeslagen sessie,
   wordt niet als nieuwe curve opgeslagen — zo raken de beperkte 6 curve-
   plekken niet gevuld met 6x hetzelfde programma. Bij een duplicaat wordt de
   bestaande curve alleen vervangen als de nieuwe curve *vollediger* is
   (meer checkpoints t.o.v. wat op basis van de duur verwacht mocht worden).

## Bekende beperkingen

- **Eerste paar sessies zijn onnauwkeurig.** Zonder geschiedenis valt de
  schatting terug op een vaste 90 minuten. Pas na een paar sessies met wat
  variatie wordt de curve-matching zinvol.
- **255-tekens-limiet.** De curve-opslag gebruikt `input_text`-helpers
  (HA-limiet: 255 tekens). Dat begrenst hoeveel checkpoints × hoeveel
  sessies je kunt bewaren. De standaardinstellingen (6 fijn + 9 grof, laatste
  6 sessies) passen ruim, maar ga je dit uitbreiden, houd de limiet in de
  gaten.
- **Programma's zonder duidelijke opwarmfase** (bijv. een koud programma)
  kunnen minder goed te onderscheiden zijn, omdat een groot deel van het
  onderscheidend vermogen van de fijne curve juist uit die opwarm-piek komt.
- **De juiste drempelwaarden vind je zelf.** Er is geen universele
  start/eind-drempel die voor elk apparaat werkt — meet uit wat bij jouw
  apparaat een betrouwbaar signaal is (zie installatie-stap 3 hieronder).

## Bestanden in deze repo

| Bestand | Wat |
|---|---|
| `packages/apparaat_sessie_voorspeller.yaml` | **Verplicht.** De kern: sessiedetectie, checkpoints, curve-matching, resterende-tijd-sensor. |
| `packages/apparaat_klaar_melding.yaml` | **Optioneel.** Push-melding + lamp-signaal (met automatisch herstel van de vorige lamp-staat) zodra een sessie is afgerond. |
| `dashboard-card.yaml` | Voorbeeld van een Mushroom-kaart die het vermogen + de resterende tijd toont. |

---

## Installatie — optie A: als package (aanbevolen)

Een *package* is één YAML-bestand waarin helpers, automations en sensoren
samen staan, in plaats van los via de UI aangemaakt te worden. Dat maakt dit
project "kopieer, vul een paar regels in, herstart" in plaats van tientallen
losse UI-stappen.

### Stap 1 — Zorg dat Home Assistant packages laadt

Check of je `configuration.yaml` dit al bevat (vaak standaard aanwezig):

```yaml
homeassistant:
  packages: !include_dir_named packages
```

Zo niet, voeg het toe. Maak (als die nog niet bestaat) een map `packages/` aan
naast je `configuration.yaml`.

### Stap 2 — Kopieer het hoofdbestand

Zet `packages/apparaat_sessie_voorspeller.yaml` in jouw `config/packages/`-map.

### Stap 3 — Vul de placeholders in

Open het bestand en zoek naar `VUL_HIER_IN` en `AANPASSEN` — dit moet je in
ieder geval doen:

1. **Je vermogenssensor**: vervang elke `sensor.VUL_HIER_IN_JE_VERMOGENSENSOR`
   door de entity_id van jouw sensor die het huidige vermogen in Watt geeft.
2. **Start-drempel**: de `above:`-waarde in de eerste automation — hoeveel
   Watt duidt betrouwbaar op "er draait een programma" (niet op standby of
   menu-navigatie)?
3. **Start-duur** (`for:`): hoe lang moet dat aaneengesloten aanhouden voor
   je het als een echte start vertrouwt?
4. **Eind-drempel**: de `below:`-waarde in de tweede automation — idealiter
   zo dicht mogelijk bij het werkelijke "stil" niveau van je apparaat. Hoe
   dichter bij het echte nulpunt, hoe kleiner de kans dat een pauze tussen
   programmafases onterecht als "klaar" wordt gezien.
5. **Eind-duur** (`for:`): hoe lang moet dat aaneengesloten aanhouden?

**Tip om de drempels te vinden**: bekijk een tijdje de geschiedenisgrafiek
van je vermogenssensor tijdens een volledig programma. Let op:
- Hoe hoog het vermogen tijdens de "echt actieve" fases minimaal is (dat
  wordt je start-drempel).
- Hoe laag het vermogen tussen fases in kan zakken zonder dat het apparaat
  klaar is (je eind-drempel moet daaronder liggen, en/of de eind-duur moet
  langer zijn dan de langste "gewone" pauze).

### Stap 4 — Herstart Home Assistant

Packages worden het betrouwbaarst geladen met een volledige herstart
(Instellingen → Systeem → Herstarten). "YAML herladen" werkt soms ook, maar
bij een nieuwe package is een herstart veiliger.

### Stap 5 — Controleer

Ga naar Instellingen → Apparaten & diensten → Entiteiten, zoek op "Apparaat"
— je zou alle helpers, de energiesensor en de resterende-tijd-sensor moeten
zien. Ga naar Instellingen → Automatiseringen en controleer dat de twee
nieuwe automations er staan en ingeschakeld zijn.

### Stap 6 (optioneel) — Klaar-melding + lamp

Wil je ook een push-melding (en optioneel een lamp-signaal) zodra een sessie
klaar is? Herhaal stap 2-4 met `packages/apparaat_klaar_melding.yaml`, en vul
daarin je eigen notify-service en lamp(en) in (of verwijder de lamp-stappen
als je alleen de melding wilt).

---

## Installatie — optie B: via de UI (stap voor stap)

Geen zin in packages, of je wilt liever alles via de UI klikken? Dat kan
ook — het kost alleen meer klikwerk. Maak de volgende helpers aan via
**Instellingen → Apparaten & diensten → Helpers → Helper toevoegen**:

| Type | Naam |
|---|---|
| Aan/uit-schakelaar | Apparaat sessie bezig |
| Datum en tijd | Apparaat sessie start |
| Tekstveld (max 255) | Apparaat sessie duren |
| Tekstveld (max 255) | Apparaat sessie kwh |
| Tekstveld (max 255) | Apparaat curve duren |
| Tekstveld (max 255) | Apparaat curve kwh |
| Tekstveld (max 255) | Apparaat sessie curves |
| Tekstveld (max 255) | Apparaat sessie curves grof |
| Tekstveld (max 255) | Apparaat huidige curve |
| Tekstveld (max 255) | Apparaat huidige curve grof |

Maak daarnaast aan:
- Een **Integratie-helper** ("Riemann-som") met als bron je vermogenssensor,
  eenheid-voorvoegsel "k", eenheid-tijd "h" — dit geeft je de cumulatieve
  kWh.
- Een **Utility Meter-helper** met als bron de zojuist gemaakte
  integratie-sensor, cyclus "geen".
- Een **Template-sensor** met de `value_template` uit
  `packages/apparaat_sessie_voorspeller.yaml` (kopieer de hele
  `value_template:`-inhoud).

Maak vervolgens de twee automations aan via **Instellingen →
Automatiseringen → Automatisering toevoegen → In YAML bewerken**, en plak
daar de inhoud van de bijbehorende `automation:`-items uit het package-
bestand (één automation per keer aanmaken).

Let op: bij deze route moet je zelf overal de entity-namen die je bij de
helpers hebt gekozen, laten overeenkomen met wat er in de automations en de
sensor-template staat (Home Assistant genereert de entity_id meestal
automatisch op basis van de naam die je invoert, bijv. "Apparaat sessie
bezig" → `input_boolean.apparaat_sessie_bezig`).

---

## Dashboard-kaart

Zie `dashboard-card.yaml`. Vereist de [Mushroom](https://github.com/piitaya/lovelace-mushroom)
custom card (te installeren via HACS). Plak de inhoud in een nieuwe kaart
via de kaart-editor (kies "Handmatig" / YAML-modus).

---

## Marges en instellingen aanpassen

Alle "magische getallen" zitten met opzet los in het automation-bestand, met
uitleg erbij in de comments:

- Checkpoint-interval en -aantal (fijn/grof)
- Duplicaat-marges (±5 min duur, ±10%/±0,05 kWh)
- Val-terug-coulance (5 minuten voordat wordt overgeschakeld naar de
  een-na-beste match)
- Hoeveel sessies/curves bewaard blijven (standaard 20 resp. 6)

Pas deze gerust aan naar wat bij jouw apparaat en gebruik past — er is geen
universeel "juist" getal, dit zijn allemaal vuistregels die tijdens de
ontwikkeling voor een specifieke wasmachine goed bleken te werken.

## Licentie

Doe ermee wat je wilt.
