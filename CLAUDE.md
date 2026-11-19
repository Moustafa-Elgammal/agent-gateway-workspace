# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

# Workspace: agent gateway for games

Two independent projects live here. Each is its own git repository.

| Folder | What it is | Language | License |
|---|---|---|---|
| `agentgw/` | Open-source gateway that lets a player's AI agent act safely in a game (MCP tools from a manifest, policy, signed backend calls, audit) | Go | Apache-2.0 |
| `tinycity-mock/` | Fictional city-builder MCP server used only as a test environment | Python | Apache-2.0 (private repo) |

Start in the project folder you are working on and read its `CLAUDE.md`.

## How the two projects relate

- `agentgw` is the product. Its call pipeline (rate limit, validate, reserve, signed backend call,
  settle, audit) lives in `agentgw/internal/gateway`. The MCP adapter (`internal/mcpserver`) is
  M1 work and does not exist yet; today the gateway is exercised through its dev HTTP API.
- `tinycity-mock` is a standalone MCP server (`server.py`, one file) that mimics a game exposing
  tiered tools, rate limits, daily caps and an audit log. It is a test target, not a dependency
  of `agentgw`.
- The mock serves a browser dashboard (city map, resources, charts, audit) at `/` on both its ports
  (8000 MCP, 8001 REST). On the MCP port, agent actions wait for a click on that page
  (`TINYCITY_APPROVAL=1`). Through the gateway, `build_structure` is held by the gateway's own
  confirmation instead. Details in `tinycity-mock/README.md`.
- `agentgw/examples/city/` is a separate fake REST backend that verifies request signatures. It
  is what `docker compose up` runs behind the gateway.
- Full design lives in `agentgw/docs/` (`ARCHITECTURE.md`, `PROTOCOL.md`, `SECURITY-MODEL.md`,
  `DECISIONS.md`, `ROADMAP.md`).

## Docker always

Run every toolchain, service and test in containers. Do not install Go, Python packages,
linters or databases on the host. If something needs an image, write the Dockerfile target or
compose service for it and prepare it; do not ask the user to install it.

From this folder (needs Docker with Compose v2.20+):

```bash
docker compose up --build              # gateway + fake city backend (127.0.0.1:8080)
docker compose up -d mock              # TinyCity MCP mock (127.0.0.1:8000/mcp)
docker compose run --rm mock-test      # mock end-to-end test
docker compose up -d --build gateway-tinycity   # gateway on 127.0.0.1:8081 in front of the REST mock
docker compose build bridge            # image agentgw-bridge:latest for Claude Desktop (stdio to gateway)
docker compose down
```

Per-project commands (tests, lint, vuln) are in each project's `CLAUDE.md`
(`cd agentgw && docker compose run --rm dev make test`).

Run one Go test or package (from `agentgw/`):

```bash
docker compose run --rm dev go test -race -count=1 -run TestName ./internal/policy/...
```

The `dev` and `mock-test` services sit behind the compose `tools` profile. They are started with
`run`, never by `up`. The mock has no per-test selector: `test_client.py` is a single end-to-end
script that must print `ALL CHECKS PASSED`.

The workspace root is a git repo whose only content is the two projects as submodules
(`.gitmodules`), plus `CLAUDE.md`, `.gitignore` and `docker-compose.yml`. `agentgw/` and
`tinycity-mock/` each keep their own history, so commit inside the project folder first, then
commit the updated submodule pointer at the root. The root `docker-compose.yml` only
`include`s the two project compose files, so service names (`dev`, `mock`, `mock-test`) are
the same from either level.
Published ports bind to 127.0.0.1 only. Dev secrets are placeholders, never real credentials.

## Independence rules (apply everywhere)

- This is a personal, independent project. Do not add code, data, API details, names,
  screenshots or documents that belong to any employer or to any real game.
- Test only against `tinycity-mock/` or the fake backend in `agentgw/examples/city/`.
- Do not mention real games or companies in code, docs, commit messages or issues.
- Every commit in `agentgw/` is signed off (`git commit -s`) under the personal identity.

## Working style

- Read the relevant doc in `agentgw/docs/` before changing behavior.
- Run the project's tests (in Docker) before saying a change works.
- Keep changes small and focused. Ask before adding a dependency.
