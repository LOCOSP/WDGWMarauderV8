# Release conventions

The wdgwars.pl portal has a flasher that serves binaries **straight from this repo's
releases**. It matches files by name and verifies them against `SHA256SUMS.txt`. That
makes the things below a contract, not a preference: break one and the portal quietly
stops offering the new version — no error, just users left on an old build.

## Asset names are fixed

Every release must attach exactly these, spelled exactly this way:

| File | What it is |
|---|---|
| `WDGWMarauderV8-marauder-merged.bin` | Marauder V8 (ESP32-C5), single image for offset `0x0` |
| `WDGWMarauderV8-node-merged.bin` | Seeed XIAO ESP32-C5 node, single image for offset `0x0` |
| `node_fw.bin` | the same node image, named for over-the-air updates from the SD card |
| `SHA256SUMS.txt` | checksums, flat names (see below) |

**A rename is a silent outage.** If a new board is ever added, tell the portal side
*before* the release, not after.

## `SHA256SUMS.txt` uses bare filenames

```
782ae666...  WDGWMarauderV8-marauder-merged.bin
27c4dae9...  WDGWMarauderV8-node-merged.bin
f127cddb...  node_fw.bin
```

No directory prefixes, so anyone can verify with one command in the folder they
downloaded into:

```bash
sha256sum -c SHA256SUMS.txt      # shasum -a 256 -c on macOS
```

It covers **the release assets only**, not the whole repo tree. That is deliberate: the
per-chip files (`app.bin`, `bootloader.bin`, `partitions.bin`) exist under *both*
`marauder-v8-c5/` and `node-xiao-c5/`, so a flat list including them would carry two
different hashes for the same name and `-c` could not resolve it.

**What a matching checksum does and does not prove.** It proves the file survived the
trip intact. It does **not** prove the release is good — the checksum and the binary come
from the same place, so anyone who could tamper with one could tamper with both. That is
why a human approves a release on the portal side before it reaches anybody. These
binaries end up on physical hardware; nothing about that should be automatic.

## The firmware states its own version

Both images carry a scannable tag and print it on boot:

```
[wdg]  WDGWCOREFW:0005 | firmware v5      <- Marauder
[node] WDGNODEFW:0005  | firmware v5      <- XIAO node
```

**This must match the release tag's major number.** `v5.x.x` → both print `v5`.

The tag is what lets anyone confirm what a device is *actually running*, instead of
trusting the name of the file that was flashed. That distinction has settled several
arguments already — a node can report a successful write and still boot the previous
image (see the partition note in the README), and a checksum on disk says nothing about
what the chip executes.

Serial console: `version` prints the core's build, the node image on the card, and every
node's reported version in one go.

Bump both `WDG_CORE_FW_VER` / `WDG_CORE_TAG` and `WDG_NODE_FW_VER` / `WDG_FW_TAG` when
cutting a release. The tag string is `__attribute__((used))` on purpose — `volatile`
alone did not stop the compiler discarding it, and it once vanished from an image
silently.

## Webhook

Releases notify the portal over a webhook configured in this repository's settings:

- Event: **`release` only** — not "send me everything"
- Content type: `application/json`
- Secret: signs `X-Hub-Signature-256`

**The secret never goes in this repo**, in a release description, or in a commit
message. It lives only in the repository's webhook settings.
