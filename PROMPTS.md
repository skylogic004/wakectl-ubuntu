# PROMPTS

These are the prompts I used with Claude (LLM) to generate `wakectl.sh`.

## Me

on ubuntu, what tool or app or script will help the user to disable devices from waking the computer from sleep? It should report the devices in a human-readable form, not just cryptic IDs.

## Claude

Several options, from most to least user-friendly:

**1. `wakeup` GUI-less approach: `/proc/acpi/wakeup` + udev (the underlying mechanism)**
Cryptic by default (`XHC`, `GLAN`, `PXSX`), which is the problem you're describing.

**2. `powertop`** (interactive, partial)
Shows device names in the Tunables tab, but it controls runtime power management, not wake-from-sleep. Not the right tool for this.

**3. Human-readable script (recommended)**
Resolves ACPI names and USB devices to product names and toggles them. Save as `wakectl`:

```bash
#!/usr/bin/env bash
# wakectl: list/toggle wake-capable devices with readable names
# usage: wakectl list | wakectl toggle <index> | wakectl off-all-usb

list() {
  echo "== USB / PCI devices (sysfs power/wakeup) =="
  i=0
  for w in /sys/devices/**/power/wakeup /sys/bus/{usb,pci}/devices/*/power/wakeup; do
    [ -f "$w" ] || continue
    d=$(dirname "$(dirname "$w")")
    st=$(cat "$w" 2>/dev/null) || continue
    [ -z "$st" ] && continue
    if [ -f "$d/product" ]; then
      name="$(cat "$d/manufacturer" 2>/dev/null) $(cat "$d/product")"
    elif [ -f "$d/vendor" ] && [ -f "$d/device" ]; then
      name=$(lspci -s "$(basename "$d")" 2>/dev/null | cut -d' ' -f2-)
    else
      continue
    fi
    printf "%-3s %-9s %-40s %s\n" "$i" "$st" "$name" "$w"
    i=$((i+1))
  done | sort -u -k4
}

case "$1" in
  list) list ;;
  toggle)
    path=$(list | awk -v n="$2" '$1==n {print $NF}')
    [ -z "$path" ] && { echo "bad index"; exit 1; }
    cur=$(cat "$path")
    [ "$cur" = enabled ] && new=disabled || new=enabled
    echo "$new" | sudo tee "$path" >/dev/null && echo "$path -> $new" ;;
  off-all-usb)
    for w in /sys/bus/usb/devices/*/power/wakeup; do
      echo disabled | sudo tee "$w" >/dev/null
    done ;;
  *) echo "usage: $0 list|toggle <index>|off-all-usb" ;;
esac
```

Needs `lspci` (`sudo apt install pciutils`). Note: indices from `list` are regenerated each run, so run `list` immediately before `toggle`.

**4. Quick one-liner for ACPI-level wake sources**
```bash
cat /proc/acpi/wakeup
```
Toggle with `echo XHC | sudo tee /proc/acpi/wakeup`. Map names to hardware via `/sys/bus/acpi/devices/*/path` and `physical_node`.

**Persistence:** none of these survive reboot. Make changes permanent with a udev rule, e.g.:

```
# /etc/udev/rules.d/90-disable-wake.rules
ACTION=="add", SUBSYSTEM=="usb", ATTR{idVendor}=="046d", ATTR{idProduct}=="c52b", ATTR{power/wakeup}="disabled"
```

Get the IDs for the rule from `lsusb`, which you can pair with the script's readable names.

**Debug tip:** to find what actually woke the machine, check `journalctl -b -1 | grep -i wake` or `cat /sys/power/pm_wakeup_irq` after resume.

There's no widely packaged GUI that does this well; the script above is the practical answer. I can extend it to generate the udev rules automatically from the selected devices if useful.

## Me

please do extend it to generate the udev rules automatically from the selected devices, make it user-friendly

## Claude

