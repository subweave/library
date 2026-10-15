# ethereum

**For:** a team that reads the chain all day. Either it pays a hosted RPC by the request and wants its own node, or it already runs reth and Lighthouse in Docker and wants them watched and repaired.

```
subweave run ethereum "mainnet full node, RPC on the private network" --on root@eth1
subweave run ethereum "mainnet; reth container reth, lighthouse container lighthouse, reth data /mnt/reth, lighthouse data /mnt/lighthouse"
```

| ARG | Level | Default | Says |
|---|---|---|---|
| `network` | MUST | — | `mainnet`, `sepolia` or `hoodi` |
| `reth_container`, `lighthouse_container` | MAY | — | adopt these running containers instead of installing; both or neither |
| `reth_data`, `lighthouse_data` | MAY | `/var/lib/eth/reth`, `/var/lib/eth/lighthouse` | the host directories of the two databases |
| `history` | SHOULD | `full` | `archive`, `full`, `minimal`, or `last <blocks> blocks`; reth keeps at least 10064 |
| `rpc` | SHOULD | `local` | JSON-RPC on `local`host only, the `private` network, or `open` behind a firewall outside the box |
| `updates` | MAY | `auto` | `auto` installs new releases; `ask` asks before each update |
| `lag` | MAY | `60` | seconds the newest block may trail the clock |

## Two ways to run

| | Install | Adopt |
|---|---|---|
| Clients | systemd units `reth` and `lighthouse` | the named Docker containers, on host networking |
| Starts and restarts | on its own | asks first, until the person answers `always` |
| Updates | `update` with signature checks, `rollback` | a newer release goes to the person |
| Resync, new peers, `fix` | on its own (`resync` asks) | asks first |
| Never | kill reth, delete chain data, open 8551 | the same, and never recreate a container unasked |

## Goals

```
installed
├─ running
│  ├─ synced
│  │  ├─ at-tip
│  │  └─ current
│  ├─ peered
│  ├─ sealed
│  ├─ fenced
│  └─ quiet
├─ room
└─ clock
```

- `at-tip` reads the age of reth's newest block. `eth_syncing` says `false` even when no consensus client drives reth any more, so it is not the signal.
- `synced` reads Lighthouse's `/eth/v1/node/health` and sync distance.
- `room` checks both data directories and the log disks.

## What happens when

1. **Lighthouse exits.** `running` fails within a minute. Installed: `start` brings it back. Adopted: the person gets the exit code and the last log lines, and answers.
2. **reth's head stops while Lighthouse is synced.** `at-tip` fails. The decider reads the host's IO pressure and load: a build or a backup on the same disks gets `ask`, not a restart. Otherwise `restart-reth` stops reth with a 600-second timeout and starts it.
3. **The request does not name a network.** `network` is MUST: `plan` shows an error, and nothing runs.
4. **The request names only one container.** `NEVER IF` stops it at `plan`.
5. **The request says "and run a validator too".** `NEVER "run a validator or hold staking keys"` refuses the whole request.
6. **First sync.** `install` passes within 30 minutes, then `synced` fails for hours to days; the decider answers `wait`.
7. **A container's log fills the disk.** `room` fails; `free-logs` vacuums the journal and empties container logs over 1 GB, in code.
8. **The coder tries `ufw allow 8551/tcp`.** The gate refuses it ([§5](https://github.com/subweave/subweave/blob/main/LIMITS.md#5-gating-shell-commands-depends-on-their-stated-purpose)). If another change gets through and reth listens beyond localhost, `sealed` fails within a minute.

## Limits

- `NEEDS` asks for 2 TB, which fits `full` and pruned nodes. The daemon measures the disk of its own state directory, not `reth_data`.
- Adopted containers keep their owner's images: this weave never upgrades them, and a node left behind a hard fork follows a dead chain until the person updates it.
- A release note that hides a migration gets past the `ASK`. `rollback` puts back binaries, not a migrated database.
- No second node: a lost box is a full resync somewhere else ([§9](https://github.com/subweave/subweave/blob/main/LIMITS.md#9-one-box-no-failover)).
- Log lines and release notes go to the decider; a crafted note can try to steer it ([§3](https://github.com/subweave/subweave/blob/main/LIMITS.md#3-text-on-the-box-can-steer-the-decider)).
- No validator, no MEV relay, no slashing protection: the weave refuses them.
