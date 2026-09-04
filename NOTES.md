# Notes

## 2026-09-04 — Central/peripheral swap (right is now central)

Swapped which half is BLE central: **right** is now central (plugs into USB/host,
pairs over Bluetooth), **left** is now peripheral. Previously left was central.

Changed in:
- `boards/arm/eyelash_sofle/Kconfig.defconfig` — `ZMK_SPLIT_ROLE_CENTRAL` and
  `ZMK_KEYBOARD_NAME` moved from `BOARD_EYELASH_SOFLE_LEFT` to
  `BOARD_EYELASH_SOFLE_RIGHT`.
- `build.yaml` — the ZMK Studio build (`studio-rpc-usb-uart` snippet) moved from
  the left board entry to the right board entry, since Studio talks to whichever
  half is central over USB.

Reflash checklist after this change: flash `settings_reset` (nice_nano_v2 board)
to **both** halves first to clear the old central/peripheral bond, then flash the
new left/right firmware, then re-pair with the host.

Also renamed the BLE device name from "Eyelash Sofle" to `split_keeb`
(`ZMK_KEYBOARD_NAME` in the same Kconfig block). This is just the advertised
name — board/file names are still `eyelash_sofle`.

### Why: left display issue

Reason for the swap — the left display (nice!view) was showing a burst of
noise/dots on boot that cleared after a moment, while left was central. Wanted
to see if moving central duties off the left half changes that.

Investigated whether this is a known ZMK software bug:
- Found a real, matching-sounding bug: zmkfirmware/zmk#3219 — after ZMK's
  Zephyr 4.1 update, nice!view widgets render as garbled/pixelated blocks,
  and specifically *every widget on the central* is affected (not just one,
  as on the peripheral). Root cause: `LV_Z_MEM_POOL_SIZE` (LVGL memory pool)
  too small after the Zephyr 4.1 / LVGL 9.3 bump. Fixed upstream by raising
  it from 4096 to 4608.
- **Does not apply here**: this repo pins `zmk` to `revision: v0.3.0` in
  `config/west.yml`, and per ZMK's own blog post the Zephyr 4.1 jump only
  ships starting in v0.4. So this repo predates the code path that causes
  #3219.
- The display driver devicetree (`nice_view_spi` / SPI GC9A01 panel init in
  `boards/arm/eyelash_sofle/eyelash_sofle.dtsi`) is identical between the
  left and right board files. Central vs. peripheral only changes
  `ZMK_SPLIT_ROLE_CENTRAL` in Kconfig, which doesn't touch the display driver
  path at all — so there's no code-level reason the central role should
  change display init behavior on this ZMK version.

Conclusion: no matching known software bug for this ZMK version. The
noise-then-clear pattern being tied to one specific physical half, with no
software explanation, points toward **hardware** on the left board or its
display module (loose FPC connector, marginal solder joint, or an
out-of-spec panel) rather than firmware/role.

Not yet done — worth trying before assuming it's fixed:
- Swap the physical nice!view display module between the left and right
  boards (leave MCUs in place) to see if the glitch follows the display
  (bad panel) or stays with the left board (connector/solder issue).
- After reflashing with right as central, watch whether the left half (now
  peripheral) ever shows the same glitch or a different symptom
  (garbled/frozen status) — if so, that confirms it's hardware, not role.
