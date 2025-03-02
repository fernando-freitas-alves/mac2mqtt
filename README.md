# mac2mqtt

`mac2mqtt` is a program that allows viewing and controlling some aspects of computers running macOS via MQTT.

It publishes to MQTT:

* Current input/output volume
* Volume mute state
* Battery charge percent

You can send topics to:

* Change the input/output volume
* Mute/Unmute
* Put the computer to sleep
* Lock the computer
* Put the computer to sleep
* Lock and put the computer to sleep
* Shut down the computer
* Turn on/off the display
* Trigger any Shortcut

## Running

To compile and install this program:

- Install the [golang toolchain](https://go.dev/doc/install)
- Clone this repository: `git clone ...`
- Set your MQTT credentials in `mac2mqtt.yaml`
- Compile and install: `sh install.sh`

## Testing

To run `mac2mqtt` on your command line, disable the system-wide `mac2mqtt` process:

    sh uninstall.sh

Now you can run `./mac2mqtt` directly from the source directory. It will use the configuration values from `mac2mqtt.yaml` in the current working directory.

To rebuild, run:

    go build

Once you're finished, resume the system-wide `mac2mqtt` process:

    sh install.sh

## Home Assistant sample config

![](https://user-images.githubusercontent.com/47263/114361105-753c4200-9b7e-11eb-833c-c26a2b7d0e00.png)

`configuration.yaml`:

```yaml
script:
  mymac_sleep:
    icon: mdi:laptop
    sequence:
      - service: mqtt.publish
        data:
          topic: "mac2mqtt/bessarabov-osx/command/sleep"
          payload: "sleep"

  mymac_shutdown:
    icon: mdi:laptop
    sequence:
      - service: mqtt.publish
        data:
          topic: "mac2mqtt/bessarabov-osx/command/shutdown"
          payload: "shutdown"

  mymac_displaysleep:
    icon: mdi:laptop
    sequence:
      - service: mqtt.publish
        data:
          topic: "mac2mqtt/bessarabov-osx/command/displaysleep"
          payload: "displaysleep"

  mymac_displaywake:
    icon: mdi:laptop
    sequence:
      - service: mqtt.publish
        data:
          topic: "mac2mqtt/bessarabov-osx/command/displaywake"
          payload: "displaywake"

  mymac_displaylock:
    icon: mdi:laptop
    sequence:
      - service: mqtt.publish
        data:
          topic: "mac2mqtt/bessarabov-osx/command/displaylock"
          payload: "displaylock"

  mymac_displaylock_sleep:
    icon: mdi:laptop
    sequence:
      - service: mqtt.publish
        data:
          topic: "mac2mqtt/bessarabov-osx/command/displaylock"
          payload: "displaylock_sleep"

  mymac_runshortcut:
    icon: mdi:laptop
    sequence:
      - service: mqtt.publish
        data:
          topic: "mac2mqtt/bessarabov-osx/command/runshortcut"
          payload: "shortcut_name"

mqtt:
  sensor:
    - name: mymac_alive
      icon: mdi:laptop
      state_topic: "mac2mqtt/bessarabov-osx/status/alive"

    - name: "mymac_battery"
      icon: mdi:battery-high
      unit_of_measurement: "%"
      state_topic: "mac2mqtt/bessarabov-osx/status/battery"

  switch:
    - name: mymac_mute
      icon: mdi:volume-mute
      state_topic: "mac2mqtt/bessarabov-osx/status/mute"
      command_topic: "mac2mqtt/bessarabov-osx/command/mute"
      payload_on: "true"
      payload_off: "false"

  number:
    - name: mymac_volume
      icon: mdi:volume-medium
      state_topic: "mac2mqtt/bessarabov-osx/status/volume"
      command_topic: "mac2mqtt/bessarabov-osx/command/volume"
```

`ui-lovelace.yaml`:

```yaml
title: Home
views:
  - path: default_view
    title: Home
    cards:
      - type: entities
        entities:
          - sensor.mymac_alive
          - sensor.mymac_battery
          - type: 'custom:slider-entity-row'
            entity: number.mymac_volume
            min: 0
            max: 100
          - switch.mymac_mute
          - type: 'custom:slider-entity-row'
            entity: number.mymac_volume
            min: 0
            max: 100
          - type: button
            name: mymac
            entity: script.mymac_sleep
            action_name: sleep
            tap_action:
              action: call-service
              service: script.mymac_sleep
          - type: button
            name: mymac
            entity: script.mymac_shutdown
            action_name: shutdown
            tap_action:
              action: call-service
              service: script.mymac_shutdown
          - type: button
            name: mymac
            entity: script.mymac_displaysleep
            action_name: displaysleep
            tap_action:
              action: call-service
              service: script.mymac_displaysleep

      - type: history-graph
        hours_to_show: 48
        refresh_interval: 0
        entities:
          - sensor.mymac_battery
```

## MQTT topics structure

Program is working with several MQTT topics. All topics are prefixed with `mac2mqtt` + `COMPUTER_NAME`.

For example, this topic with the current volume on my machine is `mac2mqtt/bessarabov-osx/status/volume`

`mac2mqtt` send info to the topics `mac2mqtt/COMPUTER_NAME/status/#` and listen for commands in topics
`mac2mqtt/COMPUTER_NAME/command/#`.

### PREFIX + `/status/alive`

There can be `true` or `false` in this topic. If `mac2mqtt` is connected to MQTT server there is `true`.

If `mac2mqtt` is disconnected from MQTT there is `false`. This is the standard MQTT thing called Last Will and Testament.

### PREFIX + `/status/volume`

The value is the numbers from 0 (inclusive) to 100 (inclusive). The current volume of the computer.

The value of this topic is updated every 2 seconds by default (see YAML config).

### PREFIX + `/status/mute`

There can be `true` or `false` in this topic. `true` means that the computer volume is muted (no sound), `false` means that it is not muted.

### PREFIX + `/status/input_volume`

The value is a number from 0 (inclusive) to 100 (inclusive). The current input volume of the computer (usually the microphone).

The value of this topic is updated every 2 seconds by default (see YAML config).

### PREFIX + `/status/battery`

The value is the number up to 100. The charge percent of the battery.

The value of this topic is updated every 60 seconds by default (see YAML config).

### PREFIX + `/command/volume`

You can send an integer number from 0 (inclusive) to 100 (inclusive) to this topic. It will set the volume on the computer.

### PREFIX + `/command/mute`

You can send `true` or `false` to this topic. When you send `true` the computer is muted. When you send `false` the computer
is unmuted.

### PREFIX + `/command/input_volume`

You can send an integer number from 0 (inclusive) to 100 (inclusive) to this topic. It will set the input volume (usually the microphone) on the computer.

### PREFIX + `/command/sleep`

You can send string `sleep` to this topic. It will put the computer to sleep mode. Sending some other value will do nothing.

### PREFIX + `/command/shutdown`

You can send the string `shutdown` to this topic. It will try to shut down the computer. The way it is done depends on the user who runs the program. If the program is run by `root` the computer will shut down, but if it is run by an ordinary user the computer will not shut down if there is another user who logged in.

Sending some other value but `shutdown` will do nothing.

### PREFIX + `/command/displaysleep`

You can send string `displaysleep` to this topic. It will turn off the display. Sending some other value will do nothing.

### PREFIX + `/command/displaywake`

You can send string `displaywake` to this topic. It will turn on the display. Sending some other value will do nothing.

### PREFIX + `/command/displaylock`

You can send the string `displaylock` to this topic to lock the screen.

You can also send the string `displaylock_sleep` to this topic to not only lock the screen but also put the computer to sleep.

Sending some other value will do nothing.

### PREFIX + `/command/runshortcut`

You can send the name of a shortcut to this topic. It will run this shortcut in the Shortcuts app.

## Building

To build this program yourself, follow these steps:

1. Clone this repo
2. Make sure you have installed `go`, for example with `brew install go`
3. Install its dependencies with `go install`
4. Build with `go build mac2mqtt.go`

It outputs a file `mac2mqtt`. Make the binary executable (`chmod +x mac2mqtt`) and run `./mac2mqtt`.
