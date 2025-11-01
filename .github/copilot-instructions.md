## Quick goal for AI coding agents

Help contributors rapidly understand, modify, and extend the Chrome DevTools MCP
server. Prioritize code that touches the MCP tool surface (src/tools/), build/test
scripts, and the CLI/server lifecycle.

## Big picture (what to read first)
- Entry points: `src/index.ts` (CLI shim), `src/main.ts` (server bootstrap).
- CLI and config parsing: `src/cli.ts` (defines `cliOptions` and defaults).
- Tool definitions: `src/tools/*.ts` and `src/tools/ToolDefinition.ts` — each
  exported tool object provides `name`, `schema`, `annotations`, and `handler`.
- Runtime context: `src/McpContext.ts` manages browser/page state, timeouts,
  network/console collectors and is used by most tool handlers.

Read those four locations together to understand how a tool request flows from
the MCP transport -> server.registerTool handler -> McpContext -> tool.handler.

## Build / test / doc workflows (concrete commands)
- Build (TypeScript -> `build/`):
  - `npm run build` (runs `tsc`, then `scripts/post-build.ts` to create stubs and copy licenses)
  - `npm run bundle` cleans, builds and runs `rollup` (produces the packaged bundle).
- Run server locally (after build):
  - `npm run start` — builds and launches `node build/src/index.js`.
  - Use `--headless`, `--channel`, `--wsEndpoint`, `--browserUrl` etc. (see `src/cli.ts`).
- Tests:
  - `npm test` — builds, then runs Node tests with `--require ./build/tests/setup.js`.
  - If you need to run tests without rebuilding: `npm run test:only:no-build`.
  - Snapshot handling: `tests/setup.ts` overrides snapshot paths so snapshots live under `tests/`.
- Docs (tool reference):
  - `npm run docs` runs build, `scripts/generate-docs.ts`, and updates `README.md`.
  - `scripts/generate-docs.ts` launches the built MCP server (via stdio) to list tools — building is required.

Notes: package.json requires Node >= 20.19 (see `engines`). The build uses
`node --experimental-strip-types` in some scripts — keep that when running the
authors' workflow.

## Project-specific conventions & patterns
- Tools are auto-registered at server startup by importing `src/tools/*` in
  `src/main.ts` and calling `server.registerTool(...)`. To add a new tool, add
  an exported ToolDefinition in `src/tools/` and export it from the barrel file
  if present.
- Many README sections are auto-generated between markers:
  `<!-- BEGIN AUTO GENERATED TOOLS -->` / `<!-- END AUTO GENERATED TOOLS -->` and
  `<!-- BEGIN AUTO GENERATED OPTIONS -->` / `<!-- END AUTO GENERATED OPTIONS -->`.
  `scripts/generate-docs.ts` is responsible for updating them — it starts the
  built server and calls `client.listTools()`.
- Devtools frontend integration: the repo includes `node_modules/chrome-devtools-frontend` entries in tsconfig `include`; post-build scripts create
  small runtime stubs (see `scripts/post-build.ts`). Be cautious when editing
  those integration points.
- Logging & debugging: use the `DEBUG` env var or CLI `--logFile` to capture
  logs. The server also prints a startup disclaimer (sensitive data exposure).

## Integration points & external dependencies
- Puppeteer (automation) and chrome-devtools-frontend (DevTools models) are
  heavily used — see `package.json` devDependencies and `tsconfig.json` includes.
- The MCP transport is stdio by default: the server uses `StdioServerTransport`
  and the docs generator uses a Stdio client to spawn `node build/src/index.js`.
  This means many developer flows require a built artifact.

## Troubleshooting tips for agents
- If documentation generation or tests fail, ensure you ran `npm run build` and
  that Chrome is accessible (some workflows open Chrome or connect via ws).
- When editing tool schemas, update `scripts/generate-docs.ts` expectations if
  schema shapes change — the docs generator consumes `cliOptions` and tool
  `inputSchema` to produce the README sections.
- Tests and snapshots: tests rely on `tests/setup.ts` to map snapshot paths to
  `tests/*.snapshot` files; do not assume snapshots are under `build/`.

## Quick file references (examples)
- `src/main.ts` — server orchestration, tool registration, transport setup.
- `src/cli.ts` — yargs options and argument parsing (useful defaults and validators).
- `src/McpContext.ts` — page/browser lifecycle, timeouts, collectors.
- `src/tools/*.ts` — tool definitions and handlers (where new behavior is added).
- `scripts/generate-docs.ts` — generates `docs/tool-reference.md` and updates the README.
- `scripts/post-build.ts` — post-build stubs and license copying; called by `npm run build`.

## Safety / privacy
The server exposes browser and DevTools content to MCP clients. Avoid exposing
secrets or personal info in examples and test data.

If anything here is unclear or you'd like more examples (for example: a
step-by-step on adding a specific tool or running tests under Windows), tell me
which area to expand and I'll update this file.
