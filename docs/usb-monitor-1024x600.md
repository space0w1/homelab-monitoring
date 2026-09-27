# USB portable monitor stuck at 800x600

The small USB monitor (1024x600 panel, MacroSilicon MS9132 chip, USB ID `345f:9132`)
comes up at 800x600 on Linux. It shows up in `xrandr` as `HDMI-1-3`.

## Fix

```
echo 'options usbdisp_usb custom_mode=78_1024x600@60' | sudo tee /etc/modprobe.d/usbdisp.conf
sudo modprobe -r usbdisp_usb; sudo modprobe usbdisp_usb
sleep 3; xrandr --output HDMI-1-3 --mode 1024x600
```

The config file persists, so after a reboot the monitor should come up at 1024x600 on its own.

Run these in a normal terminal. Don't prefix them with `!`: in zsh that inverts the exit
status and breaks `&&` chains, which leaves the driver unloaded.

## Check it's working

```
cat /sys/module/usbdisp_usb/parameters/custom_mode    # -> 78_1024x600@60
sudo dmesg | grep 'color out' | tail -1                # -> width:1024 height:600 vic:78
xrandr | grep -A2 HDMI-1-3                             # -> 1024x600 60.00*+
```

## Why it happens

- The driver is MacroSilicon's vendor DRM driver (`usbdisp_drm` + `usbdisp_usb`),
  built with DKMS from `~/ms91xx-linux-drm`.
- The monitor's EDID does advertise 1024x600@60. The driver rejects it because the chip
  can only output timings from a fixed built-in table (VIC numbers), and 1024x600 isn't in it.
- The driver's `custom_mode=<vic>_<W>x<H>@<Hz>` parameter maps a resolution to one of those
  built-in timings. VIC 78 is the chip's 1280x600 timing: it carries the 1024x600 picture,
  and the monitor accepts it. The mapping only lives in the driver's memory; nothing is
  written to the chip's flash.
- `custom_mode` is only read when the module loads, so unplugging and replugging the monitor
  doesn't apply it. Reload the module or reboot.

## What didn't work

| Attempt | Result |
|---|---|
| 1024x768 (VIC 71) via `xrandr --addmode` | Monitor shows lines, no picture (its scaler rejects the signal) |
| `custom_mode=66_1024x600@60` (800x600 timing) | Chip crops instead of scaling, so only part of the screen shows |
| Writing a custom timing to the chip's flash | Not attempted. Undocumented and could permanently break the chip. If ever needed, use the vendor's Windows tool. |

## If it breaks again

1. Check that `/etc/modprobe.d/usbdisp.conf` still exists and says `78_1024x600@60`.
2. After a kernel update, check that DKMS rebuilt the driver: `dkms status | grep msdisp`.
3. Check whether the module is loaded (`lsmod | grep usbdisp`); if not, run `sudo modprobe usbdisp_usb`.
4. If `xrandr` says "cannot find mode 1024x600", `custom_mode` wasn't applied. Reload the module.
5. Undo everything: `sudo rm /etc/modprobe.d/usbdisp.conf`, then reload the module.
