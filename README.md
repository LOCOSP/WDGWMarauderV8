# WDGWMarauderV8

Wardriving firmware for the **Marauder V8 (ESP32-C5)** — with an optional cluster of
five **Seeed Studio XIAO ESP32-C5** nodes that scan in parallel and report back over
ESP-NOW.

It logs Wi-Fi 2.4 + 5 GHz networks and BLE devices with a GPS position, writes them to
microSD in **WigleWifi-1.6** format, and uploads to **[wdgwars.pl](https://wdgwars.pl)**.

**🇵🇱 Wersja polska: [README.pl.md](README.pl.md)**

This is **not** stock ESP32Marauder. It is separate firmware written for this board.

---

## What it does

**Capture**
- Wi-Fi in promiscuous mode on 2.4 and 5 GHz, with adaptive dwell time per channel
- BLE (active scan), detection of **Flock cameras** over both Wi-Fi and BLE
- **Find My / AirTag** tracker detection and non-owner ring (anti-stalking)
- Capture **without GPS reception** (metro, garage, shopping centre) — hits are parked
  and given a position by interpolation once the fix returns

**Storage and upload**
- microSD, WigleWifi-1.6 with the frequency column
- Double-buffered writer; the card transaction happens **outside the lock**, so capture
  does not drop frames while a write is in flight
- Upload to wdgwars.pl over TLS with a pinned certificate; uploaded files are marked
- Browse logs on the device, upload or delete individual files

**Cluster (optional)**
- The Marauder stops scanning and **coordinates** a fleet of up to 18 nodes
- Channels are split by **measured traffic**, not evenly — a busy channel gets a node to
  itself and stops hopping altogether, hearing every beacon on it
- One node takes the **BLE + Flock** role (the C5 has a single radio — Wi-Fi or BLE,
  never both well)
- The core stamps each hit with the position **from the moment the node heard it**, not
  from the moment it arrived — without that, 20 s of buffering at 50 km/h puts a network
  300 m down the road
- **Over-the-air node updates** from the Marauder's SD card, no cable
- **Pairing with on-screen approval** — every rig generates its own random key

---

## Hardware

| Role | Board | Notes |
|---|---|---|
| Core / standalone wardriver | **Marauder V8 with ESP32-C5** | TFT + touch, microSD, GPS, MAX17048 |
| Cluster node (optional) | **Seeed Studio XIAO ESP32-C5** | 8 MB flash, PSRAM; no display, no card |

The cluster is optional — the Marauder works perfectly well on its own with no nodes
at all.

Power: five nodes in continuous receive draw **~600–750 mA**, plus the Marauder. A USB
hub without its own supply causes resets, and a node browning out looks **exactly** like
a node that lost the core.

---

## Install — browser flasher (easiest)

Needs nothing but a Chrome-based browser.

1. Open **[esptool.spacehuhn.com](https://esptool.spacehuhn.com/)**
2. Connect the board over USB, click **Connect**, pick the port
3. Add **one** file at offset **`0x0`**:

| Device | File |
|---|---|
| Marauder V8 | `firmware/marauder-v8-c5/WDGWMarauderV8-marauder-merged.bin` |
| XIAO ESP32-C5 (node) | `firmware/node-xiao-c5/WDGWMarauderV8-node-merged.bin` |

4. Click **Program** and let it finish
5. Power-cycle the board

The merged images contain bootloader, partition table and application, which is why the
offset is `0x0` and there is nothing else to enter.

> **Careful when re-flashing a node over USB.** A node that has ever taken an
> over-the-air update boots from the **second** partition. A merged image writes the
> first one, so without erasing the flash the chip keeps running the **old** firmware —
> even though flashing reports success. Tick **Erase device** before programming.

---

## Install — `esptool` from the command line

Requires `esptool` **5.x**. Version 4.8.1 from Homebrew **hangs on the ESP32-C5** — use
the one bundled with the Arduino ESP32 core, or `pip install --upgrade esptool`.

**Marauder V8:**

```bash
esptool --chip esp32c5 --port /dev/ttyUSB0 --baud 921600 write-flash -z \
  0x2000  firmware/marauder-v8-c5/bootloader.bin \
  0x8000  firmware/marauder-v8-c5/partitions.bin \
  0x10000 firmware/marauder-v8-c5/app.bin
```

**XIAO ESP32-C5 node:**

```bash
esptool --chip esp32c5 --port /dev/ttyACM0 --baud 921600 erase-flash
esptool --chip esp32c5 --port /dev/ttyACM0 --baud 921600 write-flash -z \
  0x2000  firmware/node-xiao-c5/bootloader.bin \
  0x8000  firmware/node-xiao-c5/partitions.bin \
  0x10000 firmware/node-xiao-c5/app.bin
```

The `erase-flash` on the node matters — see the partition note above.

Ports: on Linux usually `/dev/ttyUSB0` (Marauder, USB-UART bridge) and `/dev/ttyACM0`
(XIAO, native USB). On macOS `/dev/cu.usbserial-*` and `/dev/cu.usbmodem*`.
Every XIAO enumerates under the **same** port name, so with several of them the port
tells you nothing — read the MAC with `esptool ... read-mac` to know which one you hold.

**Verify a node after flashing** — on reset the console (115200) prints:

```
[node] WDGNODEFW:0004 | firmware v4
```

If that line is missing, the node booted the old image from the other partition.

---

## Configuration

Copy **`wdgwars.cfg.sample`** to the **root of the microSD card** and rename it to
**`wdgwars.cfg`**. The file stays on your card — the firmware never uploads it and never
prints it (passwords are only ever reported as a length).

Minimum to get going:

```ini
ssid=YourWiFiName
pass=YourWiFiPassword
key=your_64_hex_character_api_key
```

Generate the API key in your profile on wdgwars.pl — it is exactly 64 hex characters and
ties uploaded networks to your account, so use your own.

Without the file the device still **captures and logs normally** — only uploading is
unavailable.

---

## Running the cluster

1. Flash the **node** firmware onto each XIAO
2. On the Marauder: **MENU → CLUSTER**
3. Nodes announce themselves. A bar appears:

```
5 node(s) want to join:
1CD4  0178  1BBC  3924  8020
[ ACCEPT ALL ]        * = re-adopt
```

4. Check the address suffixes against your own units, then press **ACCEPT ALL**

The Marauder then generates **its own key** from the hardware RNG and hands it to the
nodes you accepted. From that point the fleet is yours.

**BACK** leaves the screen with the fleet still working. **STOP** ends the session and
closes the log file so it shows up in SYNC.

A star next to an address means "I hold a key but nothing answers me" — that is how a
node asks to be re-adopted after the core was re-keyed. No cable needed.

### Node firmware: first install, then updates

**The first time, every node needs a cable.** There is no way round it: a factory XIAO
has no firmware that could receive an update. Flash each one as described in *Install*
above — browser flasher or `esptool`, either is fine.

**After that, never again.** Updates go over the air from the Marauder's card.

#### Putting the image on the card

Download **`node_fw.bin`** from
[Releases](https://github.com/LOCOSP/WDGWMarauderV8/releases) and copy it to the
**root of the microSD card**:

```
/node_fw.bin          ← here
/wdgwars.cfg
/wdgw/                ← logs live here, the image does NOT
```

The name must be exactly `node_fw.bin` and it must sit in the root, not in `wdgw/`.
That is the only place the firmware looks.

The Marauder checks the card **at every boot**. If it finds an image, the start-up screen
shows a line like `node fw v4 on card`, and the CLUSTER screen adds a note when any node
is behind. You do not have to go looking.

> Prefer not to pull the card? There is a serial route: `nodefw <bytes>` on the console
> (115200), then send the raw file. It takes about two and a half minutes per megabyte and
> **blocks the panel** for the duration — the screen says so. Pulling the card is faster.

#### Running the update

1. **MENU → CLUSTER** — wait for the nodes to check in, then leave with **BACK**
   (not the red STOP, which would release the fleet)
2. **MENU → NODE FW → CHECK** — shows the image version and what each node is running
3. **UPDATE ALL**

What you will see, in order:

| Stage | Meaning |
|---|---|
| `asking nodes who needs it...` | a node listens on the control channel only briefly, so the core waits for each one to speak |
| `sending 42%` | the image is going out by broadcast — five nodes cost the same as one |
| `filling gaps` | broadcasts are unacknowledged, so nodes now request the chunks they missed |
| `updated` per node | written and committed |
| `all 5 now run v4` | **verified after they rebooted** — the version they report back |

**UPDATE ALL is greyed out** when there is nothing to do: no image, no nodes, a transfer
already running, or everything already up to date. Press it anyway and it will say which.

You can leave the screen mid-transfer; the update carries on.

#### Why it will not brick a node

Each node assembles the image in PSRAM and verifies the **SHA-256 of the whole thing**
before flash is touched at all — a partial or corrupted transfer simply fails and the node
keeps running what it had. After committing, the new firmware boots **on probation**:
unless it reaches the core within two minutes it puts the boot partition back and reboots
into the previous version.

That covers the case a hash cannot: an image that is intact but broken.

---

## The screens

The main screen wardrives. Everything else lives under **MENU**, which has two rows of
options and **INFO | SET | BACK** along the bottom.

### Three screens need another one run first

This trips everyone up once, so it is worth stating before anything else. Three screens
show a **frozen list gathered by a different screen** and deliberately do not gather it
themselves. Open one cold and it looks broken — it is not, it is waiting.

```
   BT SCAN      →  BACK  →   FOX HUNT
   AIRTAG SCAN  →  BACK  →   RING TAG
   CLUSTER      →  BACK  →   NODE FW
```

| You want to | Do this first | Then |
|---|---|---|
| Track down a Bluetooth device | **BT SCAN** — let the list fill | **BACK**, then **FOX HUNT**, pick your target |
| Ring a tracker | **AIRTAG SCAN** — let it find tags | **BACK**, then **RING TAG**, pick the tag |
| Update the nodes | **CLUSTER** — wait for the fleet to check in | **BACK**, then **NODE FW → CHECK → UPDATE ALL** |

**Why they work that way.** Scanning and picking cannot happen at once: the list would
reorder itself under your finger as signals came and went, and on this chip a scan running
behind a picker competes for the one radio. Freezing the list makes the choice stable.

**Leave with BACK, not STOP.** On CLUSTER the red **STOP** ends the session — it releases
the fleet and closes the log file. **BACK** just leaves the screen and everything carries
on, which is what you want before opening NODE FW: it needs the nodes online to see their
versions.

If a picker shows an empty list, it will tell you which screen to run first rather than
sitting on "scanning…" forever.

### SCAN — wardriving

Captures Wi-Fi and BLE with a GPS position and writes WigleWifi-1.6 rows to the card.
Subject to `sub wifi|ble|both` on the console; **both** is the default and the stable one
on this chip.

Underground, in a garage or a shopping centre — anywhere without sky — hits are **parked
rather than discarded**, and given a position by interpolation once the fix returns. A row
at 0,0 would be thrown away by the portal *and* would burn that MAC in the dedup table, so
the network would never be logged again after you drove back out.

### SYNC — logs and upload

Lists the logs on the card, newest first; uploaded ones are marked. Tap a file for
**UPLOAD** and **DELETE** — housekeeping without pulling the card. The file currently
being written is shown as recording and cannot be uploaded.

### BT SCAN — Bluetooth devices

Active BLE scan with a live list: address, name if advertised, RSSI. This is also the
list **FOX HUNT** picks from.

### FOX HUNT — find a specific device

Direction finding by signal strength. Pick a device and walk: the reading rises as you
close in. Useful for locating a beacon, a lost tag, or working out which device in a room
is which.

> **Scan first, then pick.** FOX HUNT deliberately does **not** scan — it shows the list
> **BT SCAN** gathered. Order: **BT SCAN → BACK → FOX HUNT**. Opening it without scanning
> first gives an empty list, which is a prompt, not a fault.

### AIRTAG SCAN — trackers

Finds Find My / AirTag-class trackers around you. Filtering matters here: manufacturer
`0x004C` alone matches **every** Find My participant, phones and Macs included, which
buries the list. The firmware filters on the status-byte category and on service UUIDs —
Apple `0xFD44`, Samsung SmartTag `0xFD5A`, DULT `0xFCB2`, Google, Tile, Chipolo — and
drops anything below −85 dBm, since anti-stalking is about what travels **with** you.

Tags separated from their owner are flagged **SEP**. That flag decides whether ringing
will work.

### RING TAG — make a tracker beep

Pick a tag from what AIRTAG SCAN found and make it sound, to locate something planted in
a bag or a car.

> Same rule: **AIRTAG SCAN → BACK → RING TAG**. The list is frozen from the scan.

**What will and will not ring — this is Apple's design, not a limitation of this
firmware:**

- **Owner ringing** (what the Find My app does) is gated by the pairing secret. Not
  possible from an ESP32, by anyone.
- **Non-owner ringing** (this, the anti-stalking path) works **only while the tag is
  separated from every one of its owner's Apple devices**.

With the owner nearby the write **succeeds at the protocol level** and the tag still stays
silent, answering `0xFFFF`. That is the most misleading symptom in the whole feature:
everything looks like success and nothing beeps.

To ring your own paired tag: switch off Bluetooth on your iPhone and Mac, wait 15–30
minutes until it shows **SEP**, then ring. An AirPods case rings readily and is a red
herring when testing.

### CLUSTER and NODE FW

The fleet and its updates — see the sections above.

### SET and INFO

Settings and device information: firmware build, GPS state, battery, card, memory.

---

## Security — what it protects, and what it does not

Stated plainly, because published binaries can be searched.

**The fleet key is generated on your device.** It is not in this repository and not in
any image here. It signs every ESP-NOW frame — every assignment, every record, every
firmware chunk.

**Correction (2026-08-06):** an earlier version of this file claimed control traffic was
also *encrypted* with AES-CCM. It is not. The key is installed as the ESP-NOW PMK, but
per-peer encryption is deliberately left off, because ESP-NOW encrypts at most **six**
peers while carrying twenty in total — turning it on would cap a fleet at six nodes. The
fleet size won. Traffic is therefore **signed but readable**, and the sentence has been
corrected rather than quietly deleted.

**What that buys you:** nobody in radio range can push firmware to your nodes or inject
fake networks into your log — which was the serious consequence of the previous,
compiled-in secret.

**What it does not give you:**
- Broadcasts (update chunks, join requests) are **signed but readable** — ESP-NOW cannot
  encrypt a broadcast
- The secret used for **joining only** is public and visible in the image
  (`wdgwars-fleet-2026`). It carries nothing but "may I join?" and grants nothing beyond
  the right to appear on your screen asking to be accepted
- The fleet key crosses the air **in the clear in a single frame**, at the moment you
  press ACCEPT. That window lasts a fraction of a second and your deliberate action opens it
- The tag is a keyed checksum, **not a cryptographic signature**. It stops impersonation
  and accidental cross-talk between fleets, not a determined adversary

**Your data:** `wdgwars.cfg`, with your API key and Wi-Fi passwords, stays on the card.
It is never written to a log, never uploaded, never printed to the console.

---

## Data format

WigleWifi-1.6 — the one wdgwars.pl expects (**not** 1.4; the difference is the frequency
column):

```
MAC,SSID,AuthMode,FirstSeen,Channel,Frequency,RSSI,Lat,Lon,AltitudeMeters,AccuracyMeters,Type
54:DB:A2:1A:D7:DC,HALNy-2.4G,[WPA2-PSK-CCMP][ESS],2026-07-30 02:41:43,1,2412,-88,50.1234,17.5678,123,8.0,WIFI
82:19:4F:FE:81:43,,[BLE],2026-07-30 02:41:43,0,0,-95,50.1234,17.5678,123,8.0,BLE
```

---

## When something is wrong

| Symptom | Check first |
|---|---|
| A node "went quiet" | power. Brown-out looks identical to losing the core |
| Node reports an old version after a cable flash | it booted the other partition — erase flash and reflash |
| Counters climbing, card empty | whether a log session is open (the screen names the file) |
| Upload rejected | `202` and `409` from the server are **success**, not errors |
| Writes stop after a while | the card — run `sdtest` on the console |

The serial console (115200) has the full set: `status`, `logstats`, `dumplog`, `cluster`,
`pair`, `nodeota`, `scanstat`, `sdtest`, `help`.

The rule that saved the most time on this project: **a confirmation of a write is not a
confirmation that something works.** Check the effect, not the claim — the version banner
after an update, `logstats` after a write, a read-back from the chip after flashing.

---

## Credits

Patterns and inspiration: [justcallmekoko/ESP32Marauder](https://github.com/justcallmekoko/ESP32Marauder),
[dark3d/ESP32DualBandWardriver](https://github.com/dark3d), C5Lab/projectZero,
[pr3y/Bruce](https://github.com/pr3y/Bruce).
Web flasher: [Spacehuhn Technologies](https://esptool.spacehuhn.com/).

---

## Licence

**Released as binaries only.** The source code is not published.

Copyright © 2026 LOCOSP / [wdgwars.pl](https://wdgwars.pl). All rights reserved.

You may download and run these images on your own hardware. Redistribution, resale,
decompilation, and presenting this work as your own are not permitted.

This firmware is for **lawful** wardriving — passive reception of publicly broadcast
frames — and for anti-stalking tracker detection. Complying with the law where you use
it is your responsibility.
