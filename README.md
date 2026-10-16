# Subweave library

The official weaves for [Subweave](https://github.com/subweave/subweave): one folder per service, each a `Weavefile` and a `README.md`.

## Run one

```
curl -fsSL https://subweave.com/install.sh | sh
subweave key openrouter        # reads the key from stdin
subweave run box
subweave run postgres "postgres 18 for our app, private network, backups to s3://acme-pg/db1"
```

- `subweave run <name>` reads `<name>/Weavefile` on `main`.
- `subweave run postgres:<tag>` reads git tag `postgres-<tag>`.

## Weaves

- [box](box): run once per box. Disk room, the clock, the journal's size, SSH and security updates, for every service on it.
- [postgres](postgres): PostgreSQL with WAL archiving to S3-compatible storage; the reference weave.
- [ethereum](ethereum): an Ethereum node, reth and Lighthouse, installed or adopted; no validator.
- [vllm](vllm): vLLM serving one open-weights model behind an OpenAI-compatible API.
- [store-edge](store-edge): Telegraf in a shop, buffering metrics while the uplink is down; no coder.
- [valkey](valkey): Valkey as a cache or a small store.
- [minecraft](minecraft): a Paper Minecraft server with world backups and build updates.

Service weaves come AFTER the `box` weave's goals and keep no copy of them. Each weave's README says what it cannot do.

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
