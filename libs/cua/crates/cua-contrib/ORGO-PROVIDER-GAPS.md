# Orgo as a cua contrib provider: gap report

Status: **blocked, not built.** Two Orgo-side gaps make the `cua_sandbox_core::Provider`
contract impossible to meet. Both need Orgo changes, so no provider code is included.
Measured on 2026-10-20 against a real Orgo Linux computer (created for the test and deleted).

## Method mapping (`Provider` trait, modelled on `e2b.rs`)

| `Provider` method | E2B call | Orgo equivalent | Verdict |
|---|---|---|---|
| `check_configured` | `E2B_API_KEY` present | `ORGO_API_KEY` present | OK |
| `create` (image) | build template `fromImage` the pinned registry ref, then `POST /v2/sandboxes` | `POST /computers` boots Orgo's own Ubuntu desktop image; templates are scripted on that image | **BLOCKED (1)** |
| `create` (ports, env, command) | sandbox env vars, entrypoint run at template build, public port URLs | env only via templates/secrets; no entrypoint from an image; ports see (2) | **BLOCKED (1, 2)** |
| `create` (labels) | sandbox `metadata` (`cua.managed`, `cua.name`) | none; only a unique name per workspace | workaround: encode in name |
| `get` | `GET /sandboxes/{id}` | `GET /computers/{id}` (`status`) | OK |
| `list` | `GET /v2/sandboxes?metadata=cua.managed` | `GET /workspaces/{id}` lists every computer; filter by name prefix | OK with name prefix |
| `delete` | `DELETE /sandboxes/{id}` | `DELETE /computers/{id}` | OK |
| `endpoint(instance, port)` | `https://{port}-{id}.e2b.app` | none | **BLOCKED (2)** |
| `suspend` / `resume` | pause / connect | `POST /computers/{id}/suspend`, `/resume` (memory kept) | OK |
| `stop` / `start` (`PowerControl::Stop`) | n/a | `POST /computers/{id}/stop`, `/start` (about 40 s each, disk kept) | OK |
| `keep_alive` | `POST /sandboxes/{id}/timeout` | `PATCH /computers/{id}/resize {auto_stop_minutes}` (idle suspend, plan-gated) | partial: idle timer, not a TTL |
| `capabilities().ports` | `PortExposure::Https` | would be `None` | **BLOCKED (2)** |

## Blocking gaps (Orgo-side)

1. **No registry image.** `Image.linux()` and every `ghcr.io/trycua/*` image carry cua-spacesd and are
   pinned by digest. Orgo cannot run an OCI image: a computer starts from Orgo's base image, and a template
   (`orgo.ai/v1`) only scripts installs and services on top of it. Checked on a live computer: no Docker or
   Podman, no `/dev/kvm`, so the image cannot be run nested either (egress to `ghcr.io` does work).
   Needed: create from an OCI image reference (or a template `from: image`), with a start command.
2. **No per-port HTTPS endpoint.** `PortExposure::Https` means the SDK reaches cua-spacesd (3211) and any
   declared port at an HTTPS URL. On Orgo a guest listener on 3211 was unreachable from outside:
   `host:3211` timed out, `www.orgo.ai/api/desktops/{id}/proxy/3211/` returned 404, and the proxy only
   forwards to the Desktop API on guest port 8080 (`/proxy/health` returned 200). The only published ports are
   template `ports[].public`, which are plain TCP on a host port, declared at template build, and
   **not returned by `GET /computers/{id}`** (only by template test-run). There is no TLS and no hostname.
   Needed: an authenticated HTTPS (and WebSocket) URL per guest port, for example
   `https://{port}-{id}.<orgo domain>` or `/api/desktops/{id}/proxy/port/{n}/...`, plus the port
   mapping in `GET /computers/{id}`.

## Why a workaround is not worth it

Running spacesd in a plain Orgo computer and publishing 3211 as a template TCP port would send the spacesd
token in cleartext to a public host port whose number the API does not return. That is weaker than every
existing contrib provider, so it is not proposed.

## Other differences (not blocking)

- Orgo's shell endpoint merges stdout and stderr and has no stdin or cancel.
- Computers run as root; a dedicated workspace is needed because ownership is by name.
- `auto_stop_minutes` is plan-gated and is an idle suspend, not a lifetime backstop (`ProviderCreate::ttl`).

Owner of the Orgo changes: Tiger. Once 1 and 2 exist, the provider is about 500 lines mirroring `e2b.rs`
(`orgo` feature, `ORGO_API_KEY`, a name prefix `cua-` for `list`, `PortExposure::Https`).
