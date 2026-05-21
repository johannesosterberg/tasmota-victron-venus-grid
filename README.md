# Smartzähler als Grid Meter für Victron Venus OS

Einen beliebigen smarten Stromzähler mit Web-Schnittstelle als virtuellen Grid Meter in Venus OS einbinden — komplett über Node-RED und den `victron-virtual`-Node. Keine SSH-Eingriffe, keine Python-Skripte, kein modifiziertes Dateisystem.

Der Charme: Durch das vorgeschaltete Function-Node lässt sich so ziemlich jeder Smartzähler mit Web-Schnittstelle (HTTP oder MQTT) in Venus OS betreiben — Tasmota mit SML-Lesekopf, Shelly 3EM, SDM630 per Modbus-Bridge, Smart-Plugs, was auch immer. Egal ob 1- oder 3-phasig, egal ob per HTTP gepullt oder per MQTT gepushed.

Die KI eures Vertrauens wird euch, sobald ihr einmal das Prinzip verstanden habt, für jedes beliebige Zählerformat den passenden Übersetzer-Code liefern.

<p align="center">
  <img src="/device-list.png" alt="Grid Meter in der Venus OS Device List" width="700">
</p>

## Voraussetzungen

- GX-Gerät mit **Venus OS Large** (Node-RED ist vorinstalliert)
- Smartzähler mit Leistungs- und Energiewerten als JSON über HTTP oder MQTT
- Bidirektionaler Zähler, wenn Bezug *und* Einspeisung unterschieden werden sollen

## Architektur

Der ganze Flow besteht aus vier Nodes plus optionalem Fehler-Catch:

<p align="center">
  <img src="/node-red-flow.png" alt="Node-RED Flow" width="650">
</p>

```
[inject 2s] → [HTTP GET] → [function] → [virtual device]
```

Der **virtual-Node** aus der Palette `node-red-contrib-victron-virtual` legt das Grid-Device selbst an und schreibt die Werte direkt rein — kein separates Anlegen in den Cerbo-Settings nötig.

## Schritt 1: Virtual Device Node konfigurieren

In der Palette unter **Victron Energy → Virtual** den `Virtual Device`-Node in den Flow ziehen und doppelklicken:

<p align="center">
  <img src="/virtual-device-config.png" alt="Virtual Device Konfiguration" width="450">
</p>

- **Device:** `Grid meter` auswählen (Dropdown zeigt alle möglichen Typen)
- **Name:** z.B. `Smartmeter Grid` oder wie du magst
- **Nr of phases:** entsprechend deinem Setup (siehe unten)
- **Initialize with defaults:** `Yes`

<p align="center">
  <img src="/device-types.png" alt="Geräte-Typ Dropdown" width="350">
</p>

> **Hinweis von Victron:** Virtuelle Geräte sind nicht offiziell für ESS-Setups empfohlen. Für produktive Anlagen empfiehlt Victron die offiziell unterstützten Zähler.

### Wie viele Phasen?

| Setup | Phasen |
|-------|--------|
| Einphasiger Multi, Zähler liefert nur Summe | **1** |
| Dreiphasiger Multi, Zähler liefert L1/L2/L3 einzeln | **3** |
| Dreiphasiger Multi, Zähler liefert nur Summe | 1 + ESS auf `Total of all phases` |

Niemals künstlich auf 3 Phasen aufteilen, wenn der Zähler nur die Summe kennt — ESS würde dann falsch regeln.

## Schritt 2: Quelle einrichten

### HTTP (empfohlen, wenn MQTT schon woanders genutzt wird)

- **Inject-Node:** Repeat `interval`, alle 2 s, "inject once after deploy"
- **HTTP-Request:** `GET http://<zähler-ip>/<endpunkt>`, Return `parsed JSON object`, Timeout 1500 ms

Für Tasmota z.B. `http://<ip>/cm?cmnd=Status%208` — liefert nur den Sensor-Block, kompakt und schnell. Für andere Zähler den jeweiligen JSON-Endpunkt (Shelly: `/status`, Shelly Gen2: `/rpc/Shelly.GetStatus` etc.).

### MQTT (Alternative)

- **MQTT-In-Node:** Topic entsprechend deinem Zähler, Output `parsed JSON object`
- Beispiel Tasmota: `tele/<topic>/SENSOR`, dazu auf dem Gerät `MqttHost <cerbo-ip>` und `SetOption59 1` für sofortiges Senden bei Werteänderungen

## Schritt 3: Function — der flexible Übersetzer

Der virtual-Node erwartet ein **flaches Objekt mit Venus-OS-Pfaden als Keys**:

```javascript
msg.payload = {
    "/Ac/Power":          773,     // + = Bezug, − = Einspeisung
    "/Ac/L1/Power":       773,
    "/Ac/L1/Voltage":     230,
    "/Ac/L1/Current":     3.36,
    "/Ac/Energy/Forward": 7850.66, // kWh kumuliert Bezug
    "/Ac/Energy/Reverse": 2.63,    // kWh kumuliert Einspeisung
    "/Ac/Frequency":      50
};
```

Die Function muss also nur das JSON deines Zählers in dieses Schema überführen. Im Folgenden zwei Beispiele.

