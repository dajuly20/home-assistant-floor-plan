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

### Option C — Plugin-Feld „Additional entities"  ·  Status: ✅ implementiert, 2× überarbeitet

Feld in den Entity-Optionen (nur bei Display type = **Label**): kommagetrennte
Liste weiterer Entity-IDs. Syntax je Token:
- `sensor.raum_humidity` – State der Zusatz-Entity
- `climate.raum|current_temperature` – `|attribut` zeigt ein Attribut statt des States

**Verlauf:**
1. `2acbb37` – jede Zusatz-Entity als eigenes `state-label` mit eigenem Hintergrund
   → **zwei getrennte Kreise**, vom User verworfen.
2. Idee „ein `markdown`-Element mit `{{ states(a) }} / {{ states(b) }}`" → **verworfen**:
   `picture-elements` erlaubt kein `markdown`/Karten-Element (nur state-badge/-icon/-label,
   icon, image, conditional, `custom:*`). `state-label` kann nur EINEN State, kein Template.
3. **Aktuell (`before-one-oval` → nächster Commit):** alle Werte in **einem Oval**,
   eine Zeile pro Wert. Das primäre `state-label` trägt die Hintergrund-Pille
   (`border-radius: 16px`, `padding` unten = `0.15 + n*1.5em`), die Zusatz-Labels
   sitzen transparent 3 % tiefer. Kein Slash – ohne HA-Helfer ist eine echte
   einzeilige Kombi nicht möglich.

**Bekannte Schwächen von 3:** Pillenhöhe ist geschätzt (Zeilenanzahl), Zeilenabstand
fix in %, breitester Wert kann aus der Pille ragen → ggf. `scale`/`position` nachjustieren.
Rollback-Tag: `before-one-oval`.

### Option D — Gerät angeben statt einzelner Entities  ·  Status: ✅ Grundfunktion umgesetzt

**Umgesetzt:** Möbelname `device.<name>` oder `device.<id>` wird erkannt
(`isHomeAssistantEntity` + `device.`). In den Entity-Optionen: Feld **„Entities"**
+ Button **„Load from HA"** → `Controller.resolveDeviceEntityIds()` ruft
`/api/template` mit `{% set d = device_id('<x>') or '<x>' %}{{ device_entities(d) | list | join(',') }}`.
Auswahl (CSV, editierbar) wird als ein Oval gerendert (`buildOvalYaml`).
`device_id()` akzeptiert Gerätename *oder* -ID.

**Noch offen:** echte Checkbox-Liste statt Textfeld; Sortierung/Heuristik nach
`device_class`; Option E (generierter Template-Helfer) als Alternativ-Rendering.

Ursprüngliche Skizze:

Idee: Möbelstück auf ein **HA-Gerät** zeigen lassen, Plugin holt dessen Entities
selbst und zeigt die passende(n) an.

**HA-Grundlagen**
- Jedes Gerät hat eine `device_id` (Hex, steht in der URL:
  `…/config/devices/device/7d7d9509e8a710386eccecfc8d827923`).
- Die Zuordnung Entity → Gerät liegt in der *entity/device registry* und ist
  **nur per WebSocket-API** abrufbar (`config/entity_registry/list`,
  `config/device_registry/list`). Das Plugin macht aktuell nur REST → sieht keine Geräte.
- **Workaround ohne WebSocket**: `POST /api/template` wertet Jinja aus und ist
  mit dem Long-Lived-Token erreichbar:
  ```
  {"template": "{{ device_entities('7d7d9509e8a710386eccecfc8d827923') | list }}"}
  ```
  → `['sensor.x_temperature', 'sensor.x_humidity', 'sensor.x_battery', ...]`
  Weitere nützliche Funktionen: `device_id('sensor.x')`, `device_attr(id,'name')`,
  `device_attr(id,'name_by_user')`.

**Möglicher Umfang**
- [ ] Möbelname-Konvention `device.<device_id>` (analog zu `sensor.*`) erkennen.
- [ ] In `Controller` eine `resolveDeviceEntities(deviceId)` per `/api/template` ergänzen.
- [ ] Beim Generieren: Haupt-Entity heuristisch wählen
      (z. B. `sensor.*` mit `device_class: temperature`), Rest als *Additional entities*.
- [ ] „Fetch entities"-Dialog um eine Geräte-Ansicht erweitern
      (Gerät wählen → dessen Entities anzeigen/auswählen).
- [ ] Fallback wenn `/api/template` deaktiviert/nicht erreichbar ist.
- [ ] Optional: statt `device_id` den (eindeutigen) Gerätenamen zulassen und
      per Template auflösen.

