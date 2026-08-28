# WaterCoolerCLI

## Overview

WaterCoolerCLI is a Linux command-line tool for controlling and monitoring compatible AORUS water coolers. The official AORUS software does not provide Linux support, so this project implements the required HID communication through reverse engineering.

The project is experimental and interacts directly with hardware. Use it at your own risk. The core protocol is not officially documented, and compatibility can vary between cooler models and firmware versions.

## Features

- **Fan and pump control**
  - Read and set fan and pump modes.
  - Read current fan and pump speeds.
  - Read and set custom fan and pump curves.

- **Monitoring**
  - Display fan and pump information through the CLI.
  - Read CPU temperature for monitoring and service mode.
  - Run a background service that periodically sends host CPU telemetry to the cooler LCD.

- **Visualization**
  - Plot fan and pump curves as an ASCII graph.

- **Interactive CLI**
  - Tab completion for commands and options.
  - Command history with arrow keys.
  - Built-in help through the `help` command.

- **Supported devices**
  - The device discovery list currently includes `VID:PID 1044:7A51`, `1044:7A4D`, and `0414:7A5E`.
  - The FW 2.0 telemetry layout was validated on an AORUS WATERFORCE X II 360I (`0414:7A5E`).

## Current host telemetry scope

The service currently reads and sends only the following host-side CPU values. It does **not** claim to decode every telemetry value available from the cooler or its HID interface.

- CPU model name, sent during service startup.
- CPU package temperature, read through the existing CPU temperature handler.
- CPU frequency, read from `cpufreq/policy0/scaling_cur_freq` when available, with a per-core sysfs fallback.
- Global CPU usage, calculated from deltas in `/proc/stat`.
- CPU package power, calculated from an energy-counter delta:
  - generic Linux `powercap` package zones exposing `energy_uj`; or
  - AMD `amd_energy` HWMON socket counters labeled `Esocket` when the generic interface is unavailable.

The CPU frequency is encoded to the LCD with one decimal place and rounded at the HID serialization boundary. For example, `2999 MHz` is transmitted as `3.0 GHz` instead of being truncated to `2.9 GHz`.

The first CPU usage and power samples establish a baseline. Their initial values can therefore be zero until the next service cycle.

## Firmware 2.0 telemetry layout

For the validated AORUS `0414:7A5E` device, the runtime telemetry payload after the report prefix uses the following fields:

```text
E0 00 00 TEMP 00 FREQ_INT FREQ_DEC 00 00 00 USAGE POWER_H POWER_L
```

The complete HID report remains padded according to the device's report size. This layout is specific to the tested firmware/device combination and should not automatically be assumed to apply to every AORUS cooler.

## Requirements

- Linux with HID access.
- .NET 10 SDK for building.
- Native build tools and the platform HID development package when required by the distribution.
- The `HidSharp` NuGet dependency, restored automatically by .NET.

On Ubuntu/Debian derivatives, the usual build prerequisites are:

```bash
sudo apt install build-essential libhidapi-dev
```

On Fedora-based systems:

```bash
sudo dnf install gcc-c++ hidapi-devel
```

The service normally requires root privileges because it writes HID reports to the cooler. Read-only CLI operations may work as a regular user depending on the installed udev rules.

## Building

Run the command from the repository root:

```bash
dotnet publish -c Release -r linux-x64 \
  --self-contained true \
  -p:PublishAot=true \
  -p:PublishTrimmed=true \
  src/WaterCoolerCLI.sln
```

The published executable is generated under:

```text
src/WaterCoolerCLI/bin/Release/net10.0/linux-x64/publish/WaterCoolerCLI
```

The project uses a native AOT, self-contained, trimmed build. `HidSharp` may emit the grouped .NET trim warning `IL2104` because that dependency is not fully annotated for trimming. The warning is currently known and does not prevent a successful build.

For other architectures, change the runtime identifier, for example:

```bash
-r linux-arm64
```

If trimming causes a runtime issue during investigation, temporarily remove `-p:PublishTrimmed=true` to compare behavior.

## Usage

### Interactive mode

Run without arguments to open the interactive CLI:

```bash
sudo ./WaterCoolerCLI
```

Type `help` to list the available commands. Examples include:

```text
get-fan-mode
set-fan-mode Turbo
set-fan-curve "0:1000,30:1500,50:2000,65:2500"
monitor 1000
plot-curves
quit
```

### Single-command mode

```bash
sudo ./WaterCoolerCLI get-speeds
./WaterCoolerCLI get-fan-mode
```

### Service mode

The `service` command periodically reads the host CPU telemetry listed above and sends it to the cooler LCD. The default update interval is 500 ms.

Test manually:

```bash
sudo ./WaterCoolerCLI service
```

Press `q` to stop the interactive service process.

A basic systemd unit can be created at `/etc/systemd/system/watercooler.service`:

```ini
[Unit]
Description=AORUS Water Cooler Telemetry Service
After=local-fs.target

[Service]
Type=simple
User=root
ExecStart=/usr/local/bin/WaterCoolerCLI service
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

Enable and start it with:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now watercooler.service
sudo systemctl status watercooler.service
```

View logs with:

```bash
journalctl -u watercooler.service -f
```

## Known limitations and future work

- [ ] Inventory the remaining safe, read-only HID telemetry reports and document which values can be used in future LCD updates. This is intentionally separate from the host CPU telemetry currently implemented.
- Compatibility with other models and firmware versions is not guaranteed.
- The runtime telemetry protocol is reverse engineered and should be validated with captured reports before adding new fields.
- The native AOT build may continue to report trim diagnostics from third-party HID dependencies.

## Troubleshooting

- **Device not found:** Check the USB device with `lsusb` and confirm the VID/PID is included in the discovery list.
- **HID access denied:** Add an appropriate udev rule or run the operation with `sudo`.
- **CPU temperature unavailable:** Verify the available Linux sensor interfaces and confirm that the CPU temperature handler can read them.
- **CPU power shows zero initially:** The first power sample initializes the energy-counter baseline; wait for the next service cycle.
- **Build warning IL2104:** This is a grouped trim warning produced by `HidSharp`; it is not a compiler error.
- **Firmware compatibility:** Do not assume that a telemetry layout validated on one model or firmware applies to another without a read-only protocol comparison.

## Contributing

Contributions are welcome. Before changing a HID payload, document the exact device model, VID/PID, firmware response, report framing, and the evidence used to map each field. Prefer read-only probes and reversible tests.

Potential contribution areas include:

- Improved cross-distribution support.
- Additional compatible devices.
- Better logging and error handling.
- Safe documentation of additional telemetry reports.

## License

This project is open source under the GPL-3.0 license.

---

*Disclaimer: This tool communicates directly with hardware. The project authors are not responsible for damage to the cooler, computer, or data. Test changes carefully and keep a known-good build available.*
