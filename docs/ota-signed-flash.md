# Signed OTA install: root SSH on a stock unit

This is the path for a **stock, sealed KD-Box II** — no case opening, no wiring, no `dd`, no
soldering. It works because the box's own firmware update path accepts any package
whose signature verifies, and the key it verifies against ships inside the box.

Two ways to use this document:

- **You already have root** (a previous build) → you can also patch the image in place; the
  OTA steps below still apply.
- **You only have the web UI** → read it anyway. Everything up to and including flashing is
  done from the web UI; the only part that needs a PC is *building* the image.

> **The flash is the irreversible step.** A rejected package leaves the box running, but a
> rootfs that fails to mount leaves a box that will not come back on the network. Keep the
> stock package — it is your only undo that does not involve physical access.

---

## Why this works

The management daemon (`minerd`) verifies update packages like this:

```c
sprintf(cmd, "openssl dgst -sha256 -verify %sfw_cmpt.verify.pem -signature %s.sig %s",
        "/mnt/config/minerd/settings/", ius_path, ius_path);
system(cmd);
```

- **Signature only.** No CRC, no length check, no table validation in that path.
- `fw_cmpt.verify.pem` is the public half of the vendor's OTA key.
- The **private** half (`fw_cmpt.pem`) ships inside the rootfs at `/root/settings/`.

So a package you build and sign with that key is accepted exactly like an official one.

### Where the signing key comes from

This repository does **not** ship the private key or its passphrase. You supply them, from
your own hardware or your own copy of an official package:

1. Take the official `firmware-<model>-<ver>.cpb` for your model (Goldshell publishes these).
2. Unpack the rootfs. It is the squashfs that starts at `0x389E00` inside `firmware.ius`;
   `build-cpb.py --unpack` writes it out for you.
3. `/root/settings/fw_cmpt.pem` is a 4096-bit RSA private key, AES-encrypted PEM. Copy
   `fw_cmpt.verify.pem` out with it — `build-cpb.py` verifies its own signature against that
   file, so it must sit beside the private key.

