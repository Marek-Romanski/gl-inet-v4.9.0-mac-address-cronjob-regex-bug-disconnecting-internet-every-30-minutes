# gl-inet-v4.9.0-mac-address-cronjob-regex-bug-disconnecting-internet-every-30-minutes
GL-Inet GL-MT6000 "Flint 2" router firmware v4.9.0 cronjob bug - internet disconnects every 30 minutes regardless of settings, when Auto Update MAC is set to 'By time'.

## TL;DR
The "Auto update MAC by time" feature saves how often the MAC should change
inside the `mac_mode` uci option (for example `r4320` = random MAC, change it
every 4320 minutes = 3 days). The script that does the changing reads that
number back with a greedy regex, which only ever grabs the **last digit**:

```sh
period=$(echo "$mode" | sed 's/.*\([0-9]\+\).*/\1/')   # "r4320" -> "0"
```

Every preset you can pick in the UI (1 day = 1440, 3 days = 4320, 7 days =
10080 minutes) ends in a zero, so the schedule always becomes "every 0
minutes". A cron job wakes up every 30 minutes just to check whether a MAC
change is due, and with a 0-minute schedule the answer is always yes. So it
changes the WAN MAC and reloads the network on every single tick: a short
internet drop and a new public IP every half hour, for everyone who has the
feature on.

The actual fix is one character class:

```sh
period=$(echo "$mode" | sed 's/[^0-9]*\([0-9]\+\).*/\1/')   # "r4320" -> "4320"
```

Quick fix:

```sh
# Backup the file
cp /usr/sbin/gl-update-cable-mac.sh /root/gl-update-cable-mac.sh.bak
# Replace the faulty regex
sed -i 's|s/\.\*\\(|s/[^0-9]*\\(|g' /usr/sbin/gl-update-cable-mac.sh
```

## Environment

| Item | Value |
|---|---|
| Device | GL.iNet GL-MT6000 (Flint 2) |
| Firmware | v4.9.0 (issue appeared after upgrading from v4.7.x) |
| Affected package | `gl-sdk4-cable` `git-2026.110.34195-928522b-1` |
| Affected script | `/usr/sbin/gl-update-cable-mac.sh` |
| Feature | Network Ports (`/netport`) -> WAN -> MAC mode "Random" + "Auto update MAC" strategy "By time" |

## Symptom

Internet died for roughly 5 seconds at exactly hh:00 and hh:30, every hour,
and the router got a different public IPv4 address after each drop.

## Investigation

### 1. The logs already tell most of the story

Router log at one of the drops (IP addresses sanitized to RFC 5737
documentation ranges):

```text
Thu Jul 23 20:30:01 kernel: mtk_soc_eth 15100000.ethernet eth1: Link is Down
Thu Jul 23 20:30:01 netifd: Interface 'wan' has link connectivity loss
Thu Jul 23 20:30:01 netifd: wan (18839): udhcpc: received SIGTERM
Thu Jul 23 20:30:01 netifd: wan (18839): udhcpc: unicasting a release of 198.51.100.37 to 203.0.113.226
Thu Jul 23 20:30:01 netifd: Interface 'wan' is now down
Thu Jul 23 20:30:01 netifd: Interface 'wan' is disabled
Thu Jul 23 20:30:01 kernel: eth1: PHY [mdio-bus:01] driver [RTL8221B-VB-CG 2.5Gbps PHY]
Thu Jul 23 20:30:01 netifd: Interface 'wan' is enabled
Thu Jul 23 20:30:04 netifd: Network device 'eth1' link is up
Thu Jul 23 20:30:07 netifd: wan (6825): udhcpc: lease of 198.51.100.42 obtained, lease time 86400
Thu Jul 23 20:30:07 netifd: Interface 'wan' is now up
```

Four things in this log rule out a bad line or the ISP, and point straight at
software on the router:

1. **The timing is too perfect.** Drops at hh:00:01 and hh:30:01, every time.
   Broken cables and flaky ISPs are never this punctual. Cron jobs are: cron
   fires at second :00 and the script needs about a second to act.
2. **`udhcpc: received SIGTERM` + DHCP release.** The DHCP client didn't
   crash or time out, it was told to shut down and give the IP back.
   Something did this on purpose, and handing the lease back is why a new
   public IP arrived after every drop.
3. **`Interface 'wan' is disabled` then `enabled`.** Pulling a cable never
   *disables* the interface. Only software restarting the interface looks
   like this.
4. **The PHY is re-probed.** The `PHY [mdio-bus:01] driver [RTL8221B-VB-CG]`
   line means the ethernet port was closed and reopened by software. The
   ~3 s until `Link is Up` is the two ends renegotiating the link, then DHCP
   needs another ~3 s. That's the whole outage.

### 2. Finding the trigger

```sh
root@GL-MT6000:~# cat /etc/crontabs/root
*/30 * * * * /usr/sbin/gl-update-cable-mac.sh time
```

A job runs every 30 minutes, matching the drops exactly. The script belongs to
the closed-source `gl-sdk4-cable` package:

```sh
root@GL-MT6000:~# opkg search /usr/sbin/gl-update-cable-mac.sh
gl-sdk4-cable - git-2026.110.34195-928522b-1
```

### 3. Ruling out configuration

The UI had WAN MAC mode "Random" with "Auto update MAC" strategy "By time" and
a 3-day cycle. Two experiments:

