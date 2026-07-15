# NUT + APC UPS on Proxmox: Graceful Shutdown on Power Loss

A from-scratch guide to getting a USB-connected APC UPS to shut your Proxmox host down safely when the power goes out — and to switch the UPS off afterward so everything comes back clean. Written against a real setup (APC Back-UPS BX950MI on a Proxmox host), tested against an actual outage.

This is the standalone case: the UPS plugs into the same machine that monitors it and shuts itself down. Monitoring the UPS from *other* machines on the network is covered as an optional add-on at the end.

## Tested on

- **Proxmox VE:** 9.2.4 (`pveversion`)
- **Kernel:** 7.0.14-4-pve (`uname -r`)
- **NUT:** 2.8.1 (`upsd -V`)
- **Hardware:** APC Back-UPS BX950MI (USB)

These are the versions this was verified against, not hard requirements — the steps should hold across nearby Proxmox and NUT releases, but the killpower specifics are worth re-checking if your NUT major version differs (the `allow_killpower` runtime-flag approach here is current as of NUT 2.8.x).

---

## Contents

1. [How it fits together](#1-how-it-fits-together)
2. [Prerequisites](#2-prerequisites)
3. [Install NUT](#3-install-nut)
4. [Find the right driver](#4-find-the-right-driver)
5. [ups.conf — define the UPS](#5-upsconf--define-the-ups)
6. [upsd.conf — the server](#6-upsdconf--the-server)
7. [upsd.users — logins](#7-upsdusers--logins)
8. [upsmon.conf — the shutdown rules](#8-upsmonconf--the-shutdown-rules)
9. [killpower — switching the UPS off](#9-killpower--switching-the-ups-off)
10. [Start everything](#10-start-everything)
11. [Check it's talking](#11-check-its-talking)
12. [Monthly battery self-test](#12-monthly-battery-self-test)
13. [Troubleshooting](#13-troubleshooting)
14. [Optional: monitoring from a second machine](#14-optional-monitoring-from-a-second-machine)

---

## 1. How it fits together

NUT isn't one program, it's three, and understanding that up front saves you a lot of head-scratching later — because when something breaks, it's almost always one of the three not talking to the next.

| Piece | What it does |
|---|---|
| **Driver** (`usbhid-ups` for USB APC units) | Talks to the UPS over USB, feeds its readings to `upsd` |
| **Server** (`upsd`) | Collects the driver's data, serves it to anything that asks |
| **Monitor** (`upsmon`) | Watches `upsd`, decides when to trigger the shutdown |

The chain during an outage looks like this:

```
Power goes out
  → driver sees the UPS switch to battery, reports "OB" (on battery) to upsd
  → upsmon polls upsd, sees OB
  → battery drops to the low threshold → UPS reports "LB" (low battery)
  → upsmon runs the shutdown command
  → host shuts down cleanly
  → killpower cuts the UPS output so it doesn't drain to zero
```

Three links, three things you can test independently: does the host see the UPS at all, does the monitor notice a state change, and does the full shutdown-plus-killpower sequence complete in order. The last one is the one people skip and the one that actually matters.

---

## 2. Prerequisites

- A USB-connected UPS (this assumes APC — other brands just mean a different driver name in step 4)
- Root or sudo on the host
- The USB cable actually plugged in — separate from the power cable

Before touching NUT, confirm the machine sees the UPS at the USB level:

```bash
lsusb
```

You want a line mentioning the maker — `American Power Conversion` for APC. If it's not there, the USB side isn't connected properly, and nothing downstream will work until it is.

---

## 3. Install NUT

On Proxmox (Debian-based):

```bash
apt update
apt install nut
```

That pulls in both `nut-server` and `nut-client`. Run it directly on the Proxmox host — not in an LXC or VM. It needs raw USB access, and passing that through to a container is more hassle than it's worth for this.

---

## 4. Find the right driver

```bash
nut-scanner -U
```

For pretty much any USB APC unit the answer is `usbhid-ups`, but check rather than assume — the scanner will print the driver name and a port value. Older serial APC units use `apcsmart` instead, and other brands differ entirely.

---

## 5. ups.conf — define the UPS

File: `/etc/nut/ups.conf`

Add at the bottom:

```ini
[apc]
    driver = usbhid-ups
    port = auto
    desc = "APC Back-UPS BX950MI"
    override.battery.charge.low = 45
    allow_killpower = 1
```

`[apc]` is the name this UPS goes by everywhere else in the config — pick it once and stay consistent, because a mismatch here is a classic "why won't it work" bug later.

Two lines are worth calling out. `override.battery.charge.low = 45` tells NUT to treat 45% as the low-battery mark instead of waiting for the UPS's own (often much lower) threshold — that gives the host more runway to shut down properly before the battery's actually in trouble. And `allow_killpower = 1` permits NUT to power the UPS fully off at the end of shutdown, which the killpower service in step 9 relies on.

---

## 6. upsd.conf — the server

File: `/etc/nut/upsd.conf`

```ini
LISTEN 127.0.0.1 3493
```

Localhost only — the server will only answer programs on this same machine. That's what you want for a standalone setup; it's simpler and it isn't exposing anything to the network. Only widen this if you set up the optional second-machine monitoring at the end, and even then, scope it to a trusted subnet rather than throwing it open.

---

## 7. upsd.users — logins

File: `/etc/nut/upsd.users`

```ini
[admin]
    password = choose_a_strong_password
    actions = SET
    instcmds = ALL

[upsmon]
    password = choose_a_different_strong_password
    upsmon primary
```

Swap in real passwords, different ones for each, and note them somewhere — they come up again in the next two steps.

`admin` is the account you'll use by hand for testing and manual commands. `upsmon` is what the monitor uses internally. `upsmon primary` marks this host as the one responsible for triggering shutdown, as opposed to just observing another host's UPS.

Then lock the file down, since it holds passwords:

```bash
chmod 640 /etc/nut/upsd.users
```

---

## 8. upsmon.conf — the shutdown rules

File: `/etc/nut/upsmon.conf`

This is the important one — it defines what actually happens when the power's out.

```ini
MONITOR apc@localhost 1 upsmon choose_a_different_strong_password primary

MINSUPPLIES 1
SHUTDOWNCMD "/sbin/shutdown -h +0"
POWERDOWNFLAG "/etc/killpower"
POLLFREQ 5
POLLFREQALERT 5
HOSTSYNC 15
DEADTIME 15
FINALDELAY 5
```

The password in the `MONITOR` line has to match the `upsmon` password from step 7 exactly — get it wrong and the monitor can't log in to watch the UPS.

The lines that matter most:

| Directive | What it controls |
|---|---|
| `MONITOR apc@localhost 1 ... primary` | The `1` is how many power supplies this UPS represents (one, for a single UPS); `primary` means this host owns the shutdown call |
| `SHUTDOWNCMD` | The literal command run when it's time to go down — `shutdown -h +0` is immediate |
| `POWERDOWNFLAG` | Path to a marker file `upsmon` drops right before shutdown, so the system remembers to also switch the UPS off |
| `DEADTIME` | How many seconds without contact before the UPS is declared unreachable — set it too low and a brief USB hiccup can trigger a false shutdown |
| `FINALDELAY` | Grace period before shutdown actually fires once triggered |

---

## 9. killpower — switching the UPS off

Here's the problem this solves. Without it, the host shuts down fine — but the UPS just keeps running on battery, powering nothing, until it drains flat. If the outage is still going at that point, recovery is messier than it needs to be. Killpower tells the UPS to switch itself off once the host is down, so when mains power returns the UPS wakes up fresh and the host cold-boots clean.

For USB APC units there's a wrinkle: the driver needs the `allow_killpower` flag set at runtime before the UPS will honour the kill command, and that flag doesn't reliably survive a driver restart. The fix is a small service that re-sets the flag every time the driver comes up — with a retry loop, because the UPS isn't always ready to answer the instant the driver starts.

Create `/etc/systemd/system/nut-killpower.service`:

```ini
[Unit]
Description=Set NUT allow_killpower flag
After=nut-driver@apc.service
PartOf=nut-driver@apc.service

[Service]
Type=oneshot
ExecStart=/bin/sh -c '\
for i in $(seq 1 30); do \
  OUT=$(/usr/bin/upsrw -s driver.flag.allow_killpower=1 -u admin -p choose_a_strong_password apc@localhost 2>&1); \
  RC=$?; \
  logger -t nut-killpower "attempt $i: rc=$RC output=$OUT"; \
  if [ "$RC" -eq 0 ]; then break; fi; \
  sleep 2; \
done'
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target nut-driver@apc.service
```

Use your real `admin` password in the `ExecStart` line. Then enable it:

```bash
systemctl daemon-reload
systemctl enable nut-killpower.service
```

The `PartOf` and `After` tie this to the driver's lifecycle, so it re-runs whenever the driver restarts — which is exactly when the flag would otherwise get lost. It's a bit inelegant with the 30-attempt loop, but it's dependable, and dependable is what you want from the one piece you only find out failed during an actual blackout.

---

## 10. Start everything

```bash
systemctl enable nut-server nut-client
systemctl start nut-server nut-client
```

---

## 11. Check it's talking

```bash
upsc apc@localhost
```

You should get a full readout. The fields worth eyeballing:

```
battery.charge: 100
battery.runtime: <seconds>
ups.status: OL
device.mfr: American Power Conversion
device.model: Back-UPS BX950MI
```

`ups.status: OL` is "on line" — mains fine, UPS idling. That's the healthy resting state.

**Testing it, safest first:**

The harmless test is to pull the UPS plug from the *wall* (not the host from the UPS). The host stays up on battery; `ups.status` should flip to `OB`. Plug it back in and it returns to `OL`. That confirms detection works without triggering anything.

The full test actually runs the shutdown, so only do it when you're ready for the host to go down:

```bash
upsmon -c fsd
```

That fires the complete sequence exactly as a real outage would — shutdown triggers, killpower cuts output, and the host should cold-boot clean when power's back. Worth doing once, deliberately, so you've seen the whole chain work end to end rather than trusting it on paper.

---

## 12. Monthly battery self-test

UPS batteries degrade quietly, and the day you want to discover a dead battery is any day except during a real outage. A scheduled self-test forces a brief real discharge to check actual capacity.

```bash
crontab -e
```

```cron
# 1st of every month, 3am — quick battery self-test
0 3 1 * * /usr/bin/upscmd -u admin -p choose_a_strong_password apc test.battery.start.quick
```

Check how it went whenever:

```bash
upsc apc@localhost ups.test.result
```

Results read as `Done and passed`, `Done and warning`, or `In progress`.

---

## 13. Troubleshooting

| Symptom | Likely cause |
|---|---|
| `upsc` returns "Data stale" | Driver isn't running — check `upsdrvctl status`, reseat the USB cable |
| `Connection failure: Connection refused` | `upsd` isn't running, or wrong bind address in `upsd.conf` |
| Driver won't claim the device | Something else is holding the USB port — commonly a leftover `apcupsd`; `apt remove apcupsd` if you're migrating from it |
| Shutdown triggers but UPS never powers off | killpower didn't run, or `POWERDOWNFLAG` path doesn't match between `upsmon.conf` and the service |
| Host doesn't boot back when power returns | Check the UPS's own auto-restart-on-AC setting (varies by model, sometimes a firmware/physical option on the unit) |
| Boot log shows an "ordering cycle" note involving `nut-driver@apc` and `nut-server` | systemd broke a dependency loop on stop; usually harmless, but worth confirming both come up cleanly after a reboot with `systemctl status nut-server nut-driver@apc.service` |

When you're stuck, this is the command that usually names the actual problem — the last handful of lines tend to spell it out:

```bash
journalctl -u nut-server -u nut-client -n 20
```

---

## 14. Optional: monitoring from a second machine

If you want a lightweight always-on node to also see UPS status without its own physical connection to the UPS:

**On the primary (UPS-connected) host**, widen the bind in `upsd.conf` — scoped to a trusted subnet, not wide open:

```ini
LISTEN 0.0.0.0 3493
```

Add a read-only user in `upsd.users`:

```ini
[secondary]
    password = another_strong_password
```

**On the second host**, install only `nut-client`, then in its `upsmon.conf`:

```ini
MONITOR apc@<primary-host-ip> 1 secondary another_strong_password secondary
```

`secondary` (rather than `primary`) means this host watches status but doesn't own the shutdown decision — the primary host still handles that.

---

## Notes for your own setup

- Confirm the driver string with `nut-scanner -U` rather than assuming `usbhid-ups`
- If you bind `upsd` to anything beyond localhost, firewall port 3493 to trusted hosts only
- Every password here is a placeholder — swap in real ones, and don't commit the real `upsd.users` to a public repo (keep a redacted `.example` copy instead)

---

*Built and running on a Proxmox host with an APC Back-UPS BX950MI. The full shutdown chain — detection, graceful shutdown, and killpower — has been verified against a real power outage, and the monthly battery self-test runs on schedule.*
