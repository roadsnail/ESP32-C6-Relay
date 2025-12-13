# ESP32-C6-Relay
 This ESP32-C6 Relay Module (ESP32-C6_Relay_X1_V1.1) is available from various sellers on the Internet. It can be powered and programmed via its USB-C socket.  
 It can also be powered from a 7-60V DC supply according to the silk screen printing on the bottom of the module (although I haven't tried this yet).  

 I haven't yet been able to find a schematic for the module so I am unsure what isolation exists between the 7-60V DC input and the USB-C socket.  

 I have programmed this using ESPHome. The initial loading of ESPHome (onto the module) was a pain taking several attempts. I'm guessing this may be due to fairly recent support for the C6 version of ESP32 module. 

 
![ESP32-C6-Relay_Module](https://github.com/user-attachments/assets/a2984527-bf0b-4460-a53b-7726257bcbf4)

![ESP32-C6-Relay_Module_1](https://github.com/user-attachments/assets/2bf053be-5092-455f-a437-f65b2f589947)


### GPIO Pin assisgnments
- GPIO2	LED
- GPIO19	Relay

### ESPHome - The ESP32-C6 is programmed using the ESPHome framework

A basic ESPHome sketch to control the relay integrating with Home Assistant subscribing/publishing to MQTT server.  

It's function is to control the module's relay using a virtual switch (relay_control) to switch the relay ON immediately. Switching OFF the relay via the virtual switch starts a time (turn_off_delay) which on expiry switches the relay OFF.  
The turn_off_delay is programmable from Home Assistant enabling a turn off time between 0 and 300 seconds  
This sketch is provided "as-is" as an example of how to control this ESP32-C6 Relay module with integration with MQTT and Home Assistant. 


- **turn_off_delay**    (in seconds, range 0 - 300) before switching off the relay after the virtual switch (relay_control) is switched off
- **relay_1**           switch (GPIO19) controls the relay off/on state
- **status_led**        switch (GPIO2) controls the onboard LED off/on state
- **relay_control**     is a virtual switch. ON turns on the relay, OFF turns off the relay after programmable time (in seconds) turn_off_delay

### Testing  
Having uploaded the following sketch to the module via ESPHome, a quick test is to paste the following into a browser and check that the relay   
opens and closes. Replace **relay.localdomain** with the IP address of your relay module.   

- **http://relay.localdomain/switch/relay_control/turn_on** and check that the relay is switched on (the relay powered LED should be lit) then...
- **http://relay.localdomain/switch/relay_control/turn_off** checking that the status_led is lit for the turn_off_delay time (in seconds) then the relay should switch off with the status_led also switching off  
- **http://relay.localdomain/switch/relay_control** will return the status of the virtual switch relay_control

To control the physical relay...
- **http://relay.localdomain/switch/main_relay__physical_/turn_off** (note underscores for spaces and brackets in Main Relay (Physical) )
OR
- **http://relay.localdomain/switch/main_relay__physical_/turn_on** (note underscores for spaces and brackets in Main Relay (Physical) )  





## Sketch

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
