# Notes for whoever builds the web flasher

Everything here comes from firmware behaviour, not preference. Each point is something
that has already gone wrong for a real user.

---

## 1. The merged image erases the fleet key. This is the big one.

`esptool merge-bin` produces one contiguous image starting at `0x0` and pads the gaps
between bootloader, partition table and application with `0xFF`. **NVS lives at `0x9000`,
inside one of those gaps.** Writing the merged file therefore wipes NVS.

NVS holds the **fleet key** — the random secret a Marauder generates when it adopts its
cluster nodes. Erase it and the rig mints a new key while the nodes still hold the old
one. The entire fleet goes silent, and nothing on screen explains why. This happened to
us, with five nodes, and cost an evening.

**What the flasher must do:**

| Situation | Offer | Why |
|---|---|---|
| First install / blank board | `WDGWMarauderV8-marauder-merged.bin` at `0x0` | there is nothing to preserve |
| Updating an existing device | **`app.bin` at `0x10000`** | never touches NVS |

If you offer only one button, offer the **update** path and make the full-erase install
the deliberate, second choice. Someone updating a working rig is the common case;
someone setting up a blank board reads instructions.

If you offer both, do not label them "safe" and "unsafe" — label them by intent:
**"Install (first time)"** and **"Update (keep pairing)"**.

Firmware from **v5.3.0** mirrors the key to the SD card and restores it when NVS comes
up empty, so a merged flash is survivable *if the card is present*. Do not rely on that:
a user flashing with the card removed still loses the fleet.

## 2. Node images are not interchangeable with core images

Two different boards, two different files:

| File | Board |
|---|---|
| `WDGWMarauderV8-marauder-merged.bin` | Marauder V8 (the one with a screen) |
| `WDGWMarauderV8-node-merged.bin` | Seeed XIAO ESP32-C5 (cluster node, no screen) |
| `node_fw.bin` | same node image, for over-the-air updates from the SD card — **not** for flashing over USB |

`node_fw.bin` is a release asset only because the Marauder reads it off the card. If a
flasher offers it as something to write over USB it will work, but it is the same bytes
as the node merged image minus bootloader and partition table — so it would be flashed
at the wrong offset. **Do not list it as a flashable target.**

## 3. Nodes need a full erase, cores do not

A node that has ever taken an over-the-air update boots from the **second** OTA slot.
A merged image writes the first one, so without erasing the flash the chip keeps running
the **old** firmware while flashing reports complete success.

For nodes: **always erase before writing.** For the Marauder: erasing is what destroys
the pairing (point 1), so do not erase unless the user asked for a clean install.

That asymmetry is unintuitive and worth a line of UI text.

## 4. Do not trust the filename to say what got installed

Both images carry a scannable build tag and print it on boot at 115200 baud:

```
[wdg]  WDGWCOREFW:0005 | firmware v5.5.0     <- Marauder
[node] WDGNODEFW:0007  | firmware v7         <- XIAO node
```

If the flasher can show the serial output after a reset, show those lines. They are the
only proof of what the chip actually executes — a checksum on disk says nothing about
what booted, and we have had a node report a successful write and then run the previous
image.

## 5. Verify downloads against `SHA256SUMS.txt`

Flat filenames, so one command works in the download folder:

```bash
sha256sum -c SHA256SUMS.txt
```

It covers the release assets only. **What it proves:** the file arrived intact.
**What it does not prove:** that the release is trustworthy — the checksum ships from the
same place as the binary. Keep a human in the approval path; these images end up on
physical hardware.

## 6. Known: the browser flasher does not work for everyone

At least one user gets `Couldn't sync to ESP. Try resetting.` from
[esptool.spacehuhn.com](https://esptool.spacehuhn.com/) while command-line `esptool`
flashes the identical file successfully with a verified hash. So it is not the image.

Likely the USB-UART bridge or how the browser drives DTR/RTS. If your flasher hits the
same wall, worth surfacing in the UI:

- hold **BOOT** while connecting
- try another cable (many are charge-only)
- some bridges need a lower baud rate
- fall back to the `esptool` command, which is in the README

Do not silently retry forever — say which of these to try.

## 7. Release asset names are a contract

`WDGWMarauderV8-marauder-merged.bin`, `WDGWMarauderV8-node-merged.bin`, `node_fw.bin`,
`SHA256SUMS.txt`. The portal matches on these names; a rename is a silent outage, not an
error. If a new board is added you will be told before the release, not after.

Details in [RELEASING.md](RELEASING.md).

## 8. What a user sees if pairing does break anyway

Worth knowing, so support answers do not contradict the device:

- The cluster screen lists nodes asking to join, marked `*` when they are re-joining a rig
  that lost its key
- **ACCEPT ALL** hands the key over; an accepted node stays listed with `...` until it
  genuinely comes back, so "it vanished" never has to mean "it worked"
- Recovery needs no cable, even for a fleet that was fully cut off

So the honest support answer to "my nodes disappeared after flashing" is: open CLUSTER,
press ACCEPT ALL, and it comes back.
