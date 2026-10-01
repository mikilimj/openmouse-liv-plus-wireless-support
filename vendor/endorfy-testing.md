# Endorfy LIV Plus Wireless (EY6A016)

Hardware exercised on 2026-10-01: `3299:00a7`, product string
`Compx LIV Plus Wireless 2.4 GHz`, mouse firmware `2.03`, on Linux hidraw.
Only this receiver is registered. Wired USB, Bluetooth, and EY6A017 have not
been exercised and have no speculative USB IDs or discovery entries.

## Evidence

The local receiver's interface 1 declares usage `ff02:0002`, report ID 8,
16-byte input/output reports. Feature report 6 on `ff04:0002` is separate
and is not used by the driver. Interfaces 0 and 2 are keyboard/mouse input
collections and must not be claimed as the configuration channel.

The report-8 framing, checksum, battery and flash read commands match the
existing VGN/Pulsar codecs. All frames include report ID 8 in the additive
checksum: the complete 17 bytes sum to `0x55` modulo 256. The driver imports
these shared codecs rather than reproducing their packet construction.

Selected real receiver replies below include the report ID:

```text
battery:  08 04 00 00 00 02 32 00 0f 11 00 00 00 00 00 00 f5
firmware: 08 12 00 00 00 02 02 03 00 00 00 00 00 00 00 00 34
flash 0:  08 08 00 00 00 0a 01 54 01 54 00 55 00 55 00 55 92
flash 10: 08 08 00 00 0a 0a 01 54 08 08 00 45 07 07 00 47 32
```

These decode to 50% battery, 3857 mV, firmware 2.03, polling code 1
(1000 Hz), one enabled DPI stage, active index 0, and stage 0 at 450 DPI.
The user did not independently know their existing DPI setting; its numerical
interpretation is based on the shared encoding and device read-back.

Polling, enabled-stage count, and active index are scalar/checksum pairs
at offsets 0, 2, and 4. DPI records start at 12 with four bytes per stage;
X/Y low bytes and high-bit flags must match, and each record sums to 0x55.
The driver rejects corrupt or unsupported settings instead of substituting
plausible defaults. The conservative six-stage ceiling and 26000 DPI ceiling
come from the vendor's published specification and factory presets:
https://endorfy.com/en/products/liv-plus-wireless

## Debounce evidence

The official LIV Plus Wireless software 1.0.0.4 (2025-03-14) lists every
integer from 0 through 20 ms in `app/Language/0-English.xml`, `DebounceTime`.
The archive is linked from https://endorfy.com/en/pages/software-2:
https://endorfy.com/cdn/shop/files/ENDORFY-LIV-Plus-Wireless-Software-2025-03-14.zip?v=10453208085035470114

The receiver initially returned `[00, 55]` at `0xa9`, the same debounce
address used by the shared VGN/Pulsar flash layout. Writes to this pair were
then confirmed on the real receiver. The driver validates the scalar checksum,
refuses values outside 0–20 ms before I/O, and reads back each write before
reporting success. It does not rewrite neighboring motion-sync settings.

## Hardware results

The production Endorfy client was exercised through a thin Python hidraw
adapter implementing the same report calls as WebHID. Actual write commands
were limited to polling bytes at offset 0, the active DPI record at 12,
and the debounce scalar/checksum pair at offset `0xa9`.

- DPI writes: 500, 800, 1600; every value read back correctly.
- Polling writes: 125, 250, 500, 1000 Hz; every value read back correctly.
- Debounce writes: 1, 4, 20, and 0 ms; every value read back correctly.
- Original 0 ms debounce was restored and read back after close/open; DPI and
  polling stayed unchanged throughout the debounce test.
- Original 450 DPI and 1000 Hz settings were restored and read back.
- The adapter additionally restored the original raw bytes on shutdown.

The shared stage editing and active-stage selection have automated transport
coverage; selecting a different enabled stage has not been tested on this
unit, which currently has only one enabled stage. Stage-count changes,
RGB, button remapping, profiles, processing, and firmware updates are not
exposed. No persistence across a power cycle or browser/Bridge transport
claim is made by the hidraw tests.

## Remaining manual checks

1. Connect through Chromium's picker or OpenMouse Bridge and verify the single
   receiver card, name, battery, firmware, DPI, polling rate and debounce.
2. Change DPI, polling rate and debounce through the UI, read back, and restore settings.
3. Exercise sleep/wake and reconnect behavior.
4. Validate other enabled DPI stages when present, including the upper DPI range.
5. Test persistence after powering the mouse off and on.

For Linux WebHID, grant access to all hidraw nodes of this product, then
reload udev rules and unplug/reconnect the receiver:

```udev
SUBSYSTEM=="hidraw", ATTRS{idVendor}=="3299", ATTRS{idProduct}=="00a7", TAG+="uaccess"
```
