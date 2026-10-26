# KD-Box II: root SSH without opening the case

The Goldshell KD-Box II is a small blake2s miner built for Kadena. When KDA stopped paying,
the stock firmware left the hardware locked to a coin nobody mines, and gave owners no way in:
SSH is installed but killed a couple of seconds after boot, and the only interface is a web UI
fronted by a factory password.

This repo is the way around that. It gets you a **root shell over SSH on hardware you already
own** — your key, not a password, surviving reboots — without opening the case or
soldering anything to the board.

It works because the box will install any firmware update it can verify, and the key it
verifies with ships inside the box. So we can build our own rootfs, sign it, and install it
through the very same web page Goldshell uses for official updates. One reboot, about three
and a half minutes, a browser and a Linux machine.

**In three lines:** build a signed image on your PC → flash it from the miner's web page →
log in with your SSH key. Steps below; screenshots in
[docs/walkthrough.md](docs/walkthrough.md).

## Bringing an AI assistant?

Point it at **[CLAUDE.md](CLAUDE.md)**. It routes to `docs/AGENTS.md` — the whole brief in
`ab/1` form (verified facts, procedure, verification gates, failure signatures, rules),
written so Claude, DeepSeek, OpenCode or anything else can act on it without rediscovering
the hardware. If you'd rather read the same things yourself with screenshots, that's
[docs/walkthrough.md](docs/walkthrough.md).

## What you get

- A root shell over SSH, key-only, that survives power cycles.
- Control of the rootfs: your own services, init tweaks, and diagnostics.
- The box still mines afterwards — the update rewrites the config partition, so re-add your
  pool and set a real admin password.
- **The WiFi toggle works.** On stock 2.2.7 the toggle is cosmetic — the box runs an *open*
  access point (`hostapd`, no encryption) and ignores the flag entirely. This image replaces
  that supervisor with one that reads the flag and converges the radio to it, off included.
- No case opening, no wiring, no `dd`.

## What it does not do

- **It does not change what the hardware can hash.** The `ICA585` chips in this model
  implement blake2s and nothing else; blake2b (Sia/Handshake) needs different silicon.
- **It is not a remote exploit.** You need to be on the same LAN as the miner and know its
  web password (factory default `123456789`). That is the point: it is a way in for the owner,
  not for a stranger.
- **It is for the KD-Box II only** (`mcbversion` `MCB_V6_3_5`, firmware 2.2.7). Another model
  or another MCB version needs a build from its own stock package, not a renamed copy.

## Step by step

The short version, in order. If you have not done this before, read
**[docs/walkthrough.md](docs/walkthrough.md)** — same steps, with screenshots of every screen.

1. **Look at your box** — note *Firmware version* / *Hardware version*, download the matching
   stock package. It is your build input **and** your rollback.
