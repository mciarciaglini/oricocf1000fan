# Fix Fan Ramping on IT8622 / IT87-Based Mini Server

This guide explains how to fix constant fan ramp-up / ramp-down behaviour on a small server or mini PC that uses an **ITE IT8622 / IT8613E-family fan controller**.

On this device, Linux may not expose the fan controller by default. The BIOS/firmware can still control the fan, but the fan may constantly speed up and slow down. Installing the `it87` DKMS driver exposes the controller to Linux so the main fan can be set to a stable manual speed.

This was tested on a Proxmox/Debian system where the fan controller was detected as:

```text
it8622-isa-0a30
```

The main ramping fan was controlled by:

```text
fan2_input
pwm2
```

## Tested Result

| Setting | Value |
|---|---|
| OS | Proxmox / Debian |
| Sensor driver | `it87` |
| Driver source | `frankcrawford/it87` |
| Forced chip ID | `0x8622` |
| Detected chip | `it8622` |
| Main fan input | `fan2_input` |
| Main fan control | `pwm2` |
| Recommended PWM value | `220` |
| Approximate fan speed | `3300–3600 RPM` |

## Warning

Use this at your own risk.

Incorrect fan settings can cause overheating. Do not set low PWM values unless you are actively monitoring temperatures.

This guide uses a higher fan value to reduce risk and stop fan hunting/ramping.

---

# Installation

## 1. Install Required Packages

On Proxmox/Debian:

```bash
apt update
apt install -y git dkms build-essential pve-headers-$(uname -r) lm-sensors
```

If the Proxmox headers package fails, check your running kernel:

```bash
uname -r
```

Then install the matching Proxmox header package manually.

---

## 2. Install the IT87 DKMS Driver

```bash
cd /usr/src
git clone https://github.com/frankcrawford/it87.git it87-frank
cd it87-frank
make dkms
```

Check that DKMS installed the module:

```bash
dkms status it87
```

Expected output should look similar to:

```text
it87/..., 6.x.x-pve, x86_64: installed
```

Example working output:

```text
it87/v1.0-207-ga9eb249.20251226, 6.17.13-2-pve, x86_64: installed
```

---

## 3. Load the Driver

Try loading the module normally first:

```bash
modprobe it87
```

If you get this error:

```text
modprobe: ERROR: could not insert 'it87': No such device
```

load it using the forced chip ID:

```bash
modprobe it87 force_id=0x8622
```

Check the kernel log:

```bash
dmesg | tail -40
```

Expected result:

```text
it87: Found IT8622E chip at 0xa30
```

---

## 4. Confirm the Controller Is Visible

Run:

```bash
sensors
```

Look for a section similar to this:

```text
it8622-isa-0a30
Adapter: ISA adapter
fan2:        3000 RPM
fan3:         700 RPM
pwm1:          ...
pwm2:          ...
pwm3:          ...
```

If you see `it8622-isa-0a30`, the driver is working.

---

# Manual Testing

## 5. Find the Correct hwmon Path

Do not assume the controller will always be `/sys/class/hwmon/hwmon7`.

The `hwmon` number can change after reboot. For example, before reboot it may be:

```text
/sys/class/hwmon/hwmon7
```

and after reboot it may become:

```text
/sys/class/hwmon/hwmon6
```

Find it dynamically:

```bash
for h in /sys/class/hwmon/hwmon*; do
  if [ "$(cat "$h/name" 2>/dev/null)" = "it8622" ]; then
    echo "$h"
    ls "$h" | grep -E 'fan[0-9]_input|pwm[0-9]$|pwm[0-9]_enable'
  fi
done
```

Example output:

```text
/sys/class/hwmon/hwmon7
fan2_input
fan3_input
fan4_input
fan5_input
pwm1
pwm1_enable
pwm2
pwm2_enable
pwm3
pwm3_enable
pwm4
pwm4_enable
pwm5
pwm5_enable
```

In the manual testing examples below, replace `hwmon7` with whatever path your system shows.

---

## 6. Check Current Fan Speeds

```bash
for i in 2 3 4 5; do
  echo -n "fan${i}_input="
  cat /sys/class/hwmon/hwmon7/fan${i}_input
done
```

Example output:

```text
fan2_input=2960
fan3_input=708
fan4_input=0
fan5_input=0
```

On the tested device:

| Fan Input | Meaning |
|---|---|
| `fan2_input` | Main ramping fan |
| `fan3_input` | Secondary low-speed fan |
| `fan4_input` | Unused |
| `fan5_input` | Unused |

