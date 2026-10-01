# WirePlumber HFP Conflict — Setup Guide

## The Problem

Modern Linux systems running PipeWire use **WirePlumber** as the session manager.
By default WirePlumber registers its own HFP Hands-Free profile with BlueZ.

When HandsFree tries to register the same UUID (`0000111e-...`), BlueZ returns:

```
org.bluez.Error.AlreadyExists
```

The conflict happens at **profile registration**, not at the audio-node level:
HandsFree owns the HFP profile (and the SCO transport) directly via BlueZ D-Bus,
so WirePlumber must be kept out of HFP entirely.

> **WirePlumber 0.5 note.** Lua configuration files (`*.lua` in
> `/etc/wireplumber/bluetooth.lua.d/`) are **no longer supported** since
> WirePlumber 0.5. The old `node.disabled` rule (kept at the end for 0.4
> reference) is silently ignored on 0.5 and only produces a warning.
> Use the SPA-JSON drop-in shipped in this repo instead.

## Solution (WirePlumber 0.5+)

The config is versioned in this repo at
[`config/wireplumber/51-disable-hfp.conf`](../config/wireplumber/51-disable-hfp.conf).
It declares an explicit role allow-list that excludes HFP/HSP, so WirePlumber
never registers the HFP profile.

Install it into the user WirePlumber config directory:

```bash
# 1. Retire any stale 0.4 Lua file (ignored on 0.5, but keeps warning)
sudo mv -v /etc/wireplumber/bluetooth.lua.d/51-handsfree-hfp.lua{,.disabled} 2>/dev/null || true

# 2. Install the repo config (creates the target dir if missing)
install -Dvm644 config/wireplumber/51-disable-hfp.conf \
  ~/.config/wireplumber/wireplumber.conf.d/51-disable-hfp.conf

# 3. Restart the session manager and watch the log live
systemctl --user restart wireplumber
journalctl --user -u wireplumber -f      # expect: no Lua warning, no HFP errors
```

> Confirm the exact property syntax against the WirePlumber 0.5 migration
> guide if a future minor version changes it:
> https://pipewire.pages.freedesktop.org/wireplumber/daemon/configuration/migration.html

## Verify

With a phone paired, WirePlumber must expose **no** HFP node:

```bash
pw-cli ls Node | grep -i hfp             # expected: empty output
```

Empty output means the HFP profile is free for HandsFree to register.
After launching HandsFree, the HFP source appears and is owned by the app
(not by WirePlumber).

## Audio Routing

With the role allow-list above, **HandsFree owns the HFP SCO transport**
end-to-end: it registers its own BlueZ profile, establishes the SCO link,
and exposes/records the call audio. WirePlumber is not involved in HFP.

- HandsFree registers the HFP profile and handles AT commands / call control.
- HandsFree establishes and consumes the SCO audio link.
- WirePlumber manages only A2DP for other Bluetooth audio use.

> **To confirm at first call (gate test).** The division above is the intended
> model; verify empirically that, once WirePlumber is out, HandsFree actually
> establishes SCO and captures non-silent audio on both channels. If SCO does
> not come up, revisit whether the SCO link must instead be delegated to
> WirePlumber (and this section rewritten accordingly).

## Alternative: Run HandsFree Before WirePlumber Connects

If HandsFree starts on login (before WirePlumber connects to BlueZ), it
registers the HFP profile first. This is a fallback and is **not needed**
when the role allow-list above is in place; prefer the config approach,
which is deterministic.

```bash
# ~/.config/autostart/handsfree.desktop
[Desktop Entry]
Type=Application
Name=HandsFree
Exec=python3 /path/to/HandsFree-Linux/main.py
Hidden=false
X-GNOME-Autostart-enabled=true
```

## Legacy reference — WirePlumber 0.4 (deprecated)

The original 0.4 approach disabled the resulting HFP *nodes* (treating the
symptom rather than the profile registration). Kept here only for users still
on 0.4; **does not work on 0.5**:

```lua
-- /etc/wireplumber/bluetooth.lua.d/51-handsfree-hfp.lua  (0.4 ONLY)
rule = {
  matches = { { { "node.name", "matches", "*hfp*" }, }, },
  apply_properties = { ["node.disabled"] = true, },
}
table.insert(alsa_monitor.rules, rule)
```
