# monitor

Readable Linux hardware monitoring dashboard for temperatures and fan speeds.

This project wraps `lm-sensors` JSON output and hwmon/sysfs PWM data into a clearer terminal view than raw `sensors`.

## Requirements

- Linux with `lm-sensors` installed (`sensors` command available)
- Python 3.10+
- Access to `/sys/class/hwmon` (default on local system)

## Project Structure

```text
.
├── .gitignore
├── monitor
├── README.md
└── sensor_thresholds.json
```

## Files

- `monitor`
  - Executable Python CLI (no extension).
  - Inputs:
    - `sensors -j` output from `lm-sensors`
    - `/sys/class/hwmon/*` files for PWM control values and device identification
    - `sensor_thresholds.json` for fallback threshold rules
  - Outputs:
    - One-shot terminal dashboard
    - Live dashboard with in-place redraw (`--watch`)
  - Main features:
    - Temperature table (sorted hottest first)
    - Fan table with live RPM and PWM control percentage
    - Device name resolution (CPU model, NVMe model, nct6686, etc.)
    - Warn/Crit thresholds from real sensor data, with fallback overrides
    - Threshold-colored temperature bars:
      - Green: below warn
      - Orange: warn reached
      - Red: at or above 90% of crit

- `sensor_thresholds.json`
  - Threshold override database used when sensor limits are missing.
  - Rule matching uses regex for `device` and `sensor`.
  - `prefer_reported: true` means "keep real sensor values when present; only fill gaps."
  - `prefer_reported: false` forces the override value to replace sensor-provided values.

- `.gitignore`
  - Ignores Python cache artifacts.

## Data Pipeline

1. `monitor` executes `sensors -j`.
2. It parses all `temp*_input` and `fan*_input` entries.
3. It resolves device names from hwmon sysfs metadata.
4. It reads PWM control channels (`pwmX`) from `/sys/class/hwmon` and maps them to `fanX`.
5. It loads `sensor_thresholds.json` and applies matching threshold fallback rules.
6. It renders terminal tables with aligned columns and load bars.
7. In watch mode, it redraws in place without flooding scrollback.

## Usage

Run once:

```bash
./monitor
```

Disable ANSI colors:

```bash
./monitor --no-color
```

Live mode (refresh every second):

```bash
./monitor --watch 1
```

## Live Mode Behavior

- Uses terminal alternate screen buffer for a clean dashboard view.
- Redraws in-place (no continuous scrolling output).
- Exit with `Ctrl+C`.

## Threshold Overrides

Edit `sensor_thresholds.json` to tune Warn/Crit values.

Each override supports:

- `name`: human-readable label
- `device_regex`: regex matched against resolved device name
- `sensor_regex`: regex matched against sensor label
- `warn`: warning threshold (Celsius)
- `crit`: critical threshold (Celsius)
- `prefer_reported`: whether to preserve real sensor-provided limits

Example rule:

```json
{
  "name": "Example Device Rule",
  "device_regex": "^Some Device$",
  "sensor_regex": "^Sensor 1$",
  "warn": 70.0,
  "crit": 85.0,
  "prefer_reported": true
}
```

## Notes

- Not all sensors export meaningful min/max/crit values.
- Some drivers report placeholders (for example `0`, `-273.15`, or unrealistic large sentinels).
- This tool filters invalid values and uses overrides to fill missing limits where appropriate.