2. **Change the rootfs** — unpack, add your **RSA** key, stop `minerd` killing sshd at boot,
   repack. Five commands: [walkthrough §2](docs/walkthrough.md#step-2--change-the-rootfs).
3. **Build and sign** — `scripts/build-cpb.py`; you want `ALL CHECKS PASSED`.
4. **Unlock the web UI** — *System* → *UNLOCK*, admin password `123456789` on a fresh box.
5. **Flash** — *Update* → pick the `.cpb` → *UPDATE*. No confirmation dialog; ~3.5 minutes.
6. **Log in** — `ssh` with the flags below, then reboot once to prove it sticks.
7. **Put it back to work** — the flash wiped the config partition: re-add your pool, change
   the admin password.

Prefer the command line? `scripts/flash-ota.sh <miner-ip> <package.cpb>` does steps 4–5, and
`KD_DRY_RUN=1` stops after login so you can inspect the request first.

Anything odd on the way? **[docs/troubleshooting.md](docs/troubleshooting.md)** — every
failure we've seen, with causes and fixes, including recovery/rollback.

### Just the commands

The build is spelled out command by command in
[walkthrough §2–3](docs/walkthrough.md#step-2--change-the-rootfs). The part you will come back
to is the login:

```sh
# the box's sshd is OpenSSH 6.4: RSA keys only, no ed25519 — an ed25519 key can never log in
ssh-keygen -t rsa -b 4096 -f ~/.ssh/kdbox_rsa -C kdbox

ssh -i ~/.ssh/kdbox_rsa \
    -o PubkeyAcceptedKeyTypes=+ssh-rsa -o HostKeyAlgorithms=+ssh-rsa \
    root@<miner-ip> 'id; cat /etc/kdbox-custom-fw'
```

Use `~/.ssh/kdbox_rsa.pub` as `--pubkey` when you build. Both `-o` flags are mandatory: without
them a modern client refuses RSA-SHA1 user auth and the RSA host key, and you get
`Permission denied (publickey)` or `no matching host key type` on a flash that actually worked.

## Why it works

- The update path is **signature-only**: `minerd` runs
  `openssl dgst -sha256 -verify …/fw_cmpt.verify.pem -signature firmware.ius.sig firmware.ius`
  and checks nothing else — no CRC, no container validation, no image-table check.
- The **private** half of that key ships in the rootfs at `/root/settings/fw_cmpt.pem`,
  encrypted with a vendor passphrase.
- A rootfs you build and sign is therefore accepted exactly like an official package.

The signing credential is an input to this process, not something this repo distributes — see
"About the signing key" below.

## What is in here

```
docs/walkthrough.md        step by step from a stock box, with screenshots of the GUI
docs/troubleshooting.md    symptom -> cause -> fix, and recovery (rollback, direct flash)
docs/signing-key.md        what the signing key is, where it lives, and how to get signing done
docs/ota-signed-flash.md   the procedure in depth: container layout, rootfs fit, sign, wrap, verify
docs/AGENTS.md             Agent Brief (format ab/1) — facts, procedure, gates, failures, rules
CLAUDE.md                  pointer for Claude-family tools
scripts/build-cpb.py       inspect / unpack / patch / sign / repack, with the gates enforced
scripts/flash-ota.sh       mcbversion guard, login, upload, reconnect watch
```

This repo contains **no Goldshell firmware, no vendor binaries, and no keys.**

## Requirements

- Linux (or WSL) with `squashfs-tools`, `openssl`, `python3`
- The **stock package for your unit's `mcbversion`** — check `curl -s http://<ip>/mcb/status`
- An **RSA** SSH public key (`ssh-keygen -t rsa -b 4096 -f ~/.ssh/kdbox_rsa`) — the box's sshd
  predates ed25519 and cannot use it
- The signing key + passphrase (below)

## About the signing key

`build-cpb.py` signs with `KD_FW_SIGNING_KEY` using `KD_FW_PASS`, and verifies the result against
`fw_cmpt.verify.pem` **sitting next to it** — the private half is useless without its public
half, and both are inside any official package:

```
/root/settings/fw_cmpt.pem          AES-encrypted private key, needs the vendor passphrase
/root/settings/fw_cmpt.verify.pem   public half — what the box verifies against
```

This repository deliberately carries neither. The key is extractable from any official package;
the passphrase is the only secret, and it is shared across the vendor's entire product line.
Publishing it would let anyone sign firmware for hardware they do not own.

**How to actually get signing done**, how to verify a package someone hands you, and why the
passphrase is not here: **[docs/signing-key.md](docs/signing-key.md)**.

## Warnings

- **Flashing rewrites the config partition.** Pools are wiped and the web admin password
  returns to the factory default `123456789`.
- **The flash is the irreversible step.** SPL and kernel are untouched, so a rejected package
  leaves the box running, but a rootfs that fails to mount leaves a box that will not come
  back on the network — recovery then needs physical access to the board's console header.
  Keep the stock package.
- **Match the `mcbversion`.** The header name, the inner image and the second container member
  are all version-bound. Rebuild from your unit's stock package; do not rename.
- **The box broadcasts an open WiFi access point while it is plugged in.** With WiFi enabled
  (the factory default) it runs `hostapd` with no encryption — SSID `Goldshell-<suffix>`,
  handing out `192.168.77.x` — so anyone in radio range can reach the web UI. If the miner is
  wired, turn WiFi off. If you need it, give it a WPA2 key.
- **`/mcb/wifisetting` answers without authentication.** It exposes the scan list (SSIDs,
  BSSIDs, security modes) and the WiFi state. No passphrase — there isn't one until you set
  it — but it is free reconnaissance and it is not fixable from this repo.
- **Change the admin password.** It is `123456789` until you do, and the UI is plain HTTP.
  That plus the open AP is the whole attack path.
- **The vendor's SSH host keys are inside every image.** The stock rootfs ships
  `/etc/ssh_host_*_key`, so every KD-Box II presents the same host key: nothing changes in
  `known_hosts` on a reflash, and host-key pinning tells you nothing about which box you
  reached. Compare the `<ip>` you typed, not the fingerprint.

## Not affiliated with Goldshell

This is documentation and tooling for hardware its owners already have. You modify your own
device, at your own risk. MIT licensed.
