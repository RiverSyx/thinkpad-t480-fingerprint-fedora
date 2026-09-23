# ThinkPad T480 Fingerprint Reader on Fedora

A straightforward guide to getting the Validity (`06cb:009a`) fingerprint sensor working reliably on Lenovo ThinkPad T480 laptops running Fedora Linux.

Fixes common issues: `Failed: 0401`, `NoSuchDevice`, `Resource busy`, and sensor failure after suspend.

---

## ⚠️ Step 0: BIOS Prerequisite (Required)

Before running commands in Linux, disable **Predesktop Authentication** in BIOS. If left enabled, the firmware claims the sensor at boot and causes `Exception: Failed: 0401`.

1. Shut down completely.
2. Turn on and press **F1** at the Lenovo logo to enter BIOS.
3. Navigate to **Security** → **Fingerprint**.
4. Set **Predesktop Authentication** to **Disabled**.
5. Press **F10** to save and exit.

---

## Installation

### 1. Install Driver Packages
Enable the COPR repository for `python-validity` and install the required components:

```bash
sudo dnf copr enable -y sneexy/python-validity
sudo dnf install -y open-fprintd fprintd-clients fprintd-clients-pam python3-validity
```

### 2. Download Firmware & Ensure Persistence
Create the runtime directory, download the sensor firmware, and ensure `/var/run/python-validity` persists across reboots:

```bash
sudo mkdir -p /var/run/python-validity
sudo validity-sensors-firmware
sudo cp /var/run/python-validity/*.xpfwext /usr/share/python-validity/
echo 'd /var/run/python-validity 0755 root root -' | sudo tee /etc/tmpfiles.d/python-validity.conf
```

### 3. Factory Reset the Sensor
Mask the service first so systemd doesn't claim the USB device while resetting:

```bash
sudo systemctl mask python3-validity
sudo systemctl stop python3-validity
sudo pkill -9 -f dbus-service
sudo python3 /usr/share/python-validity/playground/factory-reset.py
```
> **Note:** A successful reset produces no output. Any traceback indicates failure (see Troubleshooting below).

### 4. Enable Services & Configure Startup Order
Create a systemd override so `open-fprintd` always waits for `python3-validity` to finish initializing:

```bash
sudo mkdir -p /etc/systemd/system/open-fprintd.service.d
sudo tee /etc/systemd/system/open-fprintd.service.d/after-validity.conf << 'EOF'
[Unit]
After=python3-validity.service
Requires=python3-validity.service
EOF

sudo systemctl daemon-reload
sudo systemctl unmask python3-validity
sudo systemctl enable --now python3-validity
sudo systemctl add-wants multi-user.target open-fprintd.service
sleep 5
sudo systemctl restart open-fprintd
```

### 5. Fix Suspend & Resume
Add a systemd sleep hook so the sensor restarts properly after waking from sleep:

```bash
sudo tee /usr/lib/systemd/system-sleep/fprint_wakeup << 'EOF'
#!/bin/bash
if [ "$1" = "post" ]; then
    systemctl restart python3-validity
    systemctl restart open-fprintd
fi
EOF
sudo chmod +x /usr/lib/systemd/system-sleep/fprint_wakeup
```

---

## Enrollment & Usage

### 1. Enroll Your Fingerprint
Run the enrollment tool and swipe your finger when prompted:

```bash
fprintd-enroll
```

### 2. Enable Fingerprint Authentication for PAM (Login & Sudo)
Use Fedora's `authselect` tool to enable PAM fingerprint support:

```bash
sudo authselect enable-feature with-fingerprint
sudo authselect apply-changes
```

Test it in a new terminal:
```bash
sudo echo "Fingerprint working!"
```

*(Optional)* To increase the sudo password timeout so you don't have to scan your finger constantly, run `sudo visudo` and add:
```text
Defaults timestamp_timeout=60
```

---

## Troubleshooting

| Problem | Cause | Solution |
| :--- | :--- | :--- |
| `validity-sensors-firmware: Is a directory` | Missing runtime directory | `sudo mkdir -p /var/run/python-validity` |
| `factory-reset.py: Resource busy` | Service is locking the USB device | Run `sudo systemctl mask python3-validity && sudo pkill -9 -f dbus-service` and retry. |
| `factory-reset.py: Exception: Failed: 0401` or `0404` | BIOS Predesktop Auth enabled | Disable Predesktop Auth in BIOS (Step 0). Do a cold shutdown (wait 15s before powering on). |
| `fprintd-enroll: NoSuchDevice` | Driver still initializing | Run `journalctl -u python3-validity -n 20`. Once it shows `Manager is back online`, run `sudo systemctl restart open-fprintd`. |
| Fingerprint stops working after suspend | Sleep hook missing or not executable | Ensure the hook in Step 5 (`/usr/lib/systemd/system-sleep/fprint_wakeup`) exists and has `chmod +x`. |
