# Controlling Eight Sleep with an ATOMS3 Dev Kit

The Eight Sleep mattress is amazing for temperature control, but the app-based adjustment isn't ideal at night.

I wanted a simpler way to control my Eight Sleep mattress during the night, without the hassle of unlocking my phone and navigating the app. That’s when I had the idea to use the ATOMS3 Dev Kit to create a small, dedicated button for temperature control. This tiny ESP32 device features an integrated display and button, making it perfect for the task.

The device shows the current temperature, target heating level, sleep stage, and bed state on its display. With a press of the button, you can cycle through predefined temperature settings, and it integrates seamlessly with Home Assistant for enhanced automation and control.

![Atom S3 eight sleep](atoms3-eightsleep.jpeg)

## Features

- Displays:
  - Current bed temperature
  - Target heating level
  - Sleep stage (e.g., REM)
  - Bed state (e.g., Active)
- Toggle between pre-configured temperature settings with a button.
- Integration with Home Assistant for automation and control.

## Requirements

- **Hardware:**
  - [ATOMS3 Dev Kit with 0.85" display](https://thepihut.com/products/atoms3-dev-kit-w-0-85-inch-screen)
  - Eight Sleep mattress
- **Software:**
  - ESPHome
  - Home Assistant
  - [Custom Eight Sleep integration](https://github.com/lukas-clarke/eight_sleep)

## Installation

1. **Set up ESPHome**:
   - Install ESPHome on your ATOMS3 Dev Kit.
   - Use the provided `atoms3.yml` configuration file.

2. **Configure Home Assistant**:
   - Install the custom Eight Sleep integration.
   - Add the necessary sensors and automations from the example configuration.

## Example ESPHome Configuration

See the full ESPHome configuration in [atoms3.yml](esphome/atoms3.yml).

## Home Assistant Configuration

### Templates

1. **Bed State Shortening**:
   This template shortens the bed state string so it fits neatly on the display. For example, if the state is `Bed State: Active`, it extracts `Active`.

   ```yaml
   {% set state = states('sensor.jack_s_eight_sleep_side_bed_state_type') %}
   {{ state.split(':')[1] if ':' in state else state }}
   ```

2. **Percentage to App Scale Conversion**:
   This template converts the sensor’s -100% to 100% range into the app’s -10 to 10 scale, rounding to the nearest whole number.

   ```yaml
   {% set value = states('sensor.jack_s_eight_sleep_side_bed_state') | float %}
   {% set mapped_value = ((value / 100) * 10) | round %}
   {{ mapped_value }}
   ```

3. **Input Select Helper**:
   This helper lets you define a dropdown menu with options like `-2`, `-1`, `0`, and `1`, representing your preferred temperature settings. You can cycle through these options using the button on the ATOMS3.

### Automations

1. **Display Auto-Off**:
   This automation turns off the display backlight 30 seconds after the button is released, helping avoid unnecessary light during the night.

   ```yaml
   alias: 8 sleep display auto off
   description: ""
   triggers:
     - trigger: state
       entity_id:
         - binary_sensor.esphome_web_5c4f38_button
       to: "off"
       for:
         hours: 0
         minutes: 0
         seconds: 30
       from: "on"
     - trigger: state
       entity_id:
         - binary_sensor.esphome_web_5c4f38_button
       to: "off"
       for:
         hours: 0
         minutes: 0
         seconds: 30
       from: unavailable
   conditions: []
   actions:
     - action: switch.turn_off
       metadata: {}
       data: {}
       target:
         entity_id: switch.esphome_web_5c4f38_backlight
   mode: single
   ```

2. **Cycle Temperature Options**:
   This automation adjusts the Eight Sleep bed's temperature based on the currently selected temperature option. For instance, selecting `-2` lowers the temperature significantly, while `1` raises it slightly.

   ```yaml
   alias: 8 sleep button control
   description: ""
   triggers:
     - trigger: state
       entity_id:
         - input_select.eight_sleep_temperature_choices
       to: "-2"
       id: "-2"
       for:
         hours: 0
         minutes: 0
         seconds: 10
     - trigger: state
       entity_id:
         - input_select.eight_sleep_temperature_choices
       to: "-1"
       id: "-1"
       for:
         hours: 0
         minutes: 0
         seconds: 10
     - trigger: state
       entity_id:
         - input_select.eight_sleep_temperature_choices
       to: "0"
       id: "0"
       for:
         hours: 0
         minutes: 0
         seconds: 10
     - trigger: state
       entity_id:
         - input_select.eight_sleep_temperature_choices
       to: "1"
       id: "1"
       for:
         hours: 0
         minutes: 0
         seconds: 10
   conditions:
     - condition: not
       conditions:
         - condition: state
           entity_id: sensor.jack_s_eight_sleep_side_bed_state_type
           state: "off"
   actions:
     - choose:
         - conditions:
             - condition: trigger
               id:
                 - "-2"
           sequence:
             - action: eight_sleep.heat_set
               target:
                 entity_id: sensor.jack_s_eight_sleep_side_bed_temperature
               data:
                 sleep_stage: bedTimeLevel
                 target: -20
                 duration: 0
         - conditions:
             - condition: trigger
               id:
                 - "-1"
           sequence:
             - action: eight_sleep.heat_set
               target:
                 entity_id: sensor.jack_s_eight_sleep_side_bed_temperature
               data:
                 sleep_stage: current
                 target: -10
                 duration: 0
         - conditions:
             - condition: trigger
               id:
                 - "0"
           sequence:
             - action: eight_sleep.heat_set
               target:
                 entity_id: sensor.jack_s_eight_sleep_side_bed_temperature
               data:
                 sleep_stage: current
                 target: 0
                 duration: 0
         - conditions:
             - condition: trigger
               id:
                 - "1"
           sequence:
             - action: eight_sleep.heat_set
               target:
                 entity_id: sensor.jack_s_eight_sleep_side_bed_temperature
               data:
                 sleep_stage: current
                 target: 10
                 duration: 0
   mode: single
   ```

3. **Temperature Control**:
   This automation cycles through your predefined temperature settings (`-2`, `-1`, `0`, `1`) each time the button is pressed. If the backlight is off, the first press turns it on instead.

   ```yaml
   alias: 8 sleep cycle options
   description: ""
   triggers:
     - trigger: state
       entity_id:
         - binary_sensor.esphome_web_5c4f38_button
       from: "off"
       to: "on"
   conditions: []
   actions:
     - if:
         - condition: state
           entity_id: switch.esphome_web_5c4f38_backlight
           state: "off"
       then:
         - action: switch.turn_on
           metadata: {}
           data: {}
           target:
             entity_id: switch.esphome_web_5c4f38_backlight
       else:
         - action: input_select.select_next
           metadata: {}
           data:
             cycle: true
           target:
             entity_id: input_select.eight_sleep_temperature_choices
   mode: single
   ```

If you have ideas for improving this setup or additional features you'd like to see, feel free to share your suggestions in the comments. Feedback is always welcome!
