# Developer Guide

## Initial Setup

1. **Install dependencies**

   ```sh
   npm install
   ```

2. **Copy environment example** (required for local dev; `.env` is gitignored)

   ```sh
   cp .env.example .env
   ```

   Edit `.env` and set:

   - `KSOR_DB_URL` — Postgres DSN environment variable (needs pgvector extension)
   - `GEMINI_API_KEY` — embedding provider key (free tier sufficient)
   - `KSOR_AUTH=disabled-local` — required for local `npm run serve` (binds loopback)

3. **Provision the database schema and grants** (run once)

   ```sh
   npm run provision
   ```

4. **Build and ingest the record**

   ```sh
   npm run refresh
   ```

5. **Start the local MCP server**

   ```sh
   npm run serve
   ```

   The server binds `http://localhost:3000` and requires authentication with `KSOR_AUTH=disabled-local`.

## Available Scripts

All scripts are defined in `package.json`. Run with `npm run <script>` from the repo root.

| Script | Description |
|--------|-------------|
| `dev` | Start the Next.js dev server at `http://localhost:3000` (hot-reloading) |
| `build` | Run `ksor build` then build the static site into `system/site/out/` |
| `preview` | Preview the built site via `node system/site/preview.mjs` |
| `check` | Run the format checker — validates all `knowledge/` documents |
| `provision` | Apply DDL and authorize ingest (schema + grant) — run once after setup |
| `serve` | Run the MCP server (`ksor serve`) |
| `refresh` | Full refresh: build, ingest, and garbage‑collect abandoned generations |
| `schema` | Apply the DDL against the DB — privileged, run once |
| `grant` | Authorize ingest for this corpus against the DB — run once |
| `ingest` | Embed `knowledge/` into a generation and activate it (flip active pointer) |
| `gc` | Garbage‑collect abandoned generations |

## ksor CLI verbs

These run via `npx ksor` (included with `@panaversity/ksor`):

| Verb | Description |
|------|-------------|
| `ksor build --instance <file>` | Regenerate indexes, lock, and build the site |
| `ksor schema --instance <file> --apply` | Apply DDL to the database |
| `ksor grant --instance <file>` | Authorize ingest for the corpus |
| `ksor ingest --instance <file> [--flip]` | Embed knowledge and activate generation |
| `ksor gc --instance <file>` | Collect abandoned generations |
| `ksor takedown --instance <file> …` | Withdraw a document (ledger + denylist) |
| `ksor calibrate --instance <file>` | Calibrate the abstention vector floor |
| `ksor serve` | Start the MCP server |

## Workflow after editing `knowledge/`

1. Run the format check:

   ```sh
   npm run check
   ```

2. Rebuild and refresh:

   ```sh
   npm run refresh
   ```

3. The MCP server automatically picks up the new generation (no restart needed unless you used `--flip`).

## Environment variables (`.env`)

| Variable | Required? | Description |
|----------|-----------|-------------|
| `KSOR_DB_URL` | Yes (for DB‑backed operations) | Postgres connection string holding the record; needs pgvector |
| `GEMINI_API_KEY` | Yes (for embedding) | Key for the Gemini embedding provider |
| `KSOR_AUTH` | Yes | Set to `disabled-local` for loopback dev; `disabled-public` for public bind; or configure SSO variables for production |
| `KSOR_SSO_URL` | Conditional | SSO authorization server discovery URL (public deployments) |
| `KSOR_MCP_RESOURCE_URL` | Conditional | The record's resource identifier for OAuth |
| `KSOR_JWKS_URL` | Optional | Override the JWKS document discovery (normally auto‑detected) |
| `KSOR_SNAPSHOT_KEYS` | Optional | Shared signing keys for multi‑replica deployments |
| `KSOR_AUTH=disabled-local` | Required locally | Prevents an open door on non‑loopback binds |
| `KSOR_ALLOWED_HOSTS` / `KSOR_ALLOWED_ORIGINS` | Optional | Restrict binds on non‑loopback |
| `KSOR_BASE_PATH` | Optional | Sub‑path hosting prefix (e.g. `/repo`) |

## Common task sequences

### First time / new clone

```sh
npm install                      # install deps
cp .env.example .env             # fill in KSOR_DB_URL, GEMINI_API_KEY, KSOR_AUTH=disabled-local
npm run provision                # schema + grant (once)
npm run refresh                  # build + ingest + gc
npm run serve                    # start the local MCP server
```

### After editing a document in `knowledge/`

```sh
npm run check                    # validate format
npm run refresh                  # rebuild ingest, flip generation if changed
# server picks up the active generation automatically
```

### Deploy (container / production)

```sh
npm run provision                # apply DDL + grant (once)
npm run ingest                   # embed & activate generation
# Then run the container with the env vars above,
# setting KSOR_AUTH to either disabled-public or the SSO variables.
# Do NOT embed .env in the image — bake the DSN separately if needed, or provision
# the DSN at runtime via the host environment.
```

## Record surfaces

| Surface | Command |
|---------|---------|
| Human web site (pages, sidebar, search) | `npm run build` then serve `system/site/out/` |
| MCP / agent surface (tool definitions, search, llms.txt) | `npm run serve` (live MCP on `http://localhost:3000/mcp`) |
| `llms.txt` / `llms-full.txt` | Generated by `npm run build`; snapshot of what the build admitted |

## Record governance

- Documents start `status: draft` — reach **no** machine surface until approved.
- Approve with `status: stable` + `ksor.approval: { by, at }` + `generated: { by, at }`.
- `npm run check` enforces frontmatter, filenames, links, and governance rules.
- `ksor build` regenerates folder `index.md` files and the build lock.
- `ksor takedown` writes to `.ksor/takedowns.yaml` (committed ledger) and the DB denylist.
- `ksor calibrate` measures the abstention floor; add `retrieval:` to `instance.md` to gate the server.