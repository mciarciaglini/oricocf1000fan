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

If the Proxmox headers package fails, check your running kernel:

uname -r

Then install the matching header package manually.

2. Install the IT87 DKMS Driver
cd /usr/src
git clone https://github.com/frankcrawford/it87.git it87-frank
cd it87-frank
make dkms

Check that DKMS installed the module:

dkms status it87

Expected output should look similar to:

it87/..., 6.x.x-pve, x86_64: installed
3. Load the Driver

Try loading the module normally first:

modprobe it87

If you get this error:

modprobe: ERROR: could not insert 'it87': No such device

load it using the forced chip ID:

modprobe it87 force_id=0x8622

Check the kernel log:

dmesg | tail -40

Expected result:

it87: Found IT8622E chip at 0xa30
4. Confirm the Controller Is Visible

Run:

sensors

Look for a section similar to:

it8622-isa-0a30
Adapter: ISA adapter
fan2:        3000 RPM
fan3:         700 RPM
pwm1:          ...
pwm2:          ...
pwm3:          ...

If you see it8622-isa-0a30, the driver is working.

5. Find the Correct hwmon Path

Do not assume the controller will always be /sys/class/hwmon/hwmon7.

Find it dynamically:

for h in /sys/class/hwmon/hwmon*; do
  if [ "$(cat "$h/name" 2>/dev/null)" = "it8622" ]; then
    echo "$h"
    ls "$h" | grep -E 'fan[0-9]_input|pwm[0-9]$|pwm[0-9]_enable'
  fi
done

Example output:

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

In the rest of the guide, replace hwmon7 with whatever path your system shows.

6. Check Current Fan Speeds
for i in 2 3 4 5; do
  echo -n "fan${i}_input="
  cat /sys/class/hwmon/hwmon7/fan${i}_input
done

Example output:

fan2_input=2960
fan3_input=708
fan4_input=0
fan5_input=0

On the tested device:

Fan Input	Meaning
fan2_input	Main ramping fan
fan3_input	Secondary low-speed fan
fan4_input	Unused
fan5_input	Unused
7. Test PWM2 Manually

On the tested device, the main ramping fan is controlled by pwm2.

Set pwm2 to manual mode and apply a high stable speed:

echo 1 > /sys/class/hwmon/hwmon7/pwm2_enable
echo 220 > /sys/class/hwmon/hwmon7/pwm2

Confirm the setting:

cat /sys/class/hwmon/hwmon7/pwm2_enable
cat /sys/class/hwmon/hwmon7/pwm2
cat /sys/class/hwmon/hwmon7/fan2_input

Expected output:

1
220
roughly 3300–3600 RPM

Watch fan speed:

watch -n 1 'cat /sys/class/hwmon/hwmon7/fan2_input'

To return the fan to automatic BIOS/firmware control:

echo 2 > /sys/class/hwmon/hwmon7/pwm2_enable
8. PWM Value Reference

PWM values use a 0–255 scale.

PWM Value	Approximate Percentage
160	63%
180	71%
200	78%
220	86%
255	100%

Recommended values:

Goal	Value
Quiet but stable	180
Better cooling	220
Full speed	255

The tested system used:

pwm2 = 220

This stopped the fan ramping and kept CPU package temperatures around 65–70°C under light/moderate load.

9. Make the IT87 Driver Persistent

Create the module load file:

echo "it87" > /etc/modules-load.d/it87.conf

Create the module options file:

echo "options it87 force_id=0x8622" > /etc/modprobe.d/it87.conf

Verify:

cat /etc/modules-load.d/it87.conf
cat /etc/modprobe.d/it87.conf

Expected output:

it87
options it87 force_id=0x8622
10. Create a Fan Control Boot Script

Create the script:

cat > /usr/local/sbin/set-it87-fan.sh <<'EOF'
#!/bin/bash
set -e

# Find the hwmon path for the it8622 sensor chip
for h in /sys/class/hwmon/hwmon*; do
    if [ "$(cat "$h/name" 2>/dev/null)" = "it8622" ]; then
        HWMON="$h"
        break
    fi
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

Make it executable:

chmod +x /usr/local/sbin/set-it87-fan.sh

Test it manually:

/usr/local/sbin/set-it87-fan.sh

Expected output:

Set /sys/class/hwmon/hwmonX pwm2_enable=1 pwm2=220
11. Create a systemd Service

Create the service file:

cat > /etc/systemd/system/set-it87-fan.service <<'EOF'
[Unit]
Description=Set IT8622 fan speed
After=multi-user.target
ConditionPathExists=/usr/local/sbin/set-it87-fan.sh

[Service]
Type=oneshot
ExecStartPre=/bin/sleep 5
ExecStart=/usr/local/sbin/set-it87-fan.sh
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
EOF

Enable and start the service:

systemctl daemon-reload
systemctl enable set-it87-fan.service
systemctl start set-it87-fan.service

Check status:

systemctl status set-it87-fan.service --no-pager

Expected result:

Active: active (exited)
12. Confirm After Reboot

Reboot the system:

reboot

After it comes back up, run:

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

Expected output:

/sys/class/hwmon/hwmonX
pwm2_enable=1
pwm2=220
fan2_input=roughly 3300–3600
13. Useful Troubleshooting Commands

Show all detected sensors:

sensors

Find the IT8622 hwmon path:

for h in /sys/class/hwmon/hwmon*; do
  echo "==== $h ===="
  cat "$h/name" 2>/dev/null
  ls "$h" | grep -E 'fan|pwm|temp'
done

Check IT87 driver messages:

dmesg | grep -i it87

Check module status:

lsmod | grep it87

Check DKMS status:

dkms status it87

Restart fan service:

systemctl restart set-it87-fan.service

Check fan service logs:

journalctl -u set-it87-fan.service --no-pager
14. Revert Changes

To temporarily return PWM2 to automatic control:

for h in /sys/class/hwmon/hwmon*; do
  if [ "$(cat "$h/name" 2>/dev/null)" = "it8622" ]; then
    echo 2 > "$h/pwm2_enable"
  fi
done

To disable the boot service:

systemctl disable --now set-it87-fan.service

To remove the persistent module loading:

rm -f /etc/modules-load.d/it87.conf
rm -f /etc/modprobe.d/it87.conf

To remove the fan script and service:

rm -f /usr/local/sbin/set-it87-fan.sh
rm -f /etc/systemd/system/set-it87-fan.service
systemctl daemon-reload

Reboot after removing the configuration:

reboot
Final Known-Good Configuration
Driver:        it87
Driver option: force_id=0x8622
Chip detected: it8622
Main fan:      fan2
Control:       pwm2
Mode:          manual
PWM speed:     220 / 255
Approx RPM:    3300–3600 RPM


