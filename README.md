# EZLife / Golden Designs / Dynamic 2-Person Infrared Sauna — Home Assistant Integration

This guide documents how to add remote pre-heat control and temperature monitoring to an EZLife / Golden Designs / Dynamic 2-person far-infrared sauna (tested on models using the YT2R-1.4S controller board, including the DYN-6209-01) using ESPHome and Home Assistant. No proprietary app, no cloud dependency, minimal permanent modifications to the sauna (4 soldered wires). These Saunas come under many different names, the one I have in particular is from Nebraska Furniture Mart sold under the name EZLife. Seems like these saunas are re-branded under multiple different names and sold at a variety of stores like Nebraska Furniture Mart and Costco.

---

## Wiring Overview

![Wiring Diagram](./wiring-diagram.svg)

---

## How the Sauna Controller Works

The YT2R-1.4S (and equivalent CP-150 class) controller is split across two boards:

- **Roof-mounted power supply box** — the brain. Controls the heating elements, houses the fuse, NTC thermal cutoff, and labeled headers for PANEL, CONTROL, HEATER, LIGHTING, LAMPROOF, and DC 12V connections.
- **Inside control panel** — the user interface. Contains the POWER, WORK/START, TEMP, TIME, and LIGHT buttons along with the LED display. Connected to the roof box via a 10-pin Micro-Fit 3.0 harness (the multi-colored ribbon cable).

To start heating, two button presses are required in sequence: POWER (wakes the panel), then WORK (engages the heating elements). Restoring AC power alone does not start the sauna — the controller boots to idle and waits for manual input. This is why a smart outlet alone is insufficient.

---

## Bill of Materials

### Relay Control