---

## 7. Check Current PWM Modes

```bash
for i in 1 2 3 4 5; do
  echo -n "pwm${i}_enable="
  cat /sys/class/hwmon/hwmon7/pwm${i}_enable
  echo -n "pwm${i}="
  cat /sys/class/hwmon/hwmon7/pwm${i}
done
```

Example output:

```text
pwm1_enable=1
pwm1=128
pwm2_enable=2
pwm2=50
pwm3_enable=2
pwm3=50
pwm4_enable=2
pwm4=127
pwm5_enable=2
pwm5=127
```

Typical meaning:

| Value | Meaning |
|---:|---|
| `1` | Manual control |
| `2` | Automatic / BIOS-style control |

On the tested system, `pwm2` was in automatic mode and was causing the fan to ramp up and down.

---

## 8. Set PWM2 Manually

On the tested device, the main ramping fan is controlled by `pwm2`.

Set `pwm2` to manual mode and apply a stable speed:

```bash
echo 1 > /sys/class/hwmon/hwmon7/pwm2_enable
echo 220 > /sys/class/hwmon/hwmon7/pwm2
```

Confirm the setting:

```bash
cat /sys/class/hwmon/hwmon7/pwm2_enable
cat /sys/class/hwmon/hwmon7/pwm2
cat /sys/class/hwmon/hwmon7/fan2_input
```

Expected output:

```text
1
220
roughly 3300–3600 RPM
```

Watch fan speed:

```bash
watch -n 1 'cat /sys/class/hwmon/hwmon7/fan2_input'
```

To return the fan to automatic BIOS/firmware control:

```bash
echo 2 > /sys/class/hwmon/hwmon7/pwm2_enable
```

---

## 9. PWM Value Reference

PWM values use a `0–255` scale.

| PWM Value | Approximate Percentage | Notes |
|---:|---:|---|
| `160` | 63% | Quiet/stable on some systems |
| `180` | 71% | Better cooling |
| `200` | 78% | Higher cooling |
| `220` | 86% | Recommended tested value |
| `255` | 100% | Full speed |

Recommended default:

```text
pwm2 = 220
```

On the tested system, `220` stopped the fan ramping and kept CPU package temperatures around **65–70°C** under light/moderate load.

---

# Make It Persistent

## 10. Make the IT87 Driver Load at Boot

Create the module load file:

```bash
echo "it87" > /etc/modules-load.d/it87.conf
```

Create the module options file:

```bash
echo "options it87 force_id=0x8622" > /etc/modprobe.d/it87.conf
```

Verify:

```bash
cat /etc/modules-load.d/it87.conf
cat /etc/modprobe.d/it87.conf
```

Expected output:

```text
it87
options it87 force_id=0x8622
```

---

## 11. Create the Fan Control Script

This script dynamically finds the `it8622` hwmon path, so it does not matter whether the device appears as `hwmon6`, `hwmon7`, `hwmon8`, etc.

It also waits up to 30 seconds for the device to appear. This is important because the fan controller may not be ready immediately at boot.

```bash
cat > /usr/local/sbin/set-it87-fan.sh <<'EOF'
#!/bin/bash
set -e

HWMON=""

# Wait up to 30 seconds for the it8622 hwmon device to appear
for attempt in $(seq 1 30); do
    for h in /sys/class/hwmon/hwmon*; do
        if [ "$(cat "$h/name" 2>/dev/null)" = "it8622" ]; then
            HWMON="$h"
            break 2
        fi
    done
    sleep 1
done

if [ -z "$HWMON" ]; then
    echo "it8622 hwmon device not found"
    exit 1
fi

# Set PWM2 to manual mode and apply preferred fan speed
echo 1 > "$HWMON/pwm2_enable"
echo 220 > "$HWMON/pwm2"

echo "Set $HWMON pwm2_enable=1 pwm2=220"
EOF

chmod +x /usr/local/sbin/set-it87-fan.sh
```

Test it manually:

```bash
/usr/local/sbin/set-it87-fan.sh
```

Expected output:

```text
Set /sys/class/hwmon/hwmonX pwm2_enable=1 pwm2=220
```

Confirm the setting:

```bash
for h in /sys/class/hwmon/hwmon*; do
  if [ "$(cat "$h/name" 2>/dev/null)" = "it8622" ]; then
    echo "$h"
    echo -n "pwm2_enable="
    cat "$h/pwm2_enable"
    echo -n "pwm2="
    cat "$h/pwm2"
    echo -n "fan2_input="
    cat "$h/fan2_input"
  fi
done
```