- Changing the cycle from 3 days to 7 days changed nothing. The cron line is
  only a checker that wakes every 30 minutes; the cycle is enforced inside the
  script, so this was expected, but it proved the configured cycle was being
  ignored.
- The drops happened every 30 minutes regardless, and the new IP after each
  drop proved the MAC really was rotating every 30 minutes, not every 3 days.

### 4. Reading the script

The relevant configuration looks like this:

```text
network.@device[6].mac_mode='r4320'        # random, rotate every 4320 min (3 days)
network.@device[6].mac_expire='1785094839' # epoch time of the next rotation
```

And the relevant logic in `/usr/sbin/gl-update-cable-mac.sh` (excerpt):

```sh
if [ "$now" -lt "$expire" ]; then
    return                                             # not due yet -> do nothing
fi

period=$(echo "$mode" | sed 's/.*\([0-9]\+\).*/\1/')   # extract minutes from mode
new_expire=$((now + period * 60))                      # schedule next rotation
uci set "network.$device.mac_expire=$new_expire"
# ... generate random MAC, uci set macaddr ...
/etc/init.d/network reload                             # this bounces the WAN
```

## Root cause

In a POSIX basic regular expression, the leading `.*` is greedy: it consumes as
much as possible while still letting the rest of the pattern match. The rest
only needs `[0-9]\+` to match at least one digit, so for `r4320` the `.*` eats
`r432` and the capture group is left with just `0`:

```sh
root@GL-MT6000:~# echo "r4320" | sed 's/.*\([0-9]\+\).*/\1/'
0
```

With `period=0`:

```sh
new_expire=$((now + 0 * 60))   # = now, i.e. already expired
```

So on every 30-minute cron tick the schedule check `now < expire` fails, a new
random MAC is generated, and `/etc/init.d/network reload` bounces the WAN. All
UI presets are whole days or hours, so the minute value always ends in `0` and
every "By time" user gets a 30-minute rotation instead of their configured
cycle. (Even a value ending in a non-zero digit would still be wrong, just
less noticeable: only that final digit would be used as the period.)

## The fix

Anchor the regex so the capture group takes the first full run of digits:

```diff
-period=$(echo "$mode" | sed 's/.*\([0-9]\+\).*/\1/')
+period=$(echo "$mode" | sed 's/[^0-9]*\([0-9]\+\).*/\1/')
```

```sh
root@GL-MT6000:~# echo "r4320" | sed 's/[^0-9]*\([0-9]\+\).*/\1/'
4320
```

The pattern occurs twice in the script (once in the per-device function, once
in the WAN fallback branch). Apply on the router over SSH manually:

```sh
cp /usr/sbin/gl-update-cable-mac.sh /root/gl-update-cable-mac.sh.bak
sed -i 's|s/\.\*\\(|s/[^0-9]*\\(|g' /usr/sbin/gl-update-cable-mac.sh
grep -n 'period=' /usr/sbin/gl-update-cable-mac.sh   # both lines must show [^0-9]*\([0-9]\+\)
```

## Verification

```sh
# force the schedule to be expired, then simulate one cron tick
root@GL-MT6000:~# uci set network.@device[6].mac_expire=0; uci commit network
root@GL-MT6000:~# /usr/sbin/gl-update-cable-mac.sh time
# one controlled WAN bounce happens (expected: rotation was due)

# the next rotation is now a full cycle away instead of "already expired"
root@GL-MT6000:~# NOW=$(date +%s); EXP=$(uci get network.@device[6].mac_expire); \
                  echo "next rotation in $(( (EXP-NOW)/3600 )) hours"
next rotation in 71 hours

# a second immediate run is a no-op: no MAC change, no WAN bounce
root@GL-MT6000:~# /usr/sbin/gl-update-cable-mac.sh time
root@GL-MT6000:~# logread | grep -iE "wan|eth1" | tail -5
# (no new link-down events)
```

Before the fix, every run changed the MAC. After the fix, the script only
acts when the configured cycle has really passed, and the half-hour drops are
gone.

## Workarounds and caveats

- If you don't care about MAC rotation at all, just switch the feature off in the web UI. The cron line stays,
  but the script wakes up, sees "feature off", and quits without doing anything.
- The fix is not permanent: a firmware upgrade overwrites your patched file with the original buggy one.
  After every firmware update you must run the sed command again, until GL.iNet fixes it in their firmware.
- The script refuses to rotate until the router has synced its clock over the internet (NTP).
  Right after boot, before time sync, it does nothing.
- One-time thing: because the bug kept setting "next rotation" to "right now", the first tick after patching
  still sees an overdue rotation and does one last MAC change (one short drop), and only then schedules properly.

## Lessons learned

- Failures aligned to wall-clock boundaries are scheduled tasks. Check `/etc/crontabs/` first.
- A DHCP *release* plus a new IP after every drop means someone tore the
  interface down on purpose (or the MAC changed).
- A one-character-class bug in a vendor script can look exactly like an ISP
  outage.

## Files in this repo

| File | Purpose |
|---|---|
| `README.md` | This write-up |
| `gl-update-cable-mac.patch` | The minimal diff, for reference |

The full `gl-update-cable-mac.sh` is not redistributed here because
`gl-sdk4-cable` is closed-source GL.iNet code; the minimal diff above is
sufficient to understand and fix the bug.

## Status

- [ ] Reported to GL.iNet (forum.gl-inet.com): link TBD
- [x] Verified fixed locally on GL-MT6000, firmware v4.9.0
