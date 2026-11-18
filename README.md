# Agent gateway workspace

Entry point for an open-source project that lets a player's own AI agent act safely inside a game.

A game studio describes its actions in a manifest. A gateway turns them into
[MCP](https://modelcontextprotocol.io) tools, enforces safety policy (tiers, rate limits, daily
caps, cooldowns), calls the studio's backend with a signed request, and audits every attempt.
This workspace holds the gateway and the fictional game used to test it.

## What is in here

| Folder | What it is | Language | Repo |
|---|---|---|---|
| `agentgw/` | The product: the gateway, its signing library, a manifest CLI and a fake backend | Go | [Moustafa-Elgammal/agentgw](https://github.com/Moustafa-Elgammal/agentgw) |
| `tinycity-mock/` | TinyCity, a fictional city-builder MCP server used only as a test target | Python | [Moustafa-Elgammal/tinycity-mock](https://github.com/Moustafa-Elgammal/tinycity-mock) (private) |

Each folder is a git submodule with its own history, license and `CLAUDE.md`. This root repo
only pins the two projects and provides a shared `docker-compose.yml`.

## How the pieces fit

```
 player's AI agent
        |  MCP (streamable HTTP at /mcp)
        v
 +--------------------------- agentgw ---------------------------+
 |  lookup -> rate limit -> validate -> reserve -> signed call    |
 |                                  -> settle -> audit            |
 +----------------------------------------------------------------+
        |  HMAC-signed REST request
        v
 game backend  (here: agentgw/examples/city, a fake backend that verifies signatures)

 tinycity-mock: a standalone MCP server that mimics a game with tiered tools,
 limits and an audit log. A test target, not a dependency of agentgw.
```

## Status

| Milestone | Scope | State |
|---|---|---|
| M0 Foundations | Manifest, policy, signing, REST connector, audit, call pipeline, dev API | Done |
| M1 MCP transport | Official Go SDK, tools from the manifest, streamable HTTP, YAML manifest, conformance suite | Done |
| M2 Policy engine | Redis-backed policy, concurrency tests, confirmations in Postgres, kill switches | Done |
| M3 Auth | Token validation, authorization server, link tickets, consent, revocation | Exit criterion met; OIDC login at consent open |
| M4 Studio and player APIs | Manifest upload and rollback, key rotation, activity feed (grants and approvals are already there) | Next |
| M5 to M6 | Relay, hardening | Planned |

Details: [`agentgw/docs/ROADMAP.md`](https://github.com/Moustafa-Elgammal/agent-gateway-workspace-/blob/main/agentgw/docs/ROADMAP.md).

## Get started

Everything runs in containers. You only need Docker with Compose v2.20 or newer. Do not install
Go or Python on the host.

```bash
git clone --recurse-submodules <this repo>
# already cloned without submodules?
git submodule update --init --recursive

docker compose up --build                 # gateway (127.0.0.1:8080) + fake city backend
docker compose up -d mock                 # TinyCity MCP mock (127.0.0.1:8000/mcp)
docker compose run --rm mock-test         # mock end-to-end test, prints ALL CHECKS PASSED
docker compose down
```

Try a call through the gateway while it is up:

```bash
curl -s -X POST -H 'X-Dev-Player: p1' \
  -d '{"arguments":{"resource":"food","quantity":50}}' \
  http://127.0.0.1:8080/v1/tools/sell_resource/call
```

Run the Go tests, vet, format check, lint and vulnerability scan:

```bash
cd agentgw
docker compose run --rm dev make test
docker compose run --rm dev make vet fmt lint vuln
docker compose run --rm dev go test -race -count=1 -run TestName ./internal/policy/...
```

The `dev` and `mock-test` services sit behind the compose `tools` profile, so they start with
`run`, never with `up`. Published ports bind to 127.0.0.1 only. Dev secrets are placeholders.

## Where to read next

- [`agentgw/README.md`](https://github.com/Moustafa-Elgammal/agent-gateway-workspace-/blob/main/agentgw/README.md): what works today and the design rules
- [`agentgw/docs/ARCHITECTURE.md`](https://github.com/Moustafa-Elgammal/agent-gateway-workspace-/blob/main/agentgw/docs/ARCHITECTURE.md): the call pipeline
- [`agentgw/docs/MANIFEST.md`](https://github.com/Moustafa-Elgammal/agent-gateway-workspace-/blob/main/agentgw/docs/MANIFEST.md): describing a game's tools
- [`agentgw/docs/PROTOCOL.md`](https://github.com/Moustafa-Elgammal/agent-gateway-workspace-/blob/main/agentgw/docs/PROTOCOL.md): the signed request contract
- [`agentgw/docs/SECURITY-MODEL.md`](https://github.com/Moustafa-Elgammal/agent-gateway-workspace-/blob/main/agentgw/docs/SECURITY-MODEL.md): threats and controls
- [`agentgw/docs/DECISIONS.md`](https://github.com/Moustafa-Elgammal/agent-gateway-workspace-/blob/main/agentgw/docs/DECISIONS.md): decisions and open questions
- [`tinycity-mock/README.md`](https://github.com/Moustafa-Elgammal/agent-gateway-workspace-/blob/main/tinycity-mock/README.md): the test game and its tools

## Working in this repo

- Commit inside the project folder first, then commit the updated submodule pointer here.
- Every commit in `agentgw/` is signed off (`git commit -s`).
- This is an independent personal project. It contains no code, data or names from any employer
  or real game, and it is tested only against `tinycity-mock/` and `agentgw/examples/city/`.
- Guidance for Claude Code is in [`CLAUDE.md`](CLAUDE.md) and in each project's `CLAUDE.md`.

## License

Apache-2.0 for both projects. See each project's `LICENSE`.
