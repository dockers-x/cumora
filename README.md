# cumora container image

Multi-platform container images for [Cumora](https://github.com/yetone/cumora),
automatically built from upstream `main`.

Images are published to:

- `docker.io/czyt/cumora`
- `ghcr.io/dockers-x/cumora`

## What is in the image

A single Node process serves three surfaces (upstream's
`server/docker/cumora-server.Dockerfile`):

| Path | Surface |
| --- | --- |
| `/api/*` | JSON API (Express) |
| `/runtime/*` | Per-agent runtime API (JWT-authed) |
| `/*` | The React SPA bundle |

The image also ships `kubectl`, which the orchestrator uses to spawn one
Kubernetes pod per cloud agent. Without a cluster the server still boots and
serves the API and SPA; see [Cloud agents](#cloud-agents) below.

## Run

The server needs Postgres (with pgvector) and Redis. The compose stack in this
repository wires all three together:

```bash
cp .env.example .env     # set OPENAI_API_KEY and AGENT_RUNTIME_SECRET
docker compose up -d
open http://localhost:5181
```

Or run the image directly against your own database:

```bash
docker run --rm -p 5181:5181 \
  -e DATABASE_URL="postgres://cumora:${POSTGRES_PASSWORD:?set it}@host.docker.internal:5432/cumora" \
  -e REDIS_URL='redis://host.docker.internal:6379' \
  -e OPENAI_API_KEY='sk-...' \
  -e AGENT_RUNTIME_SECRET="$(openssl rand -hex 32)" \
  ghcr.io/dockers-x/cumora:latest
```

`npm run migrate` is the pre-deploy migration job — run it once against a fresh
database before starting the server. Application containers only verify the
migration ledger and never execute DDL on boot.

## Tags

| Tag | Meaning |
| --- | --- |
| `sha-<7-char commit>` | Exact upstream commit. Immutable; use this for deployments. |
| `<version>` | Upstream `package.json` version (e.g. `0.18.7`). Moves when upstream releases. |
| `latest` | Newest upstream `main` commit. Moves on every upstream push. |

`linux/amd64` and `linux/arm64` are published under the same tag as a manifest
list.

## Environment variables

`DATABASE_URL` and `OPENAI_API_KEY` are required; everything else has a working
default inside the image — except `AGENT_RUNTIME_SECRET`, which the server
refuses to start without once `NODE_ENV=production` (the image's default).

| Variable | Default | Purpose |
| --- | --- | --- |
| `DATABASE_URL` | `postgres://$USER@localhost:5432/cumora` | Postgres DSN. Use a `pgvector` image. |
| `OPENAI_API_KEY` | — | Agent chat turns, classifiers, image generation. |
| `REDIS_URL` | `redis://localhost:6379` | Pub/sub fan-out and presence. |
| `AGENT_RUNTIME_SECRET` | public dev default | Signs per-agent runtime JWTs. **Required in production** — boot fails closed on the dev default. |
| `PORT` | `5181` | HTTP + WebSocket listen port. |
| `PUBLIC_HOST` | empty | Public origin for `/uploads` URLs. Needed for image attachments, since OpenAI fetches them from its own network. |
| `CUMORA_CORS_ORIGINS` | empty | Comma-separated allowed browser origins. |
| `R2_ENDPOINT` / `R2_BUCKET` / `R2_ACCESS_KEY_ID` / `R2_SECRET_ACCESS_KEY` | empty | S3-compatible object storage. All four (or none) — otherwise attachments stay on the local volume. |
| `OPENAI_MODEL` / `OPENAI_MODEL_SUPPORT` | baked defaults | "Brain" and "cerebellum" model pins. |
| `ENABLE_AGENT_POD_GC` | `true` | `false` skips agent-pod garbage collection. |
| `ENABLE_CHROME_PVC_GC` | `true` | `false` skips Chrome-profile PVC collection. |
| `ENABLE_CLUSTER_MONITOR` | `true` | `false` skips the fuse-pressure monitor. |

The full upstream list lives in
[`.env.example`](https://github.com/yetone/cumora/blob/main/.env.example).

## Cloud agents

Cumora's managed agents run as per-agent Kubernetes pods that the server
creates with `kubectl`. A plain Docker host has no cluster, so that path cannot
work — the compose file disables the three kubectl-dependent background jobs
(`ENABLE_AGENT_POD_GC`, `ENABLE_CHROME_PVC_GC`, `ENABLE_CLUSTER_MONITOR`) and
leaves the API, SPA and BYOA paths fully functional.

To run agents without Kubernetes, use **BYOA**: a daemon on your own machine
drives a local Claude Code / Codex / Grok Build / ... as the agent brain. See
upstream's [`docs/BYOA.md`](https://github.com/yetone/cumora/blob/main/docs/BYOA.md).

To run cloud agents properly, deploy the server into Kubernetes with upstream's
[`server/k8s`](https://github.com/yetone/cumora/tree/main/server/k8s) manifests
instead of this compose file.

## Release automation

[`.github/workflows/container.yml`](.github/workflows/container.yml) checks
upstream `main` every six hours. When the commit has not been built yet it:

1. resolves and verifies the upstream commit and its `package.json` version,
2. builds `linux/amd64` and `linux/arm64` natively on GitHub-hosted runners,
3. publishes a manifest list to Docker Hub and GHCR, and
4. tags the image with `sha-<commit>`, the upstream version, and (for the
   current upstream HEAD) `latest`.

The workflow can also be dispatched manually with a specific upstream ref, and
`force_rebuild` republishes a commit after container-only fixes.

Builds use upstream's own `server/docker/cumora-server.Dockerfile` from the
pinned commit, so the image definition never drifts from upstream — this
repository only adds the registry plumbing, the multi-architecture matrix and
the change detection.

## License

The image packages [yetone/cumora](https://github.com/yetone/cumora), which is
MIT licensed.
