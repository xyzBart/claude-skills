---
name: bluetooth-resume-fix
description: Use when Bluetooth doesn't work after resume/suspend on uzer-Latitude-5420 (Dell Latitude 5420, Intel AX201 CNVi, s2idle-only). Symptom is bluetooth.service/bluetoothd running fine but no hci0 controller. Covers diagnosis and the live no-reboot fix (xhci_hcd driver rebind).
version: 1.0.0
allowed-tools: Bash, Read
---

# Bluetooth failing after resume (Dell Latitude 5420 / Intel AX201)

Machine: uzer-Latitude-5420, Intel AX201 Wi-Fi/BT CNVi combo. This laptop only
supports `s2idle` (`cat /sys/power/mem_sleep` → `[s2idle]`, no `deep`/S3) —
Dell's "Modern Standby". The internal Bluetooth radio's USB device sometimes
doesn't survive the resume, and this is the recurring, rare (roughly
monthly-or-less) failure mode.

## Symptom

Bluetooth "isn't working" after waking from suspend. `bluetooth.service` /
`bluetoothd` looks perfectly healthy (active, running) — that's a red
herring. The actual radio is gone.

## Diagnose

```bash
bluetoothctl show                 # "No default controller available"
ls /sys/class/bluetooth/          # empty
rfkill list bluetooth             # not blocked — rules out rfkill as cause
lsusb -t                          # no 8087:0026 device present
```

Confirm it's a USB enumeration failure, not a firmware/protocol error, by
checking the kernel log for the affected boot:

```bash
journalctl -k -b --no-pager | grep -iE 'bluetooth|hci0|btusb|3-10'
```

Two failure signatures have been seen on this machine, both same root cause
(the CNVi BT USB endpoint, `idVendor=8087 idProduct=0026`, doesn't reliably
survive s2idle resume) — the port/bus number can shift between boots, check
`lsusb -t` at boot time for the actual bus/port if these commands don't match:

- **Mid-operation failure** (seen 2026-07-11): `sending frame failed (-19)`,
  `Opcode 0x... failed: -19`, `HCI reset during shutdown failed`. The USB
  device was present but the xHCI controller/port wedged entirely — this one
  needed a full reboot.
- **Enumeration failure** (seen 2026-08-20): port never comes back at all —
  ```
  usb 3-10: device descriptor read/64, error -71
  usb usb3-port10: attempt power cycle
  usb 3-10: device not accepting address ..., error -71
  usb usb3-port10: unable to enumerate USB device
  ```
  Fixed live, no reboot needed (see below).

Identify which PCI xHCI controller owns the affected USB bus:

```bash
readlink -f /sys/bus/usb/devices/usb3   # -> .../pci0000:00/0000:00:14.0/usb3
lspci -k | grep -A3 -i usb              # confirm 00:14.0 is xhci_hcd
```

On this machine bus 3 + bus 4 (which carry BT, webcam, smartcard reader) are
both on PCI `0000:00:14.0` (Tiger Lake-LP xHCI). Bus 1/2 are a separate
Thunderbolt xHCI at `0000:00:0d.0` and don't carry BT. The laptop's own
keyboard/trackpad are not USB (I2C HID), so resetting `00:14.0` doesn't
interrupt input — only whatever's plugged into USB-A/C ports plus the
internal webcam/smartcard reader blip briefly.

## Fix: live recovery, no reboot (try this first)

Unbind and rebind just the `xhci_hcd` **driver** for the PCI device. This
keeps the PCI device itself intact and makes the driver core do a full
re-probe (which includes its own host-controller reset) — much gentler than
a PCI-level remove/rescan:

```bash
echo 0000:00:14.0 | sudo tee /sys/bus/pci/drivers/xhci_hcd/unbind
sleep 2
echo 0000:00:14.0 | sudo tee /sys/bus/pci/drivers/xhci_hcd/bind
sleep 3
lsusb -t          # expect the 8087:0026 port back, bound to btusb
bluetoothctl show # expect a Controller line, Powered: yes
```

This has fixed the enumeration-failure case in ~5 seconds with no visible
disruption beyond a brief webcam/smartcard-reader blip.

## Fix: full reboot (fallback)

If the driver rebind doesn't bring `hci0` back, or the kernel log shows the
xHCI controller itself failed to reinit (`Host halt failed, -19`, `init
0000:00:14.0 fail, -19`), a reboot is required. Consider also refreshing
firmware/kernel while you're at it, since a stale `linux-firmware` build was
plausibly implicated in the July incident:

```bash
sudo apt update
sudo apt install --only-upgrade linux-generic-hwe-24.04 linux-image-generic-hwe-24.04 linux-headers-generic-hwe-24.04 linux-firmware
```

## What NOT to do

Do **not** use a raw PCI-level remove/rescan of the xHCI controller as a
"lighter than reboot" attempt:

```bash
# DON'T — confirmed to make things worse on this machine (2026-07-11)
echo 1 > /sys/bus/pci/devices/0000:00:14.0/remove
echo 1 > /sys/bus/pci/rescan
```

This tears down and rebuilds the PCI device's config space rather than just
re-probing the driver, and previously caused the controller to fail
reinitialization entirely (`Host halt failed, -19`), taking down bus 3 *and*
bus 4 together (BT, webcam, smartcard reader, any plugged-in USB devices) and
requiring a reboot anyway. The driver unbind/bind approach above is the
correct lighter-weight alternative — always try that first.
