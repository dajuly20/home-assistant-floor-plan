# Anleitung: Zwei Entitäten (z. B. Temperatur + Luftfeuchte) in einem Oval

Es gibt zwei Wege. **Weg A** ist reine Plugin-Konfiguration und liefert *ein Oval
mit zwei Zeilen*. **Weg B** braucht einmalig einen HA-Helfer, liefert dafür *eine
Zeile mit frei wählbarem Trennzeichen* (`23,3 °C / 57 %`) und das exakte Oval-Aussehen.

---

## Voraussetzung

Ein Gerät, das in Home Assistant **zwei getrennte Entitäten** hat, z. B.:

- `sensor.grossesbad_temphumid_temperature`
- `sensor.grossesbad_temphumid_humidity`

Prüfen unter **Einstellungen → Geräte & Dienste → Gerät öffnen** oder in den
**Entwicklerwerkzeugen → Zustände**.

> Reine Tür-/Fensterkontakte (Reed) haben **keine** Temperatur/Feuchte – für die
> geht das nicht.

---

## Weg A – „Additional entities" im Plugin (ein Oval, zwei Zeilen)

### 1. Möbelstück anlegen

In Sweet Home 3D ein kleines Objekt (z. B. eine Box) an die Stelle im Raum setzen,
wo der Sensor sitzt. **Name** des Objekts = die **erste** Entität:

```
sensor.grossesbad_temphumid_temperature
```

Optional **Beschreibung** setzen → wird der Tooltip in Home Assistant.

### 2. Entity-Optionen öffnen

Im Plugin-Panel den Entity-Baum aufklappen, den Sensor doppelklicken (oder
Optionen-Button). Es öffnet sich der Dialog `sensor.grossesbad_temphumid_temperature`.

### 3. Felder setzen

| Feld | Wert |
| ---- | ---- |
| **Display type** | `Label` |
| **Attribute** | *(leer lassen)* |
| **Additional entities** | `sensor.grossesbad_temphumid_humidity` |

Mehrere Zusatzwerte → mit Komma trennen:
`sensor.grossesbad_temphumid_humidity, sensor.grossesbad_temphumid_co2`

Ein Attribut statt des States einer Zusatz-Entität → `|` anhängen:
`climate.grossesbad|current_temperature`

### 4. Generieren

Dialog schließen, Floor Plan generieren. Ergebnis: **ein** milchiges Oval,
Temperatur oben, Luftfeuchte in der Zeile darunter.

### 5. Nachjustieren (fast immer nötig)

Der Zeilenabstand ist fix (3 %), die Pillenhöhe wird aus der Zeilenzahl geschätzt.
Wenn's nicht sauber sitzt, in den Entity-Optionen anpassen:

- **Scale** kleiner (z. B. `80 %`) → Text und Pille kompakter
- **Position → Top / Left** → Oval verschieben
- **Background color** → z. B. `rgba(0, 0, 0, 0.4)` für dunklen Grund

### Erzeugtes YAML (zur Kontrolle)

```yaml
- type: state-label
  entity: sensor.grossesbad_temphumid_temperature
  title: Grosses Bad Temphumid Temperature
  style:
    top: 76.00%
    left: 40.97%
    text-align: center
    white-space: nowrap
    line-height: 1.4
    border-radius: 16px
    padding: 3px 10px 1.65em      # unten = 0.15 + (1 Zusatzzeile * 1.5) em
    background-color: rgba(255, 255, 255, 0.3)
    opacity: 100%
    transform: translate(-50%, -50%) scale(100%)
  tap_action: { action: more-info }
  double_tap_action: { action: none }
  hold_action: { action: more-info }
- type: state-label
  entity: sensor.grossesbad_temphumid_humidity
  title: sensor.grossesbad_temphumid_humidity
  style:
    top: 79.00%                   # 76 % + 3 %
    left: 40.97%
    text-align: center
    white-space: nowrap
    line-height: 1.4
    background-color: transparent
    opacity: 100%
    transform: translate(-50%, -50%) scale(100%)
  tap_action: { action: more-info }
  double_tap_action: { action: none }
  hold_action: { action: more-info }
```

