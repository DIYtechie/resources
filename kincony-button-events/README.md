# KinCony KC868-A16: press, release, multi-click and hold events with ESPHome

I spent a long time looking for a useful way to combine single, double and triple clicks with press-and-hold control on ordinary wired wall buttons. This is the ESPHome configuration behind my [Making wall switches smart using ESPHome](https://runesblog.com/2026/01/03/making-wall-switches-smart-using-esphome/) project.

The useful part is having **immediate press/release events alongside on-device click classification**. Release does not have to wait for a multi-click timeout, which makes it useful for stopping dimming as soon as the button is released.

[Download the YAML](kincony.yaml) · [Single button ZigBee2MQTT dimmer flow](https://flows.nodered.org/flow/b3aa835c9ead60b8ec96b63ae2e80595)

## What is included

- All 16 inputs on the KC868-A16, using its two PCF8574 input expanders.
- A separate state and timers for each input.
- `pressed`, `released`, `single`, `double`, `triple` and `long` events.
- A long-press event that fires **while you are still holding the button**.
- Ethernet using DHCP, encrypted ESPHome API, and OTA credentials supplied through local secrets.
- Two example output switches (outputs 1 and 9). The board has 16 outputs; this file does not configure all 16.

This is a sanitized copy of my supplied configuration. The button scripts are preserved.

## Timing: what is immediate and what waits?

Events use the Home Assistant event type `esphome.button_action`:

```yaml
device: kc868_a16
input: "1"
action: button_1_pressed
```

| Action | When it is emitted | Typical use |
| --- | --- | --- |
| `button_X_pressed` | On the input press | Immediate feedback or input to a dimmer flow |
| `button_X_released` | On every release, including after a hold | Stop dimming |
| `button_X_single` | After the click timer expires with one counted release | A single-click action |
| `button_X_double` | After the click timer expires with two counted releases | A second action |
| `button_X_triple` | After the click timer expires with three counted releases | A third action |
| `button_X_long` | After 1,500 ms of holding | Start a hold action |

`X` is 1–16. `input` is a string. Filter by both `device` and `input` when routing events.

The defaults are `mp_gap: 400ms`, `long_min: 1500ms`, and `debounce_ms: 0ms`. "Immediate" means no deliberate multi-click wait in the input handler; input scanning, network delivery and downstream processing still take time. Increase debounce if your physical switches bounce.

**Single/double/triple classification is not delay-free.** The finalizer restarts after each short release and waits 400 ms. For example, press at 0 ms and release at 100 ms produces `pressed` at 0, `released` at 100, and `single` at approximately 500 ms. Two quick clicks ending at 100 and 300 ms produce `double` at approximately 700 ms.

The advantage over putting every action behind a multi-click classifier is that the raw press and release remain available immediately. This is especially valuable for release-to-stop control. The classifier runs on the ESP32, but the Home Assistant event consumers still need a working connection.

There is a trade-off: acting on the first press means that action also happens when the user later completes a double click or a hold. If single, double and triple actions must be mutually exclusive, wait for the classified event. Do not wire both raw and classified events to the same toggle unless you deliberately want both actions.

## Hold to dim, release to stop

The YAML provides the events for this interaction; it does **not** send Zigbee commands or dim a lamp itself.

There are two integration choices:

1. **Use the existing Node-RED dimmer flow:** feed it `pressed`/`released` events mapped to its configured input values, or the corresponding binary sensor state changes. The flow performs its own short-press/hold detection. Do not also feed it the ESPHome `long` event as another press. Choose one input route to avoid duplicate commands.
2. **Use ESPHome's hold detection:** route `button_X_long` to start a brightness move, and `button_X_released` to stop it. Your automation must choose direction and handle the initial light state. The default start threshold is 1.5 seconds; release has no additional 400 ms wait.

My [Single button ZigBee2MQTT dimmer flow](https://flows.nodered.org/flow/b3aa835c9ead60b8ec96b63ae2e80595) handles short-press toggle and hold dimming. It uses `brightness_move` for a continuous brightness change instead of repeatedly sending small brightness steps from a loop.

For a supported light, the basic command pattern on `zigbee2mqtt/FRIENDLY_NAME/set` is:

```json
{"brightness_move": 40}
```

A negative rate moves down. On release:

```json
{"brightness_move": 0}
```

The light performs the ramp, so its smoothness does not depend on the timing of repeated application-level steps. The stop command still has to reach the light. Actual results depend on the bulb, firmware and Zigbee network; this is not a universal guarantee of perfect dimming. Plain `brightness_move` may not turn an off light on; the linked flow handles that interaction separately. See a [documented Zigbee2MQTT device example](https://www.zigbee2mqtt.io/devices/LED1623G12.html#movingstepping) and check your own light's support.

## Setup

1. Download `kincony.yaml` and `secrets.example.yaml` into your local ESPHome configuration directory.
2. Copy `secrets.example.yaml` to `secrets.yaml`, supply your own strong OTA password and a valid 32-byte base64 API encryption key. The placeholder strings are intentionally not usable credentials.
3. Confirm this hardware mapping matches your KC868-A16 revision before flashing. The file uses LAN8720, GPIO23/18 for MDC/MDIO, GPIO17 clock output, GPIO4/5 for I2C, inputs at `0x21`/`0x22`, and outputs at `0x24`/`0x25`.
4. Validate the configuration in ESPHome, build and install it for your device. DHCP replaces the original private static network settings; use a router reservation if needed.
5. Add the device to Home Assistant using your API encryption key. Enable **Allow the device to perform Home Assistant actions** in the ESPHome integration's device configuration so events can be sent. See [ESPHome API documentation](https://esphome.io/components/api/#homeassistant-event-action).
6. Listen for `esphome.button_action` in Home Assistant's event developer tools, and verify press, release, multi-click and hold on your actual buttons before attaching actions.

## Privacy and validation

The public YAML contains no copied password, API key, Wi-Fi credentials, private static IP, gateway, personal entity IDs or MQTT broker details. `!secret` names are references, not the secret values. Never upload your real `secrets.yaml`, compiled firmware, or an unreviewed full configuration folder. The local `.gitignore` excludes those common files.

Compared with the supplied file, this version removes the static IP block, enables API encryption through a secret reference, and clarifies that only two outputs are configured. It preserves everything from `globals:` onward byte-for-byte. See [VALIDATION.md](VALIDATION.md) for what was checked. No physical-board or Zigbee end-to-end test is claimed.
