---
name: graphify-docker
description: >-
  Run graphify (the knowledge-graph tool for codebases, docs, and media) and
  its MCP server through the phillarmonic/graphify-docker image. Use whenever
  the task involves graphify — building or updating a graph, querying,
  explaining, extracting paths, serving a graph over MCP, or configuring an
  MCP client to reach a graphify server — and prefer the Docker image over a
  local graphify installation for every operation.
---

# Graphify via Docker

For anything graphify-related, use the **`phillarmonic/graphify-docker`**
image. It ships the full graphify CLI (`graphifyy[all]`) and the `graphify-mcp`
server in one image, published for `linux/amd64` and `linux/arm64`. Do not
install graphify locally or suggest `uv tool install graphifyy` unless Docker
is genuinely unavailable (confirm with `docker info` before falling back).

Upstream docs: <https://phillarmonic.github.io/graphify-docker/>

## Image conventions

- The entrypoint is `docker-entrypoint.sh`. A bare `docker run <image>` (or
  args that are not `graphify`, `graphify-mcp`, `sh`, `bash`, `python`, `uv`)
  goes to `graphify-mcp`. Prefix with `graphify` to reach the CLI.
- `GRAPHIFY_OUT=/data` is set in the image; `/data` is the volume the MCP
  server serves. Mount the project's `graphify-out/` there.
- Runs as a non-root user (uid 10001); port `8080` exposed; default CMD serves
  `/data/graph.json` over Streamable HTTP on `0.0.0.0:8080`.
- Secrets are read from the environment at runtime — pass them with `-e`,
  never bake them in.

## Build or update a graph

Mount the project at `/work` and set the working directory:

```bash
docker run --rm -v "$(pwd):/work" -w /work \
  phillarmonic/graphify-docker graphify update .
```

Any `graphify` subcommand works the same way — `extract`, `query`, `path`,
`explain`, `add <url>`, `reflect`, `export callflow-html`, `prs`, and the
`--cluster-only` / `--wiki` / `--obsidian` / `--neo4j` style flags:

```bash
docker run --rm -v "$(pwd):/work" -w /work \
  phillarmonic/graphify-docker graphify query "what connects auth to the database?"
```

LLM-backed features (extraction, triage, `--mode deep`) need provider keys
passed through:

```bash
docker run --rm -v "$(pwd):/work" -w /work \
  -e OPENAI_API_KEY -e OPENAI_BASE_URL -e OPENAI_MODEL \
  phillarmonic/graphify-docker graphify update .
```

Other supported backends follow the same pattern: `ANTHROPIC_API_KEY`,
`GEMINI_API_KEY`/`GOOGLE_API_KEY`, `DEEPSEEK_API_KEY`, `MOONSHOT_API_KEY`,
Azure OpenAI (`AZURE_OPENAI_ENDPOINT`, ...), Ollama (`OLLAMA_HOST`,
`OLLAMA_MODEL`), and Bedrock (`AWS_ACCESS_KEY_ID`, `AWS_REGION`). See the
environment-variables doc for the full matrix:
<https://phillarmonic.github.io/graphify-docker/environment-variables/>

To build straight into the volume the MCP server serves, also mount `/data`:

```bash
docker run --rm -v "$(pwd):/work" -w /work -v graphify-data:/data \
  phillarmonic/graphify-docker graphify update .
```

## Serve a graph over MCP

The image's default CMD runs `graphify-mcp` over Streamable HTTP:

```bash
docker run -d --name graphify-mcp \
  -p 8080:8080 \
  -v "$(pwd)/graphify-out:/data" \
  -e GRAPHIFY_API_KEY="$SECRET" \
  phillarmonic/graphify-docker
```

The server listens at `http://localhost:8080/mcp`. **Always set
`GRAPHIFY_API_KEY` when binding beyond localhost** — without it the graph is
served unauthenticated. Clients authenticate with either
`Authorization: Bearer <key>` or `X-API-Key: <key>`.

Custom flags go after the image name (they pass straight to `graphify-mcp`):

```bash
docker run --rm -p 9000:9000 -v "$(pwd)/graphify-out:/data" \
  phillarmonic/graphify-docker \
  /data/graph.json --transport http --host 0.0.0.0 --port 9000 \
  --json-response --stateless
```

Key flags: `--transport stdio|http`, `--host`/`--port`, `--api-key`,
`--path` (default `/mcp`), `--json-response`, `--stateless` (for
load-balanced or CI use), `--session-timeout`.

A `docker-compose.yml` variant runs the same server; mount each project's
`graphify-out` under its own path and raise `GRAPHIFY_MAX_CONTEXTS` to serve
multiple projects from one instance.

## Configure an MCP client

When the user asks to point a client at a graphify MCP server, retrieve the
verified template through Repertoire:

```bash
repertoire stub list graphify-docker
repertoire stub get graphify-docker/claude-code-mcp-http
repertoire stub get graphify-docker/mcp-http
repertoire stub get graphify-docker/mcp-stdio
```

Follow the returned instructions and merge the asset into the client's
existing configuration. Preserve every existing server entry; replace the URL,
header, and graph-path placeholders; never print or commit a real API key.
MCP configuration loads at session start — if the tools don't appear, ask
the user to restart or open a new session rather than retrying.

Choose the template by transport:

- **HTTP** (preferred, the image default): client points at
  `http://localhost:8080/mcp` of a running container. One server can be shared
  by the whole team.
- **stdio**: client spawns `docker run --rm -i -v
  /absolute/path/to/graphify-out:/data phillarmonic/graphify-docker
  /data/graph.json --transport stdio`. Needs `-i` and an absolute mount path;
  no `-p`.

## Docker Compose

The graphify-docker repo ships a ready-made compose service
(`graphify-mcp` on `:8080`, `./graphify-out` mounted at `/data`,
`GRAPHIFY_API_KEY` and LLM keys passed through from the host):

```bash
GRAPHIFY_API_KEY=change-me docker compose up -d
```

For one-off CLI jobs against the same volume:

```bash
docker compose run --rm --entrypoint graphify \
  -v "$(pwd):/work" -w /work \
  graphify-mcp build .
```

## Fallback to a local install

Only if Docker is unavailable: `uv tool install "graphifyy[all]"`, then use
the `graphify` CLI and `python -m graphify.serve` directly (the same
entrypoints the image wraps). Tell the user you fell back and why.