### Grenzen von Weg A

- **Kein Trennzeichen / keine einzeilige Kombi** – `picture-elements` in HA kann in
  einem `state-label` nur *einen* State anzeigen, ohne Template. Zwei Werte in
  *einer Zeile* mit `/` gibt es nur über Weg B.
- Der breitere Wert kann seitlich leicht aus der Pille ragen.

---

## Weg B – HA-Template-Helfer (eine Zeile, Trennzeichen frei)

### 1. Helfer anlegen

**Einstellungen → Geräte & Dienste → Helfer → + Helfer erstellen → Vorlage →
Vorlage für einen Sensor**

- **Name**: `Grosses Bad Klima`
- **Status-Vorlage**:
  ```jinja
  {{ states('sensor.grossesbad_temphumid_temperature') }} °C / {{ states('sensor.grossesbad_temphumid_humidity') }} %
  ```

Trennzeichen frei wählbar – `/`, ` · `, ` | ` …
Kein Neustart nötig, der Helfer ist sofort als `sensor.grosses_bad_klima` da.

> **Zeilenumbruch statt Trennzeichen** in einer Zeile geht in einem `state-label`
> nicht zuverlässig – dafür ist Weg A da.

### 2. Im Plugin verwenden

Möbelstück benennen mit:

```
sensor.grosses_bad_klima
```

**Display type** = `Label`, **Additional entities** leer. Fertig – ein Oval,
eine Zeile, `23,3 °C / 57 %`, inkl. `tap_action` (More-Info öffnet den Helfer).

### Vor- / Nachteile Weg B

| Pro | Contra |
| --- | ------ |
| Exaktes Oval-Aussehen (normales `state-label`) | Pro Sensor einmal einen Helfer anlegen |
| Trennzeichen frei | More-Info zeigt den Helfer, nicht das Originalgerät |
| Eine saubere Zeile | – |

---

## Weg C – Gerät angeben, Entitäten im Plugin auswählen (kein Abtippen)

Wie Weg A (ein Oval, zwei Zeilen), aber du gibst **keine Entity-IDs** an, sondern
den **Gerätenamen** – das Plugin ermittelt den Rest.

Voraussetzung: funktionierende [HA-Verbindung](README.md#home-assistant-connection)
(URL + Token) im Plugin.

### 1. Möbelstück benennen

Name = `device.` + Gerätename (wie auf der Geräteseite in HA):

```
device.GrossesBad_TempHumid
```

Alternativ die Geräte-ID (Hex aus der URL): `device.7d7d9509e8a710386eccecfc8d827923`

### 2. Entitäten laden

Entity-Optionen öffnen → Display type = `Label` → neben dem Feld **„Entities"** auf
**„Load from HA"** klicken. Das Plugin löst das Gerät über `/api/template`
(`device_id()` akzeptiert Name *oder* ID) auf und trägt alle Entity-IDs des Geräts ein:

```
sensor.grossesbad_temphumid_temperature, sensor.grossesbad_temphumid_humidity, sensor.grossesbad_temphumid_battery, ...
```

### 3. Auswählen

Nicht gewünschte Einträge löschen, Reihenfolge anpassen (erste = oberste Zeile):

```
sensor.grossesbad_temphumid_temperature, sensor.grossesbad_temphumid_humidity
```

### 4. Generieren

Ergebnis wie Weg A: ein Oval, Temperatur oben, Feuchte darunter. Nachjustieren
(Scale/Position) wie dort.

> Fällt `/api/template` aus (deaktiviert / keine Verbindung), kannst du die IDs
> auch von Hand ins Feld schreiben.

---

## Welchen Weg nehmen?

- **Schnell testen, keine HA-Änderung** → Weg A.
- **Endgültige, saubere Optik mit `/` in einer Zeile** → Weg B.

Beides ist kombinierbar: die meisten Räume über Weg A, einzelne „Vorzeige-Räume"
über Weg B.
