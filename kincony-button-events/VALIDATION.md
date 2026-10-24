# Validation of this sharing copy

Reviewed 25 October 2026.

- ESPHome **2026.3.2** configuration validation passed, using temporary dummy credentials outside this published folder.
- ESPHome reported the expected warning that GPIO5 is a strapping pin. This is the supplied board's I2C mapping; verify wiring and board revision before use.
- The entire button implementation from `globals:` onward matches the supplied YAML byte-for-byte: 16 input handlers, 32 scripts and 48 globals.
- Each input's normalized press/release handler and timer scripts were checked for matching structure across all 16 inputs.
- The public YAML was checked for embedded IP addresses and credential fields. OTA and API credentials are secret references. The example secrets contain only obvious replacement placeholders.
- No real `secrets.yaml`, source download, build products or validation output are included in this folder.
- The timing description was checked against the scripts. The finalizer's unconditional count reset explains the slower-second-press limitation documented in the README.

This is configuration and source-level validation. The firmware was not compiled or flashed as part of this review, and no physical button, network timing or lamp test was performed. Validation with ESPHome 2026.3.2 does not establish compatibility with every later release.
