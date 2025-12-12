# ESP32-C6-Relay
 ESP32-C6 Relay Module

 
![ESP32-C6-Relay_Module](https://github.com/user-attachments/assets/a2984527-bf0b-4460-a53b-7726257bcbf4)

### GPIO Pin assisgnments
- GPIO2	LED
- GPIO19	Relay

### ESPHome - The ESP32-C6 is programmed using the ESPHome framework

An ESPHome sketch to control the relay integrating with Home Assistant subscribing/publishing to MQTT server.  

- **turn_off_delay**    (in seconds, range 0 - 300) before switching off the relay after the virtual switch (relay_control) is switched off
- **relay_1**           switch (GPIO19) controls the relay off/on state
- **status_led**        switch (GPIO2) controls the onboard LED off/on state
- **relay_control**     is a virtual switch. ON turns on the relay, OFF turns off the relay after programmable time (in seconds) turn_off_delay

### MQTT

The sketch 
  





```
esphome:
  name: "relay"
  friendly_name: relay
  min_version: 2025.9.0
  name_add_mac_suffix: false

  on_boot:
    priority: -10
    then:
      # Publish relay state
      - mqtt.publish:
          topic: relay-controller/switch/main_relay/state
          payload: !lambda "return id(relay_1).state ? \"ON\" : \"OFF\";"
          retain: true

      # Publish control state
      - mqtt.publish:
          topic: relay-controller/switch/relay_control/state
          payload: !lambda "return id(relay_control_state) ? \"ON\" : \"OFF\";"
          retain: true

      # Publish turn-off delay
      - mqtt.publish:
          topic: relay-controller/switch/relay_control/turn_off_delay/state
          payload: !lambda |-
            return std::to_string(id(turn_off_delay));
          retain: true

esp32:
  variant: esp32c6
  framework:
    type: esp-idf

logger:
web_server:
  port: 80

wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password
  power_save_mode: none

mqtt:
  broker: mqtt_server.localdomain
  id: mqtt_client

  on_message:
    - topic: relay-controller/switch/relay_control/turn_off_delay/set
      then:
        - lambda: |-
            int new_delay = atoi(x.c_str());
            if (new_delay < 0) new_delay = 0;
            if (new_delay > 300) new_delay = 300;
            id(turn_off_delay) = new_delay;
            ESP_LOGI("mqtt", "Turn-off delay set to %d seconds (MQTT)", new_delay);

        - mqtt.publish:
            topic: relay-controller/switch/relay_control/turn_off_delay/state
            payload: !lambda |-
              return std::to_string(id(turn_off_delay));
            retain: true

api:
ota:
- platform: esphome

# -----------------------
# GLOBALS
# -----------------------
globals:
  - id: turn_off_delay
    type: int
    restore_value: true
    initial_value: '10'

  - id: relay_control_state
    type: bool
    restore_value: false
    initial_value: "false"

# -----------------------
# SWITCHES
# -----------------------
switch:
  # Physical relay
  - platform: gpio
    id: relay_1
    name: "Main Relay (Physical)"
    pin: GPIO19

    on_turn_on:
      - mqtt.publish:
          topic: relay-controller/switch/main_relay/state
          payload: "ON"
          retain: true

    on_turn_off:
      - mqtt.publish:
          topic: relay-controller/switch/main_relay/state
          payload: "OFF"
          retain: true

  # LED
  - platform: gpio
    name: "Status LED (Delay Active)"
    id: status_led
    pin: GPIO2

  # Virtual control switch (UI)
  # Virtual control switch
  #
  # to control, send...
  # http://relay.localdomain/switch/relay_control/turn_off OR
  # http://relay.localdomain/switch/relay_control/turn_on
  #
  # http://relay.localdomain/switch/main_relay__physical_/turn_off (note underscores for spaces and brackets in Main Relay (Physical) )
  #
  #To get status...
  #
  # http://relay.localdomain/switch/relay_control
  #
  - platform: template
    id: relay_control
    name: "Relay Control"

    turn_on_action:
      - lambda: "id(relay_control_state) = true;"
      - switch.turn_on: relay_1
      - mqtt.publish:
          topic: relay-controller/switch/relay_control/state
          payload: "ON"
          retain: true

    turn_off_action:
      - lambda: "id(relay_control_state) = false;"
      - switch.turn_on: status_led
      - mqtt.publish:
          topic: relay-controller/switch/relay_control/state
          payload: "OFF"
          retain: true
      - logger.log: "Starting turn-off delay..."
      - delay: !lambda "return id(turn_off_delay) * 1000;"
      - switch.turn_off: relay_1
      - switch.turn_off: status_led

    lambda: |-
      // Report virtual switch state to HA, not physical relay
      return id(relay_control_state);

# -----------------------
# NUMBER ENTITY (HA UI)
# -----------------------
number:
  - platform: template
    id: ha_turn_off_delay
    name: "Relay Turn-Off Delay"
    min_value: 0
    max_value: 300
    step: 1
    unit_of_measurement: s
    optimistic: false

    lambda: |-
      return id(turn_off_delay);

    set_action:
      - lambda: |-
          id(turn_off_delay) = (int)x;
          ESP_LOGI("delay", "Turn-off delay set to %d via HA", id(turn_off_delay));
      - mqtt.publish:
          topic: relay-controller/switch/relay_control/turn_off_delay/state
          payload: !lambda "return std::to_string(id(turn_off_delay));"
          retain: true
```