**Pro**: keine Entity-IDs mehr abtippen, neue Sensoren am Gerät tauchen automatisch auf.
**Contra**: hängt an `/api/template`; Heuristik „welche Entity ist die Hauptanzeige"
ist nicht immer eindeutig; WebSocket wäre der „richtige" Weg, aber großer Umbau.

### Option E — Plugin GENERIERT den Template-Helfer  ·  Status: Favorit, nicht umgesetzt

Kombiniert das Beste aus B (echtes Oval, Slash) und D (nichts abtippen):
Der User pflegt **keine** Entity-Namen und **keine** Jinja – das Plugin baut den
kombinierten Sensor selbst und schreibt ihn in eine eigene YAML-Datei neben das
Dashboard-YAML.

**Ablauf beim Generieren**
1. Möbel heißt `device.<device_id>` (oder `sensor.x_temperature`, dann
   `device_id('sensor.x_temperature')` per `/api/template`).
2. `POST /api/template` mit
   `{{ expand(device_entities('<id>')) | selectattr('attributes.device_class','in',['temperature','humidity','carbon_dioxide']) | map(attribute='entity_id') | list }}`
   → die relevanten Entities des Geräts, nach `device_class` sortiert.
3. Plugin erzeugt einen Block in `home_assistant_floor_plan_sensors.yaml`:
   ```yaml
   template:
     - sensor:
         - name: "Grosses Bad Klima"
           unique_id: shfp_7d7d9509e8a710386eccecfc8d827923
           state: >
             {{ states('sensor.grossesbad_temphumid_temperature') }}{{ sep }}{{ states('sensor.grossesbad_temphumid_humidity') }}
   ```
   (Trennzeichen global konfigurierbar, Default `" / "`; optional device-basiert
   statt fester Entity-IDs, dann self-healing bei Umbenennung.)
4. Im Dashboard-YAML wird `sensor.grosses_bad_klima` als normales `state-label`
   referenziert → exaktes Oval, `tap_action` etc.

**Einmalige HA-Einrichtung durch den User**
`configuration.yaml`: `template: !include home_assistant_floor_plan_sensors.yaml`
(oder Inhalt einmal kopieren). Danach bei jedem Re-Generieren automatisch aktuell.

**Offene Punkte**
- [ ] Globale Option „kombinierte Klima-Sensoren generieren" + Trennzeichen-Feld.
- [ ] `Controller`: `/api/template`-Helper `evaluateTemplate(String)` (REST, kein neuer Dep).
- [ ] Gerät bestimmen: aus `device.<id>` direkt, sonst `device_id(<entity>)`.
- [ ] `device_class`-Zuordnung (welche Entities rein, in welcher Reihenfolge).
- [ ] Schreiben der `*_sensors.yaml` + Doku „einmal !include".
- [ ] Fallback wenn `/api/template` 404/deaktiviert → auf Weg A zurückfallen.
- [ ] Optional: Helfer per HA-API anlegen statt Datei (braucht WebSocket + Admin → später).

**Pro**: nichts abtippen, echtes Oval + Slash, überlebt Re-Generierung, ein `!include`.
**Contra**: braucht `/api/template`; einmaliges `!include`; Plugin schreibt eine 2. YAML-Datei.

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

---

## 4. OFFENER BUG: `device`-Gruppe erscheint nicht im Picker

**Stand 2026-08-31, ~12:40** — Commits `133e5cc` + `83c9046` eingespielt, „Fetch entities"
neu geklickt (Last fetch 12:36:12), Entity-Count 3741 → 3743. Trotzdem **keine
`device`-Gruppe** im rechten Baum „Available in Home Assistant", **kein Fehler-Popup**.

### Was gebaut wurde (Ist-Zustand im Code)

- `Controller.fetchDeviceNamesFromHomeAssistant()` — Template
  `{{ states | map(attribute='entity_id') | map('device_id') | reject('none') | unique | map('device_attr','name') | reject('none') | unique | list | join('|') }}`
- `fetchEntitiesFromHomeAssistant()` hängt `"device." + name` an `entityIds` an,
  setzt `lastDeviceFetchWarning` bei Exception **oder leerer Liste**.
- `Panel.triggerFetchEntities()` zeigt `getLastDeviceFetchWarning()` als Popup.
- `isHomeAssistantEntity()` kennt `device.`; `SUPPORTED_DOMAINS` enthält `device`.
- `EntityOptionsPanel`: „Entities"-Feld + „Load from HA" (`resolveDeviceEntityIds()`).

### Hypothesen (in Reihenfolge prüfen)

1. **Kein Popup + Count +2 ⇒ `fetchDeviceNamesFromHomeAssistant()` warf keine Exception
   und lieferte nicht-leer?** Dann müssten `device.*` in `cachedHaEntityIds` sein und
   der rechte Baum eine `device`-Gruppe zeigen. Tut er nicht → Verdacht auf
   **rechten Baum-Builder** (`populateHaEntitiesTree` ~Panel.java:1300 ff.):
   filtert er Domains gegen eine Whitelist? Accordion-Mode? → *zuerst hier gucken*.