`wakectl` is ready. I syntax-checked it and ran `list` in my sandbox, which has no devices, so the toggle and save paths are untested on real hardware.

**Install**
```bash
sudo install -m 755 wakectl /usr/local/bin/wakectl
sudo apt install pciutils usbutils   # readable names
sudo wakectl
```

**What it does**
- Lists every USB and PCI device that can wake the machine, with its product or vendor name, its current Wake state (red = enabled, green = disabled) and its ID.
- Numbers you type are toggled immediately (`2 5 7`).
- `s` writes `/etc/udev/rules.d/90-disable-wake.rules` and reloads udev. Each rule has the device name as a comment above it.
- A "Saved" column shows which devices already have a rule. Existing rules are loaded on start, so you can re-run it to change them.
- Re-enabling a device removes its rule. Quitting with unsaved changes prompts you to save.
- `wakectl list` prints the table without root.

**Behaviours to know**
- USB rules match `idVendor:idProduct`, so two identical devices (e.g. two of the same mouse) are both affected.
- Disabling a root hub (type `USB hub`) stops everything behind it from waking the machine. The script warns you when you do this.
- PCI rules match the slot address (e.g. `0000:00:14.0`), which is stable unless you change hardware.
- Wake-on-LAN is a separate mechanism, so disabling the NIC here may not be enough. Also run `sudo ethtool -s <iface> wol d`, or set `ethtool` options in NetworkManager or netplan to persist it.
- Entries that appear only in `/proc/acpi/wakeup`, with no PCI or USB device behind them, aren't covered. Say if you need a systemd oneshot for those.

## Me

This script (`wakectl`) seems to have less options than those shown in `cat /proc/acpi/wakeup`. In particular, I don't see the "LID	  S4	*enabled   platform:PNP0C0D:00" item which is for waking when the laptop lid is opened. This should definitely be included in the `wakectl` script. How do we fix it?

## Claude

The root cause: `wakectl` only walked sysfs (`/sys/bus/usb` and `/sys/bus/pci`), while `LID` is an ACPI-level wake source from `/proc/acpi/wakeup` with no USB or PCI device behind it. I'll add `/proc/acpi/wakeup` as a third source, with its own persistence mechanism, since udev can't set these.

`wakectl` now reads `/proc/acpi/wakeup` as well as USB and PCI, so `LID` shows up as "Laptop lid (opening it wakes the machine)". I tested it against a fake `/proc/acpi/wakeup`: parsing, the generated unit and the save/reload round trip all work. I haven't run the real toggle on actual hardware.

Reinstall it with `sudo install -m 755 wakectl /usr/local/bin/wakectl`.

**What changed**
- **ACPI entries:** every row of `/proc/acpi/wakeup` is listed with Type `ACPI`, including entries that map to PCI devices (`XHC`, `GLAN`) and ones with no device attached.
- **Names:** PCI-backed entries get their `lspci` name. Common platform IDs get plain names (lid, power button, sleep button, PS/2 keyboard and mouse). Anything else shows as `ACPI device <id>`.
- **Toggling:** `/proc/acpi/wakeup` is a toggle, so the script reads the current state and writes the name only if it needs to change. It checks the result afterwards and reports a failure instead of claiming success.
- **Persistence:** udev can't set ACPI wake sources. Saving now also writes `/etc/systemd/system/wakectl-acpi.service`, a oneshot that disables each selected source at boot, and enables it. Saving with no ACPI entries removes the unit. The Saved column and the reload-on-start logic cover both mechanisms.

**Things to know**
- For devices that appear in both ACPI and PCI/USB sections (e.g. `XHC` and the xHCI controller), disabling one does not necessarily stop the other from waking the machine. If something still wakes it, disable both. The table footer says so.
- Some firmware resets ACPI wake state after resume. If the lid starts waking the machine again after a suspend cycle, add `WantedBy=suspend.target` to the unit, or tell me and I'll make the script do it.
