# Home Assistant Blueprints

## Kostal Battery external Control.

**Inspired by the FHEM automation Kostal Plenticore 10 Plus by Christian (ch.eick).\
Many thanks to Christian for his excellent work on the FHEM project.**

## Additional Config 
## Additional rewuired Intergration
[Core Integration Kostal](https://www.home-assistant.io/integrations/kostal_plenticore)\
[Kostal Smartmeter](https://github.com/MeisterTR/ha-kostal-smartmeter)\
[Solar Forecast ML](https://github.com/Zara-Toorox/ha-solar-forecast-ml)

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
        name: Zwangsladen
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

###4 Debug
Um die Automation zu Debuggen in den Entwicklerwerkzeugen - Template in HA hier eine Beispiel zum Sperren der Entladung des Speichers.

```yaml
{% set v_release_soc = 90 %}
{% set min_soc_val = states("number.wr_1_battery_min_soc") | float(0) %}
{% set fc0 = states("sensor.none_prognose_heute") | float(0) %}
{% set v_fc_limit = 10 %}
{% set current_lock_val = states("number.wr_1_battery_min_home_consumption") | float(0) %}
{% set wb_charging = states("sensor.enector_status") == 'Ladend' %}
{% set current_soc = states("sensor.wr_1_battery_soc") | float(0) %}
{% set current_season = states("sensor.season") %}
{% set is_winter = current_season == 'winter' %}

{% set v_midday_start = "11:30" %}
{% set start_h = v_midday_start.split(':')[0] | int %}
{% set is_winter_mode = is_winter %}

{% set zeit_bedingung_erfuellt = not is_winter_mode and now().hour >= 8 and now().hour < start_h and fc0 > v_fc_limit %}

target_lock: >
    {% if (fc0 < v_fc_limit and current_soc <= min_soc_val and is_winter) or wb_charging %} 30000
    {% elif current_lock_val > 100 and current_soc >= v_release_soc and not wb_charging %} 50
    {% elif current_lock_val > 100 and not wb_charging and not (fc0 < v_fc_limit and current_soc <= min_soc_val and is_winter) %} 50
    {% else %} {{ current_lock_val }}
    {% endif %}

---
### DIAGNOSE DER VARIABLEN:
* **Release SoC:** {{ v_release_soc }} %
* **Aktueller Batterie SoC (current_soc):** {{ current_soc }} %
* **Minimaler SoC (min_soc_val):** {{ min_soc_val }} %
* **Prognose Heute (fc0):** {{ fc0 }} kWh (Limit ist {{ v_fc_limit }})
* **Wallbox lädt? (wb_charging):** {{ wb_charging }} (Aktueller Status: '{{ states("sensor.enector_status") }}')
* **Aktueller Lock-Wert (current_lock_val):** {{ current_lock_val }} W
* **Ist Winter? (is_winter):** {{ is_winter }}
* **Jahreszeit:** {{ current_season }}

* **Mittag Startzeit (v_midday_start):** {{ v_midday_start }} (Stunde extrahiert: {{ start_h }} Uhr)
* **Aktuelle Uhrzeit (Stunde):** {{ now().hour }} Uhr
* **Bedingung 1 (Kein Winter):** {{ not is_winter_mode }}
* **Bedingung 2 (Zwischen 8 und {{ start_h }} Uhr):** {{ now().hour >= 8 and now().hour < start_h }}
* **Bedingung 3 (Prognose {{ fc0 }} > Limit {{ v_fc_limit }}):** {{ fc0 > v_fc_limit }}
* **GESAMTERGEBNIS ZEIT-BEDINGUNG:** **{{ zeit_bedingung_erfuellt }}**
```
