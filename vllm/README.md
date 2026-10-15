# vllm

**For:** a product team that serves its own open model instead of paying per token. Example: a support-tools startup with one rented 48 GB GPU, serving Qwen3-8B to its app servers.

```
subweave run box --on root@gpu1
subweave run vllm "serve Qwen/Qwen3-8B to our app servers on the private network" --on root@gpu1 --name gpu1
```

Disk room, the clock, the systemd journal, SSH and Ubuntu's security updates are the [box](../box) weave's: run it first on each box. `serving` comes AFTER `box:room`; `held` keeps the box's updates off the driver and CUDA.

| ARG | Level | Default | Says |
|---|---|---|---|
| `model` | MUST | — | a Hugging Face model id, like `Qwen/Qwen3-8B` |
| `access` | SHOULD | `local` | the API on `local`host only, or on the `private` network with `VLLM_API_KEY` |
| `gated` | MAY | `no` | `yes` for a gated model: the daemon then asks for `HF_TOKEN` once |

## Goals

```
driver
├─ gpu-healthy
└─ installed
   ├─ running
   │  ├─ serving   also after box:room
   │  ├─ keeping-up
   │  └─ quiet
   ├─ old-models
   └─ held
```

## What happens when

1. **The request says "serve a small coding model".** `model` is MUST, and "a small coding model" is no Hugging Face id: `plan` shows an error, and nothing runs.
2. **A new box with a bare GPU.** `driver` fails. `ASK BEFORE install-driver` asks the person on their channel; after yes, the driver installs and the box reboots, `WITHIN 20m`. `install` then gets an hour, for the model download.
3. **Traffic doubles at 09:00.** Preemptions climb and `keeping-up` fails. The decider picks between `lower-concurrency` and `restart` from the KV cache use and the logs; a restart gets 15 minutes to load the weights.
4. **Xid 79 at night.** `gpu-healthy` fails. The `nvidia-xid` skill says reboot, and `ASK BEFORE reboot` asks the person first.
5. **`subweave tell gpu1 "serve Qwen/Qwen3-14B instead"`.** `serving` fails, because `/v1/models` still names the old model, and `serve` moves the service to the new one.

## Drills

A drill breaks the service on purpose; the loop must notice and repair it (`subweave drill vllm <name>` runs one now).

- `stop`: stops vLLM; loading the model again takes minutes. `running` must fail within 3m and hold again within 30m. Every 30 days.

## Limits

- Goals watch the queue, not the answers. A wrong chat template or a bad quantization serves wrong answers with every goal green.
- No software fixes a dead GPU. On a rented GPU, the person opens a ticket with the provider.
- A spot or preemptible GPU disappears with the daemon on it ([LIMITS §9](https://github.com/subweave/subweave/blob/main/LIMITS.md#9-one-box-no-failover)).
- The coder runs as root on a box with your model and tokens ([§11](https://github.com/subweave/subweave/blob/main/LIMITS.md#11-the-coder-is-a-model-with-a-root-shell)).
- vLLM changes flags between releases, so every vLLM upgrade asks.
