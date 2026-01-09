# Home Assistant Blueprints

## Kostal Battery external Control.

**Inspired by the FHEM automation Kostal Plenticore 10 Plus by Christian (ch.eick).\
Many thanks to Christian for his excellent work on the FHEM project.**

## Additional Config 

### 1. Der "Wartungs-Schalter" (Maintenance Switch)
Damit du die Automatisierung jederzeit pausieren kannst, ohne sie im Backend suchen zu müssen:

Gehe zu Einstellungen > Geräte & Dienste > Helfer.

Erstelle einen neuen Helfer: Schalter (Toggle).
```
Name: Batterie Steuerung Wartung

Entity-ID: input_boolean.battery_control_maintenance
```
Die Automatisierung läuft nur, wenn der Wartungsschalter AUS ist.

### 2. Status-Sensoren (Winter, Sperre, Drossel)
Die Logik "Winter-Mode" oder "Drosselung" findet bisher nur im Kopf der Automatisierung statt. Damit wir sie im Dashboard sehen, legen wir Template-Sensoren in deiner configuration.yaml (oder über die UI bei Helfern > Template) an:

```yaml
template:
  - sensor:
      - name: "Batterie Ladestatus Logik"
        unique_id: battery_control_status
        state: >
          {% set lock = states('number.wr_1_battery_min_home_consumption') | float(0) %}
          {% set forced = states('input_boolean.speicher_trigger_laden') %}
          {% if forced == 'on' %}
            🔴 Zwangsladung / Entladung aktiv
          {% elif lock > 100 %}
            🟡 Gesperrt (Laden bis 90% SOC)
          {% else %}
            🟢 Normalbetrieb (Entladen frei)
          {% endif %}
        icon: >
          {% if states('number.wr_1_battery_min_home_consumption') | float(0) > 100 %} mdi:battery-lock
          {% else %} mdi:battery-check
          {% endif %}

      - name: "Batterie Saison Modus"
        unique_id: battery_season_mode
        state: >
          {% set fc1 = states('sensor.solar_forecast_ml_energy_production_tomorrow') | float(0) %}
          {% if fc1 < 10 %}
            ❄️ Winter (Schlechte Prognose)
          {% else %}
            ☀️ Sommer (Gute Prognose)
          {% endif %}
        icon: mdi:weather-snowy-heavy
```
### 3. Das Dashboard (Lovelace-Konfiguration)
Hier ist der Code für eine Vertical Stack Card, die alle Steuerungswerte, den Status und die Zwangsladung zusammenfasst.

Kopiere diesen YAML-Code in dein Dashboard (Karte hinzufügen > Manuell):

```yaml
type: vertical-stack
cards:
  - type: entities
    title: 🔋 Kostal Speicher Steuerung
    show_header_toggle: false
    entities:
      - entity: input_boolean.battery_control_maintenance
        name: Wartungsmodus (Pause)
        icon: mdi:alert-octagon
      - type: divider
      - entity: sensor.batterie_ladestatus_logik
        name: Aktueller Status
      - entity: sensor.batterie_saison_modus
        name: Prognose-Saison
      - entity: number.wr_1_battery_min_soc
        name: Gesetzter Min-SoC
  - type: entities
    title: ⚡ Manuelle Befehle
    show_header_toggle: false
    entities:
      - entity: input_boolean.trigger_forced_charge
        name: Zwangsladen/-entladen
      - entity: number.wr_1_dc_power_abs
        name: DC Leistung (negative Werte)
        secondary_info: last-changed
  - type: entities
    title: 🔍 Register Monitoring (Live)
    show_header_toggle: false
    entities:
      - entity: number.wr_1_battery_min_home_consumption
        name: Entladesperre (30k = Sperre)
      - entity: number.wr_1_battery_max_charge_power
        name: Ladedrossel (1000W = Aktiv)
      - entity: number.wr_1_battery_max_soc
        name: Max-SoC (Register 23_09)
      - entity: sensor.active_power
        name: Netz-Bezug
      - entity: sensor.active_power_2
        name: Netz-Einspeisung
      - entity: sensor.sum_output_inverter_ac
        name: PV-Erzeugung AC
  - type: glance
    title: 🌗 Schattenmanagement
    entities:
      - entity: switch.wr_1_shadow_management_dc_string_1
        name: String 1
      - entity: switch.wr_1_shadow_management_dc_string_2
        name: String 2
grid_options:
  columns: full
```