Expected output:

```text
/sys/class/hwmon/hwmonX
pwm2_enable=1
pwm2=220
fan2_input=roughly 3300–3600
```

---

## 12. Create the systemd Service

Create the service file:

```bash
cat > /etc/systemd/system/set-it87-fan.service <<'EOF'
[Unit]
Description=Set IT8622 fan speed
After=systemd-modules-load.service
Wants=systemd-modules-load.service

[Service]
Type=oneshot
ExecStart=/usr/local/sbin/set-it87-fan.sh
RemainAfterExit=yes

[Install]
WantedBy=default.target
EOF
```

Reload systemd:

```bash
systemctl daemon-reload
```

Enable the service:

```bash
systemctl enable set-it87-fan.service
```

Start it immediately:

```bash
systemctl start set-it87-fan.service
```

Check status:

```bash
systemctl status set-it87-fan.service --no-pager
```

Expected result:

```text
Active: active (exited)
```

Check logs:

```bash
journalctl -u set-it87-fan.service --no-pager -b
```

Expected log line:

```text
Set /sys/class/hwmon/hwmonX pwm2_enable=1 pwm2=220
```

---

## 13. Confirm After Reboot

Reboot the system:

```bash
reboot
```

After it comes back up, run:

```bash
for h in /sys/class/hwmon/hwmon*; do
  if [ "$(cat "$h/name" 2>/dev/null)" = "it8622" ]; then
    echo "$h"
    echo -n "pwm2_enable="
    cat "$h/pwm2_enable"
    echo -n "pwm2="
    cat "$h/pwm2"
    echo -n "fan2_input="
    cat "$h/fan2_input"
  fi
done
```

Expected output:

```text
/sys/class/hwmon/hwmonX
pwm2_enable=1
pwm2=220
fan2_input=roughly 3300–3600
```

Example confirmed working output:

```text
/sys/class/hwmon/hwmon6
pwm2_enable=1
pwm2=220
fan2_input=3515
```

If you see this instead:

```text
pwm2_enable=2
pwm2=50
```

then the driver loaded, but the fan service did not apply. Check the service status and logs:

```bash
systemctl status set-it87-fan.service --no-pager
journalctl -u set-it87-fan.service --no-pager -b
```

---

# Troubleshooting

## Show All Detected Sensors

```bash
sensors
```

## Find the IT8622 hwmon Path

```bash
for h in /sys/class/hwmon/hwmon*; do
  echo "==== $h ===="
  cat "$h/name" 2>/dev/null
  ls "$h" | grep -E 'fan|pwm|temp'
done
```

## Check IT87 Driver Messages

```bash
dmesg | grep -i it87
```

Expected:

```text
it87: Found IT8622E chip at 0xa30
```

## Check Whether the Module Is Loaded

```bash
lsmod | grep it87
```

## Check DKMS Status

```bash
dkms status it87
```

## Restart the Fan Service

```bash
systemctl restart set-it87-fan.service
```

## Check Fan Service Logs

```bash
journalctl -u set-it87-fan.service --no-pager
```

## Manually Reapply the Fan Setting

```bash
/usr/local/sbin/set-it87-fan.sh
```

---

# Reverting Changes

## Temporarily Return PWM2 to Automatic Control

```bash
for h in /sys/class/hwmon/hwmon*; do
  if [ "$(cat "$h/name" 2>/dev/null)" = "it8622" ]; then
    echo 2 > "$h/pwm2_enable"
  fi
done
```

## Disable the Boot Service

```bash
systemctl disable --now set-it87-fan.service
```

## Remove Persistent Module Loading

```bash
rm -f /etc/modules-load.d/it87.conf
rm -f /etc/modprobe.d/it87.conf
```

## Remove the Fan Script and Service

```bash
rm -f /usr/local/sbin/set-it87-fan.sh
rm -f /etc/systemd/system/set-it87-fan.service
systemctl daemon-reload
```

Reboot after removing the configuration:

```bash
reboot
```

---

# Final Known-Good Configuration

```text
Driver:        it87
Driver option: force_id=0x8622
Chip detected: it8622
Main fan:      fan2
Control:       pwm2
Mode:          manual
PWM speed:     220 / 255
Approx RPM:    3300–3600 RPM
```

Confirmed after reboot:

```text
/sys/class/hwmon/hwmon6
pwm2_enable=1
pwm2=220
fan2_input=3515
```