2. **`map('device_id')` / `map('device_attr','name')` funktioniert nicht in `/api/template`.**
   `map` ruft Jinja-Filter; HA-eigene Filter sind evtl. nicht über `map` erreichbar.
   → In **Entwicklerwerkzeuge → Vorlagen** exakt dieses Template testen. Falls Fehler:
   auf `namespace`-Schleife über `states` zurück, ABER Timeout war das Problem →
   Schleife auf wenige Domains begrenzen (`states.sensor` + `states.binary_sensor` …)
   oder `device_attr` weglassen und nur `device_id`s sammeln, Namen einzeln per
   zweitem Call.
3. **Popup wurde angezeigt, aber vom User übersehen** (unwahrscheinlich, aber:
   Popup-Text ggf. in `checkEntities()`-Dialog untergegangen). → Warnung zusätzlich
   in ein sichtbares Label neben „Last fetch" schreiben.
4. **Cache-Persistenz zerlegt Gerätenamen**: `saveEntityCache`/`loadEntityCache`
   join/split auf `,` (Controller.java:~519). Gerätename mit `,` („Kitchen, Nook")
   → zersplittert. Eher `\n`-getrennt speichern oder Namen bereinigen.
5. **Der User lief doch auf altem Build.** → Versionslabel/Log beim Start ausgeben
   (`Plugin` Version in Titel oder Konsole), damit das eindeutig ist.
6. **Rechter Baum wird nach dem Fetch nicht neu aufgebaut**, nur `haEntityCountLabel`.
   → prüfen ob `populateHaEntitiesTree()` in `triggerFetchEntities`-Erfolgspfad
   aufgerufen wird (aktuell: `checkEntities()` — reicht das?).

### Wichtiger Nebenpfad — zuerst verifizieren

Die `device`-Gruppe im Picker ist nur **Komfort/Discovery**. Der eigentliche Pfad ist:
Möbel `device.<Name>` → Optionen → **„Load from HA"** (`resolveDeviceEntityIds`).
**Als Erstes testen ob DIESER Button funktioniert**, unabhängig vom Picker:
- Möbel `device.KleinesBad_TempHumid` benennen (exakter Gerätename!)
- Optionen öffnen, „Load from HA" klicken
- Erwartung: Feld füllt sich mit `sensor.temphumidkleinesbad_temperature, …`
- Fehler-Popup? Text notieren.

### Umsetzungsplan (wenn wieder Tokens da sind)

1. Template 2 in Dev-Tools verifizieren; ggf. robustes Fallback-Template festzurren.
2. `resolveDeviceEntityIds` (Load-from-HA-Button) end-to-end testen — das ist der
   kritische Pfad, muss zuerst grün sein.
3. Rechten Baum-Builder prüfen: wird `device`-Domain gerendert? Fetch → Rebuild?
4. Cache-Persistenz auf `\n`-Trennung umstellen (Namen können `,`/Leerzeichen haben).
5. Sichtbares Status-/Warnlabel statt/zusätzlich zum Popup.
6. Erst dann: echte Checkbox-Liste im Options-Dialog statt Textfeld.

Rollback-Tag für den ganzen device-Zweig: `before-device-groups`.

### ✅ Bearbeitet in `567f352` (2026-08-31)

- **Ursache 3+6 gefunden:** `checkEntities()` (rechter Baum) nutzt
  `getHaSelectedEntityIds()`, das `cachedHaEntityIds` auf die Domains
  **filtert, die der User schon als Möbel hat**. Ohne `device.*`-Möbel →
  keine `device`-Gruppe. → jetzt: `device` wird immer durchgelassen.
- **Punkt 1:** Template auf expliziten Loop mit **Pipe-Filtern**
  (`eid | device_id | device_attr('name')`) umgestellt — `map('device_id')`
  ist ein Kontext-Filter und bricht in `map()`.
- **Punkt 4:** Entity-Cache jetzt `\n`-getrennt (alte `,`-Caches laden weiter).
- **Punkt 5:** Fetch-Label zeigt `"<n> entities, <m> devices"`; Popup bei 0 Geräten.

**Noch zu verifizieren (nächste Runde):**
- „Load from HA"-Button (`resolveDeviceEntityIds`) end-to-end mit echtem HA.
- Ob `m devices` > 0 anzeigt und die `device`-Gruppe im rechten Baum + im
  „Select entities"-Dialog erscheint.
- Wenn 0/Fehler: Popup-Text bringt die Ursache (`/api/template` HTTP-Code).
- Danach: echte Checkbox-Liste im Options-Dialog.
