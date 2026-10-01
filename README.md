# OpenMouse

OpenMouse is a browser-based control panel for supported gaming mice.

Connect a mouse, view its information, and change supported settings such as DPI
and polling rate without installing a different app for every brand.

This branch is deployed as the public development control panel.

## Development

```bash
npm install
npm run dev
```

Run the full local check before pushing changes:

```bash
npm run check
```

### Linux WebHID permissions

If a device appears in Chromium's picker but OpenMouse reports `Failed to open
the device`, check the permissions on its `/dev/hidraw*` nodes. Linux does not
grant user access to every HID device by default.

First inspect the connected device IDs:

```bash
lsusb
```

VXE R1 SE+ hardware ships with more than one receiver implementation. If the
mouse and 1K receiver appear as `3554:f58f` and `3554:f58e`, respectively,
install these narrowly scoped rules:

```udev
SUBSYSTEM=="hidraw", ATTRS{idVendor}=="3554", ATTRS{idProduct}=="f58f", TAG+="uaccess"
SUBSYSTEM=="hidraw", ATTRS{idVendor}=="3554", ATTRS{idProduct}=="f58e", TAG+="uaccess"
```

Save it as `/etc/udev/rules.d/70-openmouse-vxe.rules`, reload the rules, then
unplug and reconnect both devices:

```bash
sudo udevadm control --reload-rules
```

Grant access to every `hidraw` node for each product. Chromium opens the HID
device before OpenMouse selects its vendor configuration collection, so access
to only the `0xff02:0x0002` collection's node is insufficient.

### Linux: rapid double-clicks filtered at 0 ms

OpenMouse's debounce control changes the mouse's firmware setting. Linux's
[libinput button filter](https://wayland.freedesktop.org/libinput/doc/latest/button-debouncing.html)
can still merge rapid press/release pairs afterward. On the tested Endorfy LIV
Plus Wireless receiver, 26 raw presses became 14 browser presses with firmware
debounce set to 0 ms. The interval between presses does not measure firmware
debounce latency.

To pass these clicks through, use libinput **1.30 or newer**, built with
[Lua plugin support](https://wayland.freedesktop.org/libinput/doc/latest/lua-plugins.html),
and a compositor that loads plugins. The local KDE Wayland setup uses KWin
6.7.5 and libinput 1.32.0; its KWin build loads plugins from the default paths
unless `KWIN_LIBINPUT_NO_PLUGINS=1`. Other desktop environments may not load
them. This procedure has not yet been verified after a desktop restart.

1. Set **Debounce time** to **0 ms** in OpenMouse.
2. Create a plugin that disables only libinput's button filter for the
   `3299:00a7` USB receiver's mouse interface:

   ```bash
   sudo install -d -m 755 /etc/libinput/plugins
   sudo tee /etc/libinput/plugins/10-endorfy-liv-plus-no-debounce.lua >/dev/null <<'LUA'
   libinput:register({1})

   libinput:connect("new-evdev-device", function(device)
       local info = device:info()
       local usages = device:usages()
       if info.bustype == evdev.BUS_USB
           and info.vid == 0x3299 and info.pid == 0x00a7
           and usages[evdev.BTN_LEFT]
           and usages[evdev.REL_X] and usages[evdev.REL_Y] then
           device:disable_feature("button-debouncing")
           libinput:log_info("Endorfy LIV Plus Wireless: disabled button debouncing")
       end
   end)
   LUA
   sudo chmod 644 /etc/libinput/plugins/10-endorfy-liv-plus-no-debounce.lua
   ```

3. Save your work, **log out and back in**, then reconnect the receiver and
   repeat your click test. Reloading udev rules alone does not reload plugins.
   Compare individual button presses, rather than the browser's `dblclick`
   event, which also depends on desktop double-click settings. If nothing
   changes, check whether your compositor loads plugins and its logs for Lua
   errors; installing a plugin does not enable unsupported compositors.

Removing this filter also passes through unwanted switch bounce. Raise the
mouse's firmware debounce if ordinary clicks start registering twice. To undo
the Linux override, remove the file and log out and back in again:

```bash
sudo rm /etc/libinput/plugins/10-endorfy-liv-plus-no-debounce.lua
```

## Contributing

OpenMouse is one repository in a family — with the **Desktop** app,
**mouse-protocol**, and **OpenMouse-Bridge**.

Before you start, read the [contribution guide](https://docs.openmouse.app).
It explains how the repositories fit together, per-repo setup and
conventions, and safe reverse-engineering practices.

## Bridge updates

The Settings page compares the connected OpenMouse Bridge against its latest
stable GitHub release. It only retrieves the version, changelog, and download
link; it never downloads or installs an update in the background. Users can
also run the check manually.

When Bridge is reachable, the control panel prefers its loopback native HID
transport over browser WebHID. The same `@openmouse/protocol` drivers continue
to handle device discovery and settings; Bridge only carries HID reports. This
keeps Razer control interfaces protected by Chrome 153+ usable without relying
on the browser's former protected-collection bypass.

Bridge connection attempts, disconnects, request failures, scan durations, and
device inventory changes are written to the browser console with the
`[OpenMouse Bridge]` prefix. The same lifecycle events are included in the
downloadable diagnostics from a connected device's **Advanced** panel; HID
report payloads are not written to the console.

The control panel is organized by responsibility: `control.ts` coordinates the
application, while the template, events, DOM helpers, persisted preferences,
battery history, device selection, and rendering live in focused modules under
`src/`.

Packet codecs and WebHID drivers live in the standalone
[`@openmouse/protocol`](https://github.com/OpenMouse-Project/mouse-protocol)
library. Its codec entry points remain transport-independent, while its
`drivers` entry points own discovery filters, device clients, retries, and
application-facing status conversion. OpenMouse consumes the same public
exports that external consumers use.

## Adding a vendor

Codec and driver contributions belong in the `mouse-protocol` repository.

1. Add transport-independent packet definitions and codecs under the vendor's
   `mouse-protocol/src/<vendor>/` folder.
2. Add the WebHID implementation under `mouse-protocol/src/drivers/<vendor>/`.
3. Register the driver and browser filters in the shared driver layer.
4. Add or extend codec and driver tests, state which product IDs were verified
   on hardware, and run the checks in both repositories.

OpenMouse should only need changes when a driver introduces a genuinely new UI
capability. The control UI otherwise discovers supported clients through the
library registry automatically.

Hardware-specific validation checklists live in the protocol repository's
`docs/` directory.

### Testing a local protocol driver

Keep `mouse-protocol` beside this repository. Build the library, then install
the local package into OpenMouse without changing its manifest or lockfile:

```bash
cd ../mouse-protocol
npm run check
cd ../openmouse
npm install --no-save --package-lock=false ../mouse-protocol
npm run check
npm run dev
```

Restart the development server after switching protocol builds. Running
`npm ci` restores the manifest's dependency; repeat the local installation to
test a different driver build again.

This fork includes Endorfy LIV Plus Wireless support through a packaged
`@openmouse/protocol` snapshot, with 0–20 ms debounce, DPI, polling rate,
battery and firmware reads over the `3299:00a7` receiver. Its mouse artwork
is served locally. See [vendor/README.md](vendor/README.md) for the source
patch, hardware evidence and package-refresh steps. A normal `npm ci` uses
this snapshot, so a fresh checkout has the same driver.

## License

[GNU AGPL-3.0](LICENSE). Contributions are accepted under the same license.
