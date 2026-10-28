# wakectl

List and disable wake-from-sleep sources on Linux (ACPI, USB, PCI) by readable device name, with persistence across reboots.

`/proc/acpi/wakeup` shows entries like `XHC`, `GLAN`, `LID`. `wakectl` resolves them to names like "Laptop lid (opening it wakes the machine)" or the `lspci`/USB product string, lets you toggle them by number, and saves your choices so they survive a reboot.

```
#   Wake      Saved  Type     Device                                                 ID
0   disabled         ACPI     ACPI wake source (no device attached)                  PEG0/S4
1   enabled          ACPI     Ethernet controller [...] I219-V                       GLAN/S4
2   enabled   yes    ACPI     Laptop lid (opening it wakes the machine)              LID/S4
3   enabled          USB      Logitech USB Receiver                                  046d:c52b
```

## Install

```bash
sudo apt install pciutils usbutils    # lspci / lsusb for readable names
sudo install -m 755 wakectl /usr/local/bin/wakectl
```

Requires bash 4+. Tested on Ubuntu; should work on any systemd-based distro.

## Usage

```bash
sudo wakectl      # interactive
wakectl list      # print the table and exit (no root needed)
```

Interactive commands:

| Input        | Action                                         |
|--------------|------------------------------------------------|
| `2 5 7`      | Toggle those devices (applied immediately)     |
| `s`          | Save choices (persist across reboots)          |
| `r`          | Rescan                                         |
| `q`          | Quit (prompts if there are unsaved changes)    |

Toggling changes the live state only. Press `s` to persist.

## Sources and persistence

| Source                  | Read from                        | Persisted as                                         |
|-------------------------|----------------------------------|------------------------------------------------------|
| ACPI                    | `/proc/acpi/wakeup`              | `/etc/systemd/system/wakectl-acpi.service` (oneshot) |
| USB                     | `/sys/bus/usb/devices/*/power/wakeup` | udev rule in `/etc/udev/rules.d/90-disable-wake.rules`, matched on `idVendor:idProduct` |
| PCI                     | `/sys/bus/pci/devices/*/power/wakeup` | same udev file, matched on slot address         |

Re-running `wakectl` loads existing rules, so the Saved column reflects what is on disk. Toggling a saved device back to enabled removes its rule.

To undo everything: delete `/etc/udev/rules.d/90-disable-wake.rules`, and run `sudo systemctl disable --now wakectl-acpi.service && sudo rm /etc/systemd/system/wakectl-acpi.service`.

## Caveats

- ACPI and PCI/USB entries are separate switches. A device listed in both (e.g. `XHC` and the xHCI controller) may need both disabled.
- USB rules match vendor:product, so identical devices (two of the same mouse) are affected together.
- Disabling a USB root hub stops every device behind it from waking the machine. The script warns you.
- Wake-on-LAN is configured separately: `sudo ethtool -s <iface> wol d`.
- Some firmware resets ACPI wake state after resume. If the lid starts waking the machine again, add `WantedBy=suspend.target` to the generated unit.
- ACPI entries with no attached device are listed but have no readable name.

## Debugging what woke the machine

```bash
journalctl -b -1 | grep -i wake
cat /sys/power/pm_wakeup_irq     # after resume
```
