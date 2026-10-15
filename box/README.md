# box

**For:** every box that runs a weave. It keeps what all the box's services share, once.

```
subweave run box
```

Run it once per box, before or beside the service weaves. They come AFTER its goals (`AFTER box:room`) and read its facts (`box:disk`); `subweave plan` and `subweave run` refuse a service weave that names `box` on a box without it ([Composition](https://github.com/subweave/subweave/blob/main/WEAVEFILE.md#composition)).

| ARG | Level | Default | Says |
|---|---|---|---|
| `journal_max` | MAY | `1GB` | the systemd journal's cap: journald's `SystemMaxUse` and the one vacuum size |
| `ssh_port` | MAY | `22` | the port `ssh` checks and the firewall keeps open |
| `updates` | MAY | `auto` | `auto`: unattended upgrades; `ask`: each round of security updates waits for the person |

## Goals

```
room       every filesystem is less than 85% full
journal    the systemd journal stays under $journal_max
clock      NTP keeps the clock right
ssh        SSH listens on $ssh_port and ufw lets it in
patched    unattended-upgrades is on            IF updates == 'auto'
secure     no security update waits
```

| Fact | Type | Reads |
|---|---|---|
| `disk` | `NUMBER %` | the fullest mounted filesystem, tmpfs and image filesystems left out |
| `journal_mb` | `NUMBER MB` | `/var/log/journal` and `/run/log/journal` |
| `pending` | `NUMBER` | security updates `apt list --upgradable` shows, without packages a weave holds back from unattended upgrades |

## Actions

| Action | For | Changes |
|---|---|---|
| `clean` | `room`, when the apt cache holds over 100 MB | data |
| `cap-journal` | `journal`: a journald drop-in, then one vacuum to `$journal_max` | config, data |
| `sync-clock` | `clock` | config |
| `start-ssh` | `ssh`, when nothing listens | restart |
| `open-ssh` | `ssh`, when ufw shuts it out | network |
| `unattend` | `patched` | config |
| `upgrade` | `secure`: `unattended-upgrade` now | version, restart |

With `updates` `ask`, `ASK BEFORE CHANGES version` holds `upgrade` for the person.

## What happens when

1. **A service's data fills a disk.** `room` fails. With a large apt cache, `clean` runs; otherwise the person is asked: the box never deletes a service's files. Every service goal AFTER `box:room` waits meanwhile, with no repair and no ask of its own.
2. **The clock drifts.** `clock` fails, `sync-clock` turns NTP on. Goals AFTER `box:clock` wait until it holds.
3. **A firewall change shuts SSH out.** `ssh` fails, `open-ssh` allows the port again.
4. **A security update is published.** `pending` rises; within 6 hours `secure` fails and `upgrade` installs it.

## Limits

- `disk` is one number for all filesystems: a goal reading `box:disk` cannot tell which one is full.
- `pending` is as fresh as apt's package lists, which unattended-upgrades updates once a day.
- `room` frees only the apt cache and the journal; what a service stores is the service weave's, or the person's.
- Ubuntu and Debian only: `ufw`, `apt` and `timedatectl`.