| Item | Notes | Approx. Cost |
|---|---|---|
| [LC Technology ESP8266 Relay X2 board](https://a.co/d/0howLEoG) | ESP-01 based, dual relay, UART-controlled | $10 |
| [TOBSUN or equivalent 12V to 5V DC-DC buck converter](https://a.co/d/01iOdCmf) | Screw terminal type | $8 |
| [680µF electrolytic capacitor](https://a.co/d/05bnxP4j) | Across 5V output of buck converter | $1 (Variety Pack) |
| [Molex Micro-Fit 3.0 10-pin connector kit](https://a.co/d/01lkA7RM) | For 12V power tap from panel harness | $8 |
| [Shelly 16A](https://a.co/d/0gMgNsz7) | Power monitor on wall outlet | $25 |

### Temperature Monitoring

| Item | Notes | Approx. Cost |
|---|---|---|
| [ESP-01 module](https://a.co/d/03ptQbaO) | Standalone, not the relay board | $4 |
| [AMS1117 3.3V regulator breakout](https://a.co/d/06HKiK6P) | Powers the temp ESP-01 from 5V | $6 (10-pack) |
| [Waterproof DS18B20 module](https://a.co/d/0awZXOCP) | Breakout board with built-in pull-up resistor | $8 |
| [100µF electrolytic capacitor](https://a.co/d/05bnxP4j) | Across 3.3V output of AMS1117 | $1 (Variety Pack) |

### Already Assumed

- Home Assistant instance running on your network
- ESPHome add-on installed in Home Assistant
- Basic soldering iron and multimeter
- Wiring and WAGO connectors or preferred connection method
- Preferred mounting solution, I used [this junction box mounting plate](https://a.co/d/01tzoLNM)
- USB to serial adapter for ESP flashing (flashing instructions not covered here)

---

## Why Not Use the OEM Temperature Sensor

The factory NTC thermistor plugs directly into the roof power supply and is part of the thermal safety chain. Tapping it risks adding parallel impedance that skews the controller's temperature reading and can cause premature thermal cutoff or refusal to heat. A separate sensor adds no risk to the OEM safety chain and gives a more useful reading at seated head height rather than the ceiling vent location the factory probe occupies.

---

## Why the Relay Board Cannot Host the Temperature Sensor

The LC Technology ESP8266 Relay X2 board uses an ESP-01 module. The ESP-01 exposes only four GPIOs. GPIO1 and GPIO3 are consumed by the UART interface to the relay controller chip. GPIO0 and GPIO2, while exposed on the programming header, are connected to the relay controller MCU on the board's PCB traces and cannot be used for 1-Wire communication. A dedicated ESP-01 for temperature sensing is the clean solution.

---

## Shelly 16A — Outlet Power Monitor

A Shelly 16A is installed on the wall outlet that the sauna plugs into. It acts as a hard AC master kill controllable from Home Assistant and monitors real power consumption so you can verify the sauna is actually heating.

The Shelly exposes three entities to Home Assistant automatically via its native integration:

- `switch.sauna_outlet` — turns AC power on and off
- `sensor.sauna_outlet_power` — live wattage draw in watts
- `sensor.sauna_outlet_energy` — cumulative energy in kWh

---

## Wiring

### 12V Power Tap
The roof power supply box has labeled connectors. The two connectors silkscreened "12V" on the power supply are switched — only live when the sauna is already powered on. These are not suitable.

The correct 12V source is a pair of wires on the 10-pin Micro-Fit 3.0 harness running between the inside control panel and the roof power supply. On this build the correct pair was the **black and green wire**. This was confirmed by probing with a multimeter in idle state and verifying heater operation was not interrupted when the pair was loaded. Create a Y-splice using a Micro-Fit 3.0 pass-through cable — do not cut the original harness.
<img width="479" height="359" alt="image" src="https://github.com/user-attachments/assets/4c26f71e-aabd-420f-948e-48bb5ea51856" />



### Buck Converter
Connect the 12V pair to the INPUT terminals of the buck converter. Verify 5.0V at the OUTPUT terminals. Connect the 680µF electrolytic capacitor across the OUTPUT terminals observing polarity (long leg to positive). This prevents both ESPs from failing to boot when AC power is restored due to slow voltage ramp-up.
<img width="564" height="410" alt="image" src="https://github.com/user-attachments/assets/927f8ea2-84af-451b-a462-7d412c43e047" />

### Relay Board Power
Connect buck converter 5V and GND outputs to the IN+ and IN- screw terminals on the LC Technology relay board. The AMS1117 3.3V regulator for the temperature ESP-01 is powered from the **5V pin on the right-side header of the relay board**, not directly from the buck converter.

### Button Pad Wiring
Open the inside control panel housing. Locate the POWER and WORK/START tactile switches. Use a multimeter in continuity mode to identify the two active terminals on each button. Solder a pair of 22 AWG wires to each button and route them to the roof alongside the existing harness.

At the relay board, connect POWER button wires to Relay 1: COM1 and NO1. Connect WORK button wires to Relay 2: COM2 and NO2. NC terminals are left unconnected.

### Control board face for reference
<img width="363" height="543" alt="image" src="https://github.com/user-attachments/assets/9d61a30b-6bd1-423e-93b8-85498705a4d1" />

### Control board with wires attached
<img width="355" height="483" alt="image" src="https://github.com/user-attachments/assets/0b86e3fb-8a0e-4847-90b3-780d7c083c81" />

### Relay Board
<img width="630" height="428" alt="image" src="https://github.com/user-attachments/assets/bfc76d29-a452-4ea4-b5c3-27d5fe7b6fd2" />

### AMS1117
The AMS1117 takes 5V from the relay board's header pin and outputs 3.3V. Connect the 100µF capacitor across the 3.3V output. Connect AMS1117 3.3V/GND output to ESP-01 and DS18B20.

### Temperature Sensor Wiring
Only the data wire connects to ESP-01 GPIO2.

The DS18B20 probe is mounted using aluminum tape in the opening left by the removed 3.5mm aux jack on the inside of the sauna. The sauna has built-in Bluetooth for audio so the aux port is not needed in my use case.
<img width="471" height="339" alt="image" src="https://github.com/user-attachments/assets/b3eb4b90-1863-4cc1-ab94-b93eaa74d8b0" />


### Everything all together mounted on a PCB back plate screwed into some scrap wood
<img width="853" height="658" alt="image" src="https://github.com/user-attachments/assets/48593135-186a-4073-bb3f-3ab51c64109a" />

- Top left is the DS18B20 board
- Bottom left is the ESP8266 relay board
- Top right is the 12v to 5v DC step down transformer
- Bottom right is the AMS1117 converter and ESP-01
---

## ESPHome Configuration

### Relay Board — sauna.yaml

```yaml
esphome:
  name: sauna-relay
  friendly_name: sauna-relay

esp8266:
  board: esp01_1m

logger:
  baud_rate: 0

api:
  encryption:
    key: "YOUR_KEY_HERE"

ota:
  - platform: esphome
    password: "YOUR_PASSWORD_HERE"

wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password
  min_auth_mode: WPA2

uart:
  tx_pin: GPIO1
  rx_pin: GPIO3
  baud_rate: 115200

switch:
  - platform: template
    internal: true
    id: relay01
    optimistic: true
    restore_mode: ALWAYS_OFF
    turn_on_action:
      - uart.write: [0xA0, 0x01, 0x01, 0xA2]
    turn_off_action:
      - uart.write: [0xA0, 0x01, 0x00, 0xA1]
    on_turn_on:
      - delay: 100ms
      - switch.turn_off: relay01

  - platform: template
    internal: true
    id: relay02
    optimistic: true
    restore_mode: ALWAYS_OFF
    turn_on_action:
      - uart.write: [0xA0, 0x02, 0x01, 0xA3]
    turn_off_action:
      - uart.write: [0xA0, 0x02, 0x00, 0xA2]
    on_turn_on:
      - delay: 100ms
      - switch.turn_off: relay02

button:
  - platform: template
    name: "Sauna Power"
    id: power_button
    icon: mdi:power
    on_press:
      then:
        - switch.turn_on: relay01

  - platform: template
    name: "Sauna Heat"
    id: work_button
    icon: mdi:heat-wave
    on_press:
      then:
        - switch.turn_on: relay02

  - platform: template
    name: "Sauna Start"
    id: start_button
    icon: mdi:sauna
    on_press:
      then:
        - button.press: power_button
        - delay: 1500ms
        - button.press: work_button
```

**Notes:**

- `baud_rate: 0` disables serial logging, freeing GPIO1 and GPIO3 for relay UART
- The `on_turn_on` automation closes each relay for 100ms then opens it, simulating a momentary button press
- `Sauna Start` presses POWER and WORK in sequence with a 1500ms gap — this is the button to use for pre-heat
- Relays restore to ALWAYS_OFF on boot so buttons are never held closed during power cycling

### Temperature Sensor — sauna-temp.yaml

```yaml
esphome:
  name: sauna-temp
  friendly_name: Sauna Temperature

esp8266:
  board: esp01_1m

api:
  encryption:
    key: "YOUR_KEY_HERE"

ota:
  - platform: esphome
    password: "YOUR_PASSWORD_HERE"

wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password
  min_auth_mode: WPA2

one_wire:
  - platform: gpio
    pin: GPIO2

sensor:
  - platform: dallas_temp
    name: "Sauna Temperature"
    update_interval: 15s
    filters:
      - filter_out: NAN
      - sliding_window_moving_average:
          window_size: 4
          send_every: 4
```


---

## Home Assistant Controls
Here is how I setup my controls for the Sauna in Home Assistant. I use a general dashboard with quick controls and pop up cards on my phone for anything manual.

- I assume anything above 1000w means the sauna is on, draws ~1600w while heating. Theoretically you could turn off the heat and it would still be on but I dont have a use case for that. This also controls an automation for notification of the Sauna being on for too long.
- ~~I plan on adding a feature to the start/scheduling portion to send a notification to the user that started the sauna when it is pre-heated to about 130F.~~ Finished this - located below the controls yaml.

### Example of controller - Sauna OFF
<img width="567" height="486" alt="image" src="https://github.com/user-attachments/assets/6bcd0e8e-149d-4f7d-861d-f0c111f24ff4" />
<br>

**Notes:**

- I like to display temp/wattage when it is off.
- Schedule feeds an automation which turns on a helper so the automation triggers at the specified time and then turns off the scheduler after it runs.



### Example of controller - Sauna ON
<img width="577" height="534" alt="image" src="https://github.com/user-attachments/assets/040bf5bd-8c5d-4416-a0a8-38fb7ec5f31e" />
<br>

**Notes:**

- Have a conditional timer for total time Sauna has been running based on last changed power entity.



### YAML for Sauna Controls in HA
```yaml
type: vertical-stack
cards:
  - type: custom:bubble-card
    card_type: pop-up
    hash: "#sauna-popup"
    name: Sauna
    icon: mdi:heat-wave
    margin_top_mobile: 16px
    margin_top_desktop: 74px
    width_desktop: 540px
    bg_color: rgba(30, 30, 30, 0.95)
    bg_blur: true
    styles: |
      .bubble-pop-up-container {
        backdrop-filter: blur(50px) !important;
      }
      .bubble-pop-up {
        border: 1px solid rgba(255, 87, 34, 0.3) !important;
        box-shadow: 0 8px 32px rgba(0, 0, 0, 0.5) !important;
      }
  - type: custom:mushroom-template-card
    primary: |-
      {% if states('sensor.sauna_power_power') | float > 1000 %}
        🔥 Sauna is Heating
      {% else %}
        ❄️ Sauna is Off
      {% endif %}
    secondary: |2-
            {% if states('sensor.sauna_power_power') | float(0) > 1000 %}
              {{ states('sensor.sauna_power_power') | float(0) | round(0) }}W · {{ states('sensor.sauna_temperature_sauna_temperature') | round(0) }}°F
            {% else %}
              {{ states('sensor.sauna_temperature_sauna_temperature') | round(0) }}°F · Ready to start ({{ states('sensor.sauna_power_power') | float(0) | round(0) }}W)
            {% endif %}
    icon: |-
      {% if states('sensor.sauna_power_power') | float > 1000 %}
        mdi:heat-wave
      {% else %}
        mdi:sauna
      {% endif %}
    color: |-
      {% if states('sensor.sauna_power_power') | float > 1000 %}
        deep-orange
      {% else %}
        grey
      {% endif %}
    features_position: bottom
    card_mod:
      style: |
        ha-card {
          {% if states('sensor.sauna_power_power') | float > 1000 %}
          background: linear-gradient(135deg, rgba(255, 87, 34, 0.25), rgba(255, 138, 101, 0.15));
          border: 1px solid rgba(255, 87, 34, 0.4);
          box-shadow: 0 4px 12px rgba(255, 87, 34, 0.3);
          {% else %}
          background: rgba(255,255,255,0.05);
          border: 1px solid rgba(255,255,255,0.1);
          {% endif %}
        }
  - type: conditional
    conditions:
      - condition: numeric_state
        entity: sensor.sauna_power_power
        above: 1000
    card:
      type: custom:mushroom-template-card
      icon: mdi:timer-outline
      color: deep-orange
      features_position: bottom
      primary: >-
        {% set start =
        as_timestamp(states.sensor.sauna_power_power.last_changed) %}
                {% set elapsed = (now().timestamp() - start) | int %}
                {% set hours = elapsed // 3600 %}
                {% set minutes = (elapsed % 3600) // 60 %}
                {% if hours > 0 %}
                  Sauna has been on for {{ hours }}hours {{ minutes }}minutes
                {% else %}
                  Sauna has been on for {{ minutes }} minutes
                {% endif %}
      card_mod:
        style: |
          ha-card {
            background: rgba(255,255,255,0.05);
            border: 1px solid rgba(255,255,255,0.1);
            border-radius: 12px;
          }
  - type: horizontal-stack
    cards:
      - type: custom:mushroom-template-card
        primary: Start Sauna
        icon: mdi:play-circle
        icon_color: green
        tap_action:
          action: perform-action
          target: {}
          perform_action: script.sauna_smart_start
        card_mod:
          style: |
            ha-card {
              background: linear-gradient(135deg, rgba(76, 175, 80, 0.25), rgba(129, 199, 132, 0.15));
              border: 1px solid rgba(76, 175, 80, 0.3);
            }
      - type: custom:mushroom-template-card
        primary: Stop Sauna
        icon: mdi:stop-circle
        icon_color: red
        tap_action:
          action: call-service
          service: button.press
          target:
            entity_id: button.sauna_sauna_power
        card_mod:
          style: |
            ha-card {
              background: linear-gradient(135deg, rgba(244, 67, 54, 0.25), rgba(239, 83, 80, 0.15));
              border: 1px solid rgba(244, 67, 54, 0.3);
            }
  - type: custom:mushroom-title-card
    title: ⏰ Schedule
  - type: entities
    entities:
      - entity: input_boolean.sauna_schedule_enabled
        name: Enable Schedule
        icon: mdi:clock-outline
      - entity: input_datetime.sauna_schedule_time
        name: Start Time
        icon: mdi:clock-start
    card_mod:
      style: |
        ha-card {
          background: rgba(255,255,255,0.05);
          border: 1px solid rgba(255,255,255,0.1);
          border-radius: 12px;
        }
```

### Smart Sauna Start
- Has some logic since we dont have status on the power button - if the sauna doesnt start (wattage > 1000w) within 6 seconds it will send the start sequence again since it most likely turned the sauna off because power was left on.
- Captures the UUID for user initiation and notification.
```yaml
sauna_smart_start:
  alias: "Sauna - Smart Start"
  description: >
    Guards against double-pressing power, retries once if wattage does not 
    climb after the start sequence. Also captures the initiating user for 
    pre-heat notification.
  sequence:
    # Store who initiated — only if triggered by a real user (not a schedule/system call)
    - if:
        - condition: template
          value_template: "{{ context.user_id is not none and context.user_id != '' }}"
      then:
        - service: input_text.set_value
          target:
            entity_id: input_text.sauna_initiated_by
          data:
            value: "{{ context.user_id }}"
    # Bail out if already heating
    - condition: numeric_state
      entity_id: sensor.sauna_power_power
      below: 1000
    # First start attempt
    - service: button.press
      target:
        entity_id: button.sauna_sauna_start
    # Wait to see if wattage climbs
    - delay:
        seconds: 6
    # Retry once if sauna did not start (covers accidental power-off case)
    - if:
        - condition: numeric_state
          entity_id: sensor.sauna_power_power
          below: 1000
      then:
        - service: button.press
          target:
            entity_id: button.sauna_sauna_start
```

### Automation for the scheduled start
- Create a helper toggle called sauna schedule enabled.
- Create helper time called sauna schedule time.
- Script uses the toggle and time to run the start script and then turn off the toggle.
```yaml
  alias: Sauna Scheduled Start
  description: ""
  triggers:
    - at: input_datetime.sauna_schedule_time
      trigger: time
  conditions:
    - condition: state
      entity_id: input_boolean.sauna_schedule_enabled
      state:
        - "on"
  actions:
    - action: script.sauna_smart_start
      data: {}
    - action: input_boolean.turn_off
      metadata: {}
      target:
        entity_id: input_boolean.sauna_schedule_enabled
      data: {}
```

### Automation for notifying the user that initiated the Sauna of target temp
- Create a Text Helper named "Sauna Initiated By" with max length of 255 characters.
- Find your User ID's and the notification services - UUID is in Settings -> People -> Users -> Click a user -> At the top. Notification service should be in the mobile app device.


### Automation YAML for Capturing a Scheduled Start
```yaml
- alias: "Sauna - Capture Schedule User"
  description: Stores the HA user who enabled the sauna schedule
  trigger:
    - platform: state
      entity_id: input_boolean.sauna_schedule_enabled
      to: "on"
  action:
    - service: input_text.set_value
      target:
        entity_id: input_text.sauna_initiated_by
      data:
        value: "{{ trigger.to_state.context.user_id }}"
```

### Automation YAML for sending the notification
```yaml
- alias: "Sauna - Ready Notification at [ENTER TEMP]°F"
  description: Notifies only the user who started the sauna when it reaches temp
  trigger:
    - platform: numeric_state
      entity_id: sensor.sauna_temperature_sauna_temperature
      above: [ENTER TEMP]
  condition:
    - condition: numeric_state
      entity_id: sensor.sauna_power_power
      above: 1000
  action:
    - variables:
        # Replace with your actual user UUIDs and companion app notify service names
        user_notify_map:
          "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx": "notify.mobile_app_user"
          "yyyyyyyy-yyyy-yyyy-yyyy-yyyyyyyyyyyy": "notify.mobile_app_other_user"
        initiated_by: "{{ states('input_text.sauna_initiated_by') }}"
        notify_target: >
          {{ user_notify_map.get(initiated_by, 'notify.persistent_notification') }}
    - service: "{{ notify_target }}"
      data:
        title: "🔥 Sauna is Ready!"
        message: >
          Sauna has reached [ENTER TEMP]°F
          ({{ states('sensor.sauna_temperature_sauna_temperature') | round(0) }}°F).
          Enjoy your session!
    - service: input_text.set_value
      target:
        entity_id: input_text.sauna_initiated_by
      data:
        value: ""
```

### Automation to clear UUID if manual stop
```yaml
- alias: "Sauna - Clear Initiated By on Shutoff"
  description: Clears the stored user when the sauna stops heating
  trigger:
    - platform: numeric_state
      entity_id: sensor.sauna_power_power
      below: 1000
  action:
    - service: input_text.set_value
      target:
        entity_id: input_text.sauna_initiated_by
      data:
        value: ""
```
---

## Credits and References
- Not sure who is behind awholenother.com but the write-ups there inspired all of this
  - [awholenother.com — Adding remote starter to a Costco infrared sauna](https://www.awholenother.com/2025/06/26/sauna-remote-start.html)
  - [awholenother.com — ESPHome sauna controller update](https://www.awholenother.com/2026/02/12/remote-esphome-sauna-controller-update.html)
- Claude for doing my documentation cause I would never do this manually
