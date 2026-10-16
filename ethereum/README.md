# ethereum

**For:** a team that reads the chain all day. Either it pays a hosted RPC by the request and wants its own node, or it already runs reth and Lighthouse in Docker and wants them watched and repaired.

```
subweave run box --on root@eth1
subweave run ethereum "mainnet full node, RPC on the private network" --on root@eth1
subweave run ethereum "mainnet; reth container reth, lighthouse container lighthouse, reth data /mnt/reth, lighthouse data /mnt/lighthouse"
```

Disk room on every filesystem, the clock, the systemd journal, SSH and security updates are the [box](../box) weave's: run it first on each box. `running` comes AFTER `box:room`, `at-tip` AFTER `box:clock`.

| ARG | Level | Default | Says |
|---|---|---|---|
| `network` | MUST | — | `mainnet`, `sepolia` or `hoodi` |
| `reth_container`, `lighthouse_container` | MAY | — | adopt these running containers instead of installing; both or neither |
| `reth_data`, `lighthouse_data` | MAY | `/var/lib/eth/reth`, `/var/lib/eth/lighthouse` | the host directories of the two databases |
| `history` | SHOULD | `full` | `archive`, `full`, `minimal`, or `last <blocks> blocks`; reth keeps at least 10064 |
| `rpc` | SHOULD | `local` | JSON-RPC on `local`host only, the `private` network, or `open` behind a firewall outside the box |
| `updates` | MAY | `fix` | `auto`, `security`, `fix`, `ask` or `off`: see [Updates](#updates) |
| `lag` | MAY | `60s` | how long the newest block may trail the clock |

## Two ways to run

| | Install | Adopt |
|---|---|---|
| Clients | systemd units `reth` and `lighthouse` | the named Docker containers, on host networking |
| Restarts, settings, ports, logs | on its own | asks first, until the person answers `always` |
| Resync | asks first | asks first |
| Installs and updates | as `updates` says | never: the person updates the containers |
| Never | kill reth, delete chain data by hand, open 8551 | the same, and never recreate a container unasked |

Adopted, two rules by effect do this. `NEVER BEFORE CHANGES version` refuses `install` and `update`. `ASK BEFORE CHANGES restart, config, data, network` holds every other action; one `always` answers it for all of them.

## Goals

```
installed
├─ running        also after box:room
│  ├─ synced
│  │  ├─ at-tip   also after box:clock
│  │  ├─ current   IF updates == 'auto' OR updates == 'ask'
│  │  └─ patched   IF updates == 'auto' OR updates == 'security' OR updates == 'ask'
│  ├─ peered
│  ├─ sealed
│  ├─ fenced
│  └─ quiet
└─ old-logs
```

- `at-tip` reads the age of reth's newest block. `eth_syncing` says `false` even when no consensus client drives reth any more, so it is not the signal.
- `synced` reads Lighthouse's `/eth/v1/node/health` and sync distance.
- `old-logs` reads `box:disk`: on a full disk, container log files over 1 GB go.
- Other services on the box read the node as `ethereum:head_age` and wait on `ethereum:at-tip`.

## Updates

`reth` and `lighthouse` are `PART`s. The running version comes from the client, so adopted containers count too; the newest comes from GitHub releases. `current` reads `reth.behind`, `patched` reads `reth.security`, and `update` reads `reth.notes`, the same for `lighthouse`.

| `updates` | `current` | `patched` | `update` for a failure its notes fix | asks |
|---|---|---|---|---|
| `auto` | on | on | yes | no |
| `security` | off | on | only a security release | no |
| `fix` | off | off | yes | no |
| `ask` | on | on | yes | before each update |
| `off` | off | off | never | — |

- `update` copies the old binaries to `/opt/eth/previous` first.
- `rollback` is the `UNDO` of `update`. It runs when the update fails, or breaks a goal that held, within 2 hours.
- `security` is the decider's reading of the notes. `subweave known <service> reth security|routine <version>` overrides it.

## What happens when

1. **Lighthouse exits.** `running` fails within a minute. Installed: `start` brings it back. Adopted: the person gets the exit code and the last log lines, and answers.
2. **reth's head stops while Lighthouse is synced.** `at-tip` fails. The decider reads the host's IO pressure and load: a build or a backup on the same disks gets `ask`, not a restart. Otherwise `restart-reth` stops reth with a 600-second timeout and starts it.
3. **The request does not name a network.** `network` is MUST: `plan` shows an error, and nothing runs.
4. **The request names only one container.** `NEVER IF` stops it at `plan`.
5. **The request says "and run a validator too".** `NEVER "run a validator or hold staking keys"` refuses the whole request.
6. **First sync.** `install` passes within 30 minutes, then `synced` fails for hours to days; the decider answers `wait`.
7. **A container's log fills the disk.** `box:room` fails, and `running` and what follows it wait. `old-logs` fails; `free-logs` empties container logs over 1 GB, in code.
8. **The coder tries `ufw allow 8551/tcp`.** The gate refuses it ([§5](https://github.com/subweave/subweave/blob/main/LIMITS.md#5-gating-shell-commands-depends-on-their-stated-purpose)). If another change gets through and reth listens beyond localhost, `sealed` fails within a minute.
9. **Lighthouse crashes on a hard fork its release does not support.** `running` fails. The decider reads `lighthouse.notes` and picks `update` over `start`. Installed: it updates, as `updates` allows. Adopted: `update` is refused, and the person gets a card that says why.

## Drills

`subweave drill ethereum <name>` runs one now ([Drills](https://github.com/subweave/subweave/blob/main/WEAVEFILE.md#drills)).

- `cl-stop`: stops Lighthouse. `running` must fail within 3m and hold again within 20m. Every 30 days, only when the weave installed the clients (no `reth_container`): adopted containers are yours.

## Limits

- `NEEDS` asks for 1 TB, which fits `full`, `minimal` and pruned nodes; `archive` needs about 3 TB. The daemon measures the box's largest disk, wherever `reth_data` is.
- Adopted containers keep their owner's images: this weave never updates them. With `updates=fix`, a hard fork shows first as a failure, so the node stops until it updates, or, adopted, until its owner does.
- A release note that hides a migration gets past the `ASK`. `rollback` puts back binaries, not a migrated database.
- No second node: a lost box is a full resync somewhere else ([§9](https://github.com/subweave/subweave/blob/main/LIMITS.md#9-one-box-no-failover)).
- Log lines and release notes go to the decider; a crafted note can try to steer it ([§3](https://github.com/subweave/subweave/blob/main/LIMITS.md#3-text-on-the-box-can-steer-the-decider)).
- No validator, no MEV relay, no slashing protection: the weave refuses them.
