# AD4M-Container

A headless container image for the [AD4M](https://ad4m.dev) executor — the agent-centric
peer-to-peer runtime from [coasys/ad4m](https://github.com/coasys/ad4m) — exposing its
REST/WS-RPC API and MCP server.

Published to `ghcr.io/mrhegemon/ad4m-container`.

## Relationship to upstream PR #778

This image **tracks [coasys/ad4m#778](https://github.com/coasys/ad4m/pull/778)**
("feat: Docker image for headless AD4M executor", branch `feat/docker-image`) and is
derived from the Dockerfile and entrypoint on that branch. The build recipe — the git
wrapper that intercepts oversized submodule fetches, the local `deno` / `deno_core` /
`rusty_v8` clones that replace the git dependencies, the floneum cargo-git pre-cache and
the `Cargo.lock` rewrite — and the agent auto-generate / auto-unlock logic in the
entrypoint are upstream's work, kept close to the original so this stays easy to reconcile
when #778 merges.

Two structural differences:

- Upstream's Dockerfile lives *inside* the ad4m repo and does `COPY . /home/builder/ad4m`.
  This repo is separate, so it clones the source at a **pinned commit** (`ARG AD4M_REF`)
  rather than a branch name — the branch can be force-pushed or rebased.
- The API port is read from `AD4M_PORT` instead of being hard-coded in the readiness probe.

**#778 is not merged.** Until it is, this image builds from an unreleased commit.

## Why build from source rather than package a release

PR #778 carries a fix that **no release contains**. On every tag (`v0.12.0`,
`v0.13.0-test-1`) and on `dev`, the MCP server's bind host is hard-coded to `127.0.0.1`:
`McpServerConfig::default()` sets it and `lib.rs` builds the config with
`..Default::default()`, so `--mcp-port` changes the port but never the address, and no
flag exists that does. #778 replaces that with a host derived from `config.localhost`,
which is what makes MCP reachable from outside the container without a socat bridge.

Note the API surface here is **not** the released one. `dev` and this branch serve REST +
WS-RPC under `/api/v1` (WebSocket at `/api/v1/ws`) on port 12000, and GraphQL has been
removed. Released v0.12.0 served GraphQL on port 4000.

## Usage

```bash
docker run -d --name ad4m \
  -p 12000:12000 -p 3001:3001 \
  -v ad4m-data:/data \
  -e ADMIN_CREDENTIAL=change-me \
  -e AGENT_PASSPHRASE=change-me-too \
  -e ENABLE_MCP=true \
  ghcr.io/mrhegemon/ad4m-container:pr778
```

`AGENT_PASSPHRASE` matters: on first boot the entrypoint generates the agent with it, and
on later boots it unlocks the wallet. Without an unlocked wallet,
`wallet.get_secret_key("main")` returns `None` on every JWT-issuing path, so multi-user
signup and login fail with *"main key not found"*. If the agent isn't initialised and no
passphrase is set, the container **fails fast** rather than starting something unusable.

Connect over MCP — the admin credential goes in as a **bearer** token:

```bash
curl -s -X POST http://127.0.0.1:3001/mcp \
  -H 'Content-Type: application/json' \
  -H 'Accept: application/json, text/event-stream' \
  -H 'Authorization: Bearer change-me' \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2024-11-05","capabilities":{},"clientInfo":{"name":"cli","version":"1"}}}'
```

The branch exposes ~45 MCP tools across perspectives, subjects, collections, flows,
languages, neighbourhoods, profiles, auth and subscriptions — `list_perspectives`,
`query_links`, `add_link`, `create_subject`, `query_subjects`, `get_subject_collection`,
`neighbourhood_join_from_url`, `infer`, `get_models`, `request_capability` and more.

## Configuration

| Variable | Default | Purpose |
|---|---|---|
| `ADMIN_CREDENTIAL` | *(unset)* | Admin capability token. Bearer token for MCP. |
| `AGENT_PASSPHRASE` | *(unset)* | Generates the agent on first boot, unlocks it thereafter. Container exits if the agent is uninitialised and this is unset. |
| `AD4M_PORT` | `12000` | REST / WS-RPC port. |
| `ENABLE_MCP` | *(unset)* | Set `true` to run the MCP server. |
| `MCP_PORT` | `3001` | MCP port. Reachable on `0.0.0.0` thanks to the #778 fix. |
| `ENABLE_MULTI_USER` | *(unset)* | Set `true` for multi-user signup/login. |
| `RUN_DAPP_SERVER` | `true` | Serve the bundled dApp on 8080. |

`/data` must be persisted — it holds the agent's keys. Losing it means losing the identity.

## Notes

- **amd64 only.** The toolchain fetches are x86_64-specific, matching upstream.
- The builder compiles `rusty_v8` from source. Upstream reports ~10 min on 32 cores;
  expect substantially longer on a stock CI runner, and note it is disk-hungry (the
  publish workflow frees space before building).
- The container runs as root only long enough to fix `/data` bind-mount ownership, then
  drops to the unprivileged `ad4m` user via `gosu`.
- `Ad4mConfig::prepare()` sets `port = Some(12000)`, which pre-empts `find_and_set_port`'s
  4000 fallback — so 12000 is the real default, not 4000.
- `connect_holochain` defaults to `false` on this branch.

## Updating

Bump `ARG AD4M_REF` in the `Dockerfile` to a newer commit on the PR branch (or, once it
merges, on `dev`), then push to `main` — or run the workflow manually with an `ad4m_ref`
input. Images are tagged `pr778-<short-sha>` plus `pr778` and `latest` on the default
branch, since there is no upstream release version to track yet.

When #778 merges and coasys cuts a release containing it, this repo should switch to
packaging that release — or be retired in favour of an official image.

## Licensing

AD4M is licensed under the **Cryptographic Autonomy License 1.0 (CAL-1.0)**, a strong
copyleft license — see [coasys/ad4m](https://github.com/coasys/ad4m/blob/dev/LICENSE).
This repository contains packaging only (a Dockerfile, an entrypoint and a workflow),
both derived from PR #778; upstream source is at the linked repository.