You still need the passphrase. It is a single vendor-chosen string, the same on every unit, and
it is deliberately **not** reproduced here: the key is extractable from any public package, so
the passphrase is the one thing standing between a reader and the ability to sign for **every
Goldshell model**. [Why, in full](signing-key.md#why-this-repo-does-not-just-include-the-passphrase).

If you cannot obtain it, the alternatives are:

- use a prebuilt signed image from someone who has one and trusts you with their SSH key, or
- ask that person to sign a build that carries **your** public key.

---

## What you need

| item | why |
|---|---|
| Linux host (or WSL) | `squashfs-tools`, `openssl`, `python3` |
| the stock `.cpb` for **your** MCB version | source of the container, firmware and the second member |
| the vendor signing key + passphrase | see above |
| `fw_cmpt.verify.pem` | the matching public half — `build-cpb.py` verifies against it, so it must sit next to the private key |
| your SSH **RSA** public key | becomes the only key the image trusts (this sshd has no ed25519) |
| a browser or `curl` on the same LAN | to upload the package |

Check the version first:

```sh
curl -s http://<miner-ip>/mcb/status
# {"hardware":"B2.CS.SA","model":"Goldshell-KDBoxⅡ","mcbversion":"MCB_V6_3_5","firmware":"2.2.7"}
```

**`mcbversion` decides everything.** `MCB_V6_3_5` is what the reference build was made for.
A different value means a different internal image and a different rootfs offset — rebuild
from *that unit's* stock package. Do not rename ours.

---

## The container, so you know what you are editing

```
+0x000  512-byte header: "firmware-MCB_V6_3_5.tgz" NUL-padded
+0x200  gzip member 0 → tar{ firmware.ius (16,630,272), firmware.ius.sig (512) }
        …padded to the next 1024-byte boundary (filler is high-entropy, not zeros)
+…      gzip member 1 → tar{ firmware.ius (44,570,112), firmware.ius.sig (512) }
+…      trailer, file padded to 1024
```

Rules that bite:

- **Both members are required.** Ship only member 0 and `minerd` crashes and respawns
  instead of updating — the log shows `Minerd start !!!` and the box stays on the old image.
- Member 1 is the hashrate-board image. Copy it **byte-identical**, trailer included.
- Member 1 is located by scanning for the next gzip magic after member 0, *not* by
  arithmetic: the stock packer inserts filler beyond the minimum alignment.
- After decompression, member 0's tar must be exactly 16,640,000 bytes for 2.2.7.

Inside `firmware.ius`:

```
0x00      "+ius"
0x04      format version (2)
0x10      version string ("2.2.7")
0x1c      image count (6 for MCB_V6_3_5)
0x30      16-byte entries (tag, index, 0x40000, size)
0x389E00  rootfs squashfs   ← patch here, in place
          region 12,919,808 bytes to EOF. The stock image occupies 12,918,848 of
          them, so you have ~960 bytes of slack, not kilobytes.
```

---

## Procedure

`scripts/build-cpb.py` and `scripts/flash-ota.sh` implement the steps below. Read them
before running them — they are short and they check everything they claim to check.

### 1. Inspect, then unpack

```sh
python3 scripts/build-cpb.py --inspect firmware-MCB_V6_3_5-stock.cpb
```

It prints the header name, member sizes, `.ius` version, image count, rootfs region and the
squashfs magic. If that does not look right, stop. `--unpack <dir>` then writes out
`firmware.ius`, its signature, the header, member 1 and the rootfs for editing.

### 2. Modify the rootfs

`unsquashfs`, then apply the minimum set. The commands are in
[walkthrough §2](walkthrough.md#step-2--change-the-rootfs); the *why* for each is here:

| change | why |
|---|---|
| `root/.ssh/authorized_keys` ← **your RSA** public key (700/600) | key-based root login; this sshd has no ed25519 |
| `etc/init.d/S50sshd`: replace the body of `stop()` with an echo | `minerd` runs `S50sshd stop` at boot, which is `killall sshd`. This one call *is* the "SSH disabled" mechanism. |
| `etc/sshd_config`: append `PermitRootLogin yes`, `PasswordAuthentication no`, `UseDNS no` | root login allowed, keys only; `UseDNS no` avoids a 10 s stall |
| `etc/kdbox-custom-fw` marker | tells you from a shell which image booted |
| `usr/bin/wifi-ctl` + `/etc/inittab` respawn swapped | the stock `wifi` binary starts the AP unconditionally; stock's own toggle writes a flag nothing reads. `wifi-ctl` polls that flag and converges the radio (on: AP up; off: `hostapd`/`dnsmasq` killed, `wlan0` down) |

**Fit matters.** The stock image occupies 12,918,848 of the 12,919,808-byte region — about
**960 bytes** of slack, and `mksquashfs` output is not byte-stable, so practically any edit
overflows. (`unsquashfs -s` reports a logical filesystem size of 12,917,584; the difference
is block alignment. Trust the region, not that number.) The reference build drops
`root/yottamc/fonts/materialdesignicons-webfont.64d4cf64.eot` (~1 MB, IE-only format; the
woff2/ttf siblings stay), which frees ~470 KB and measures 12,446,550 B repacked. Never drop
anything `minerd`, `xminerd` or `kdaminer` links against.

Repack with the stock parameters, and assert the output fits the region:

```sh
mksquashfs root rootfs.sqfs \
    -comp gzip -b 131072 -no-xattrs -noappend -all-root -nopad -Xcompression-level 9
```

### 3. Patch, sign, wrap

The command is in [walkthrough §3](walkthrough.md#step-3--build-the-signed-package); the
environment is `KD_FW_SIGNING_KEY` + `KD_FW_PASS`, and `--pubkey` must be the **RSA** key
already inside the image (this sshd is OpenSSH 6.4, no ed25519). What the script does:

1. zero the rootfs region and drop your image at `0x389E00`, leaving length and the image
   table untouched;
2. sign `firmware.ius` with the vendor key and **verify the signature** against the public
   half before continuing;
3. rebuild the `.cpb` with the stock header, your member 0, and the stock member 1 + trailer;
4. run the full verification checklist (see below) and refuse to write the output if any
   check fails — including `--pubkey` not being the key in `root/.ssh/authorized_keys`.

### 4. Verify before flashing

The script enforces all of this; check it reported success, not just that it exited:

1. header byte-identical to stock;
2. member 0 decompresses to exactly 16,640,000 bytes;
3. that tar contains `firmware.ius` (16,630,272) and `firmware.ius.sig` (512);
4. `firmware.ius` is the same length as stock and **every differing byte** is inside
   `[0x389E00, +12,919,808)`;
5. signature verifies against the vendor public key;
6. the squashfs inside the packaged `.ius` mounts and lists `root/.ssh/authorized_keys` and
   `etc/kdbox-custom-fw`, and that `authorized_keys` contains the key you passed as `--pubkey`;
7. member 1 + trailer byte-identical to stock;
8. member 1 starts on a 1024-byte boundary.

### 5. Flash from the web UI

```
http://<miner-ip>/  →  System  →  UNLOCK (admin password)  →  Update
                    →  choose the .cpb  →  UPDATE
```

Default admin password `123456789`; the field accepts `.tgz` and `.cpb`; there is no
confirmation dialog. The click-to-working total is **~3.5 minutes**, and on success the page
ends on `Update process terminated. Please choose the officially issued update package.`
because the daemon kills userspace to write flash — that message means wait and reload, not
re-upload. Screen-by-screen detail: [walkthrough §4–5](walkthrough.md#step-4--get-to-the-update-card).

Scripted equivalent: `scripts/flash-ota.sh <miner-ip> <package.cpb>`.

### 6. After it comes back

```sh
ssh -i ~/.ssh/kdbox_rsa \
    -o PubkeyAcceptedKeyTypes=+ssh-rsa -o HostKeyAlgorithms=+ssh-rsa \
    root@<miner-ip> 'id; cat /etc/kdbox-custom-fw'
```

The legacy options are **mandatory**: the box runs OpenSSH 6.4, RSA host key and RSA user auth
only. Without them a modern client refuses the connection; with an ed25519 key in
`authorized_keys` no flag will help, because this sshd has no ed25519 support at all. Expect
`uid=0(root)` and the marker line.

The flash rewrites the config partition, so **pools are gone** and the web admin password is
back to `123456789`; [walkthrough §7](walkthrough.md#step-7--put-the-box-back-to-work) is the
aftercare checklist. Also turn WiFi off if the miner is wired: the factory default runs an
unauthenticated open access point (`Goldshell-<suffix>`) that reaches the web UI, and
`/mcb/wifisetting` itself answers without authentication and exposes the scan list and state.

---

## When the uploader refuses (observed regression)

On one unit, after running our custom image for a while, *every* upload started failing the
same way — curl and the browser GUI alike:

- `/dev/shm/minerd/download/` ends up with **two files**: `firmware-<mcbversion>.tgz` holding
  exactly 16,445,440 bytes, plus a second file with a garbage name holding most of the rest;
- `minerd` exits and init respawns it (`Minerd start !!!` in syslog) or simply does nothing;
- the box never reboots and keeps running the old image.

The split offset is **fixed** (0xFAF000), independent of package content — we proved it by
uploading a deliberately corrupted package and seeing the identical split. We also confirmed
`/dev/shm` had 123 MB free and accepted a 40 MB test write, so it is not disk space.

**Fallback that worked:** with a root shell (or on a box that already runs this image), write
the repacked rootfs straight to the partition, the way the vendor's own updater does:

```sh
scp new.sqfs root@<miner-ip>:/dev/shm/new.sqfs        # verify md5 after transfer
ssh root@<miner-ip> 'sh -c "trap \"\" HUP; \
    dd if=/dev/shm/new.sqfs of=/dev/spiblock1 bs=65536; \
    sync; sync; reboot"'
```

- `spiblock1` is the rootfs partition (18,874,368 bytes; `/dev/root` symlinks to it).
- `trap "" HUP` matters: if the SSH session dies mid-`dd`, a SIGHUP to a live flash write is
  exactly how you manufacture a brick. Detach first.
- `sync` twice; the second is the one that actually flushes NOR.
- A read-back verify (`dd if=/dev/spiblock1 bs=65536 count=190 | md5sum`) is worth doing
  before the reboot.

After it reboots, run the verification gates again (§6).

## Rolling back

Re-flash the stock `.cpb` through the same updater. Keep it on disk before you start; it is
your only undo that does not involve physical access to the board.

Flashing replaces **the rootfs only** (`spiblock1`). SPL and kernel (`spiblock0`) are never
written by this path, so a bad rootfs still boots far enough to accept an upload in most
cases. If it does not come back on the network, recovery needs physical access to the board's console
header. That is the one scenario this procedure cannot rescue you from.

---

## Gotchas

- **Single-member package** → `minerd` crash/respawn (`Minerd start !!!`), no update, box
  stays on the old image. Harmless, but wasted time.
- **`mksquashfs` without `-all-root`** → image owned by your uid; daemons cannot read config.
- **`mksquashfs` without `-nopad`** → image larger than the region.
- **Member not 1024-aligned** → the package scan never finds it and the update silently
  does not run.
- **Wrong `mcbversion`** → the only reliable way to end up needing physical access. Rebuild,
  do not rename.
- **A wrong password returns HTTP 500**, not 401. That is the login API rejecting you, not a
  broken server.
- **The GUI remembers your last file.** Open the chooser, pick deliberately, then confirm the
  name *and size* in the field before clicking UPDATE.

---

## What this does not buy you

Root does not change the hash engine. The KD-Box II's `ICA585` chips implement blake2s and
nothing else. Blake2b (the SC/Sia line) needs `ICT580` silicon, and the blake2b-sha3 HS line
uses `ICA590`/`ICC590` — no firmware on this image changes what the chips compute.
