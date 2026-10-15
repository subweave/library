# Subweave library

The official weaves for [Subweave](https://github.com/subweave/subweave): one folder per service, each a `Weavefile` and a `README.md`.

## Run one

```
curl -fsSL https://subweave.com/install.sh | sh
subweave run postgres "postgres 18 for our app, private network, backups to s3://acme-pg/db1"
```

- `subweave run <name>` reads `<name>/Weavefile` on `main`.
- `subweave run postgres:<tag>` reads git tag `postgres-<tag>`.

## Weaves

| Weave | Runs | For | Box needs |
|---|---|---|---|
| [postgres](postgres) | PostgreSQL with continuous WAL archiving to S3-compatible storage | a 6-person SaaS team with no DBA | 2 CPU, 4 GB RAM, 40 GB disk |
| [ethereum](ethereum) | Ethereum node: reth and lighthouse, no validator | a wallet startup leaving hosted RPC | 8 CPU, 32 GB RAM, 1 TB disk |
| [vllm](vllm) | vLLM serving one open-weights model behind an OpenAI-compatible API | a team serving its own open model | 8 CPU, 1 GPU, 64 GB RAM, 500 GB disk |
| [store-edge](store-edge) | Telegraf in a shop: forwards metrics, buffers them while the uplink is down | a 300-shop retail chain | 2 CPU, 4 GB RAM, 64 GB disk |
| [valkey](valkey) | Valkey as a cache or a small store | an agency with 40 client boxes | 1 CPU, 2 GB RAM, 20 GB disk |
| [minecraft](minecraft) | Paper Minecraft server with RCON, world backups and build updates | a teacher with a coding club | 4 CPU, 8 GB RAM, 50 GB disk |

`postgres` is the reference weave: skills, restore drills, a major upgrade by sentence. Each weave's README says what it cannot do.

## Write or check a weave

- Format: [WEAVEFILE.md](https://github.com/subweave/subweave/blob/main/WEAVEFILE.md)
- What Subweave does badly: [LIMITS.md](https://github.com/subweave/subweave/blob/main/LIMITS.md)
- `subweave review <dir>` scores a weave against the authoring rules.

## Add a weave

1. One folder, named after the weave.
2. In it, a `Weavefile` and a `README.md`.
3. `subweave review <folder>` passes.

## License

[MIT](LICENSE)
