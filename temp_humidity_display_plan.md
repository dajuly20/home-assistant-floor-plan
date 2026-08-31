# Plan: Temperatur + Luftfeuchte an einem Marker anzeigen

> **Ziel**: An *einer* Stelle im Grundriss mehrere Sensorwerte zeigen, typisch
> Temperatur (°C) **und** Luftfeuchte (%), ohne für jeden Wert einen eigenen
> Marker mühsam zu positionieren.
> **Stand**: 2026-08-31

---

## 0. Ausgangslage / Analyse

- Jede Entity erzeugt in `Entity.buildYaml()` **genau ein** `picture-elements`-Element
  (`state-icon` / `state-label` / `state-badge`).
- Das Feld **Attribute** (nur bei Display type = Label) schreibt genau *eine*
  `attribute:`-Zeile – kein zweiter Wert, keine zweite Entity.
- `Controller.moveEntityIconsToAvoidIntersection()` schiebt überlappende Marker
  sogar **aktiv auseinander** – zwei Marker exakt übereinander gehen also nicht.
- **Wichtig / Stolperfalle**: Ein reiner Reed-/Tür-Kontakt (z. B. Tuya `TS0203`)
  hat in Zigbee2MQTT nur `contact` + `battery` – **keine** Temperatur/Feuchte.
  Für Klimawerte braucht es ein echtes Temp-/Feuchte-Gerät
  (Aqara WSDCGQ11LM, Sonoff SNZB-02, …), das in HA zwei Entities anlegt:
  `sensor.<raum>_temperature` und `sensor.<raum>_humidity`.

---

## 1. Lösungsoptionen (alle, die diskutiert wurden)

### Option A — HA-seitig: kombinierter Template-Sensor  ·  Status: dokumentiert, keine Plugin-Änderung

Ein Sensor in Home Assistant, dessen State beide Werte als ein String enthält.

```yaml
# configuration.yaml
template:
  - sensor:
      - name: "Klima Wohnzimmer"
        state: >
          {{ states('sensor.wohnzimmer_temperatur') }}°C /
          {{ states('sensor.wohnzimmer_luftfeuchte') }}%
```

Optional DRY per Makro in `config/custom_templates/climate.jinja`:

```jinja
{% macro climate(temp, hum) %}
{{ states(temp) }}°C / {{ states(hum) }}%
{% endmacro %}
```

```yaml
template:
  - sensor:
      - name: "Klima Wohnzimmer"
        state: "{% from 'climate.jinja' import climate %}{{ climate('sensor.wohnzimmer_temperatur','sensor.wohnzimmer_luftfeuchte') }}"
```

Danach `sensor.klima_wohnzimmer` im Plugin als Label verwenden.

- **Pro**: keine Plugin-Änderung, überlebt Re-Generierung, ein einziger Marker.
- **Contra**: pro Raum ein YAML-Eintrag in HA, `template:` kann Sensoren nicht per Schleife erzeugen.

### Option B — Zwei Marker  ·  Status: geht ohne Änderung

Beide Entities (oder dieselbe Entity zweimal mit unterschiedlichem Attribut) in
den Grundriss legen, beide als Label. Landen dicht nebeneinander (Plugin schiebt
sie leicht auseinander).

- **Pro**: null Aufwand im Code.
- **Contra**: zwei Marker-Objekte in Sweet Home 3D pflegen/positionieren, Optik "zwei Punkte".

### Option C — Plugin-Feature „Additional entities"  ·  Status: ✅ IMPLEMENTIERT (Commit `2acbb37`)

Neues Feld in den Entity-Optionen (nur sichtbar bei Display type = **Label**):
kommagetrennte Liste weiterer Entity-IDs. Jede wird als eigenes `state-label`
**unter** dem Haupt-Label gestapelt (3 % Versatz pro Zeile), erbt Position,
Style, Aktionen und Display-Condition des Haupt-Markers.

Syntax je Token:
- `sensor.raum_humidity` – zeigt den State der Zusatz-Entity
- `climate.raum|current_temperature` – `|attribut` zeigt ein Attribut statt des States

---

## 2. Umsetzung Option C (erledigt)

| Datei | Änderung |
|-------|----------|
| `Entity.java` | Konstante `SETTING_NAME_ADDITIONAL_ENTITIES`, `ADDITIONAL_ENTITY_LINE_SPACING_PERCENT = 3.0`; Feld `additionalEntities`; Getter/Setter/`isAdditionalEntitiesModified()`; Laden in `loadDefaultAttributes()`, Nullen in `resetToDefaults()`; `buildYaml()` refaktoriert → `buildElementYaml(...)` extrahiert; neu: `buildAdditionalEntitiesYaml(String elementType)` |
| `EntityOptionsPanel.java` | `additionalEntitiesLabel` + `additionalEntitiesTextField` (+ Tooltip); Layout-Zeile nach „Attribute"; Sichtbarkeit in `showHideComponents()` an Label gekoppelt; `markModified()` |
| `ApplicationPlugin.properties` | `additionalEntitiesLabel.text` + `.tooltip` |
| `README.md` | Abschnitt „The Additional Entities Field" + Zeile in Options-Tabelle |

### Persistenz
Wie alle Entity-Settings über `Settings` im `.sh3d`-Projektfile, Key
`<entityname>.additionalEntities` (CSV-String, `null` wenn leer).

### Erzeugtes YAML (Beispiel)
```yaml
- type: state-label
  entity: sensor.wohnzimmer_temperatur
  title: Wohnzimmer Temperatur
  style: { top: 40.00%, left: 55.00%, ... }
  ...
- type: state-label
  entity: sensor.wohnzimmer_luftfeuchte
  title: sensor.wohnzimmer_luftfeuchte
  style: { top: 43.00%, left: 55.00%, ... }   # +3 %
  ...
```
Bei Display-Condition ≠ Always werden alle Elemente gemeinsam in den
`conditional`-Block (`elements:`) eingerückt.

---

## 3. Offene TODOs / mögliche Erweiterungen

- [ ] **Titel der Zusatz-Labels**: aktuell = Entity-ID. Besser: aus Beschreibung/HA
      `friendly_name` ableiten, oder `entity|attribut|Titel`-Syntax.
- [ ] **Layout-Richtung / Abstand konfigurierbar** (hoch/runter, % pro Zeile) statt
      fixer 3 % nach unten.
- [ ] **Einheiten/Format**: optional `°C` / `%` als Suffix mitgeben
      (`suffix:` im state-label), damit man nicht auf HA-`unit_of_measurement` angewiesen ist.
- [ ] **Prefix pro Zeile** (z. B. `🌡️` / `💧`) analog zum Person-Prefix.
- [ ] **Anti-Overlap**: Zusatz-Labels sind reine YAML-Ausgabe, nicht Teil von
      `moveEntityIconsToAvoidIntersection()`. Prüfen, ob ein anderer Marker in die
      gestapelten Labels hineinrutschen kann.
- [ ] **Validierung** im Panel: Warnung bei ungültiger Entity-ID (kein Punkt) /
      Abgleich gegen gefetchte HA-Entities.
- [ ] **UI**: bei vielen Zusatz-Entities ist ein einzeiliges Textfeld eng –
      evtl. mehrzeilig oder Liste mit +/–.
- [ ] Screenshot in `doc/` aktualisieren (`entityOptionsFurniture.png` zeigt das neue Feld noch nicht).
