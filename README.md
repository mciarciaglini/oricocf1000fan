# oricocf1000fan

# Fix Fan Ramping on IT8622 / IT87-Based Mini Server

This guide documents how to fix the constant fan ramp-up / ramp-down issue on a small server or mini PC that uses an **ITE IT8622 / IT8613E-family fan controller**.

The issue is usually that Linux does not expose the fan controller by default, so the BIOS/firmware keeps dynamically ramping the fan up and down. By installing the `it87` DKMS driver and setting the correct PWM channel manually, the fan can be stabilized.

## Tested Configuration

| Item | Value |
|---|---|
| OS | Proxmox / Debian |
| Sensor driver | `it87` |
| Driver source | `frankcrawford/it87` |
| Forced chip ID | `0x8622` |
| Detected chip | `it8622` |
| Main fan input | `fan2_input` |
| Main fan control | `pwm2` |
| Recommended PWM value | `220` |
| Approximate fan speed | ~3300–3600 RPM |

## Disclaimer

Use this at your own risk.

Fan control mistakes can cause overheating. Do not set low PWM values unless you are actively monitoring temperatures. This guide uses a higher fan value to reduce risk.

Before making anything persistent, test manually first.

---

## 1. Install Required Packages

On Proxmox/Debian:

```bash
apt update
apt install -y git dkms build-essential pve-headers-$(uname -r) lm-sensors