### Beispiel A: Tasmota mit SML-Smartmeter (Treiber E320)

Typisch für IR-Lesekopf am EVU-Zähler (EMH, Iskra, Logarex etc.).

```javascript
const s = msg.payload?.StatusSNS?.E320;
if (!s) {
    const last = context.get("last");
    if (!last) return null;
    msg.payload = last;
    return msg;
}
const power = Number(s.power) || 0;
msg.payload = {
    "/Ac/Power":          power,
    "/Ac/L1/Power":       power,
    "/Ac/L1/Voltage":     230,
    "/Ac/L1/Current":     power / 230,
    "/Ac/Energy/Forward": Number(s.E_in)  || 0,
    "/Ac/Energy/Reverse": Number(s.E_out) || 0,
    "/Ac/Frequency":      50
};
context.set("last", msg.payload);
return msg;
```

### Beispiel B: Shelly 3EM mit Tasmota (dreiphasig, ENERGY-Block mit Arrays)

```javascript
const e = msg.payload?.StatusSNS?.ENERGY ?? msg.payload?.ENERGY;
if (!e) return null;
const P = Array.isArray(e.Power)   ? e.Power   : [e.Power, 0, 0];
const U = Array.isArray(e.Voltage) ? e.Voltage : [e.Voltage, 0, 0];
const I = Array.isArray(e.Current) ? e.Current : [e.Current, 0, 0];
msg.payload = {
    "/Ac/Power":          P[0]+P[1]+P[2],
    "/Ac/L1/Power":       P[0], "/Ac/L2/Power":   P[1], "/Ac/L3/Power":   P[2],
    "/Ac/L1/Voltage":     U[0], "/Ac/L2/Voltage": U[1], "/Ac/L3/Voltage": U[2],
    "/Ac/L1/Current":     I[0], "/Ac/L2/Current": I[1], "/Ac/L3/Current": I[2],
    "/Ac/Energy/Forward": Number(e.Total_in)  || Number(e.Total) || 0,
    "/Ac/Energy/Reverse": Number(e.Total_out) || 0,
    "/Ac/Frequency":      50
};
return msg;
```

Für andere Zähler (Shelly Pro 3EM nativ, SDM630 per ESPHome, Eastron etc.) lässt sich nach demselben Muster eine Function schreiben — einfach das JSON deines Geräts im Debug-Sidebar anschauen und die Felder auf die Venus-OS-Pfade mappen.

## Schritt 4: Verbinden und Deploy

```
[inject] → [HTTP] → [function] → [virtual device]
```

Nach dem Deploy sollte unter dem virtual-Node ein grünes Statuslabel erscheinen wie `Updated 7 paths for grid`. Das ist die Bestätigung, dass die Werte ankommen.

## Verifikation

1. Cerbo Device List → dein Grid Meter erscheint mit Live-Werten
2. Energiefluss-Übersicht zeigt den Netz-Pfeil in die richtige Richtung
3. **Vorzeichen-Test:** Mittags bei PV-Überschuss → `/Ac/Power` wird negativ

## Häufige Stolpersteine

| Symptom | Lösung |
|---------|--------|
| `Invalid payload type: number` | Werte in Objekt verpacken mit `/Ac/...`-Keys statt Einzelwert senden |
| Virtual-Node ohne Status nach Deploy | Function gibt `null` zurück — JSON-Pfad zur Zählerantwort im Debug-Sidebar prüfen |
| Vorzeichen immer positiv | Zähler nicht bidirektional, ggf. signed auslesen oder anderen Sensor verwenden |
| Werte verschwinden bei Zähler-Ausfall | `context.set/get("last", …)` für Last-Known-Good (in Beispielen enthalten) |

## Vorteile

- Komplett über Node-RED, **keine zusätzliche Software** auf Venus OS
- Updatesicher (keine `/data/etc/...`-Hacks)
- Format-agnostisch durch Function-Node — funktioniert mit jedem JSON-fähigen Zähler
- Der Zähler kann gleichzeitig Home Assistant, IOBroker und Venus OS füttern

## Haftungsausschluss

Diese Anleitung beschreibt eine private Bastellösung und wird ohne jegliche Gewährleistung bereitgestellt. Die Nutzung erfolgt auf eigenes Risiko und in eigener Verantwortung. Eine Haftung für Schäden an Geräten, Datenverlust, fehlerhafte Abrechnungen, Beeinträchtigungen der Stromversorgung oder sonstige unmittelbare oder mittelbare Folgen ist ausgeschlossen, soweit gesetzlich zulässig.

Arbeiten an der Elektroinstallation dürfen ausschließlich durch eine zugelassene Elektrofachkraft ausgeführt werden. Für die Einspeisung von Strom ins öffentliche Netz gelten die Vorgaben des jeweiligen Netzbetreibers (insbesondere VDE-AR-N 4105 in Deutschland); die hier beschriebene Lösung ist ausdrücklich **kein** zertifizierter, eichrechtskonformer oder von einem Netzbetreiber zugelassener Energiezähler und darf nicht als solcher verwendet werden. Sie dient ausschließlich der internen Visualisierung und Steuerung im eigenen Energiemanagement.
