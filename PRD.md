# PRD: Bob's Cloudflare MCP

## 1. Summary

"Bob's Cloudflare MCP" is a Cloudflare Worker, backed by a single Durable Object, that serves two audiences from one deployed service:

1. **MCP clients** (Claude Desktop, MCP Inspector, other Model Context Protocol clients) connect over the MCP Streamable HTTP transport and can read Bob's Cloudflare Learnings as a Markdown resource and/or invoke a tool that returns the same content.
2. **Human browsers** that hit the service URL directly get back a rendered HTML landing page.

Both the Markdown and the HTML are content, not application state: they ship in the repo, get bundled into the Worker at build time, and are seeded into Durable Object storage on deploy. There is no runtime write path — the only way to change what's served is to edit the source files and redeploy.

## 2. Goals

- Provide a working, minimal MCP server that exposes "Cloudflare Learnings" content to any MCP-compatible client.
- Provide a human-friendly HTML view of the same service for anyone who opens the URL in a browser.
- Keep content management dead simple: one Markdown file, one HTML file, both versioned in git, both immutable at runtime.
- Run entirely on Cloudflare's free/standard primitives: Workers + Durable Objects. No external database, no KV, no auth provider.

## 3. Non-Goals

- No runtime content editing (no PUT/PATCH endpoints, no admin UI, no CMS).
- No authentication/authorization — the service is public and read-only.
- No multi-document support — exactly one Markdown resource and one HTML page.
- No analytics, rate limiting, or usage tracking (may be revisited later; not in scope now).
- No client SDK or published npm package — this is a standalone deployable service.

## 4. Users / Use Cases

| User | Use case |
|---|---|
| Bob, via Claude Desktop or another MCP client | Connects to the MCP server to pull up "Cloudflare Learnings" as reference material while working (as a resource, or by invoking a tool). |
| Anyone with the service URL, via a browser | Opens the URL and sees a rendered HTML page describing/presenting the service (e.g., "Bob's Cloudflare MCP" landing page, possibly showing the learnings content or linking to it). |

## 5. Architecture

### 5.1 Components

- **Worker (`src/index.ts`)** — thin entry point. Every request is routed to a single, fixed-name Durable Object instance (a singleton, e.g. `idFromName("singleton")`). The Worker itself holds no logic beyond routing.
- **Durable Object (`LearningsHub`, `src/learnings-hub.ts`)** — the whole service lives here:
  - Extends Cloudflare's `McpAgent` (from the `agents` package) to get MCP Streamable HTTP transport handling for free, since `McpAgent` is itself Durable-Object-backed.
  - Overrides `fetch()` to branch on path:
    - `/mcp` (and any MCP sub-paths the SDK requires) → delegate to `McpAgent`'s built-in MCP handling.
    - `/` (anything else, i.e. a plain browser GET) → read the stored HTML from Durable Object storage and return it directly.
  - Registers MCP capabilities in its `init()`/`server` setup:
    - **Resource**: `cloudflare-learnings` — `text/markdown`, returns the stored Markdown.
    - **Tool**: `get_cloudflare_learnings` — no input, returns the same Markdown as tool output.
  - On construction (or on first request), checks a stored content version against the build-time version constant; if they differ (including "not yet seeded"), overwrites storage with the bundled Markdown/HTML and updates the stored version.

### 5.2 Content pipeline

#### Existing content inventory

Real learnings content already exists in the repo under `content/`:

```
content/
├── cloudflare_worker.agent.md         # intro + hand-written TOC (links are currently stale, see below)
└── cloudflare/
    ├── 01-overview.md
    ├── 02-cloudflare-worker-api-guidelines.md
    ├── 03-cloudflare-d1-sqlite-guidelines.md
    ├── 04-durable-objects-guidelines.md
    ├── 05-kv-namespaces-document-store.md
    ├── 06-example-structure.md
    └── 07-workers-ai-image-generation-guidelines.md
```

The seven numbered files already follow a clean, self-imposed convention — numeric-prefix ordering, one topic per file, and same-directory cross-links between each other (e.g. `01-overview.md` links to `07-workers-ai-image-generation-guidelines.md`; `04-durable-objects-guidelines.md` and `05-kv-namespaces-document-store.md` link to each other). They also link out to external URLs (GitHub issues, Cloudflare docs) — those are untouched by merging, since only same-directory `.md` cross-links get rewritten.

**Known issue found in this content:** `cloudflare_worker.agent.md`'s hand-written TOC links point at `sections/cloudflare/01-overview.md` etc., but the files actually live at `cloudflare/01-overview.md` (no `sections/` folder exists). This is exactly the kind of drift a hand-maintained TOC is prone to, and exactly what the build validation step below is designed to catch instead of ship silently. See open questions for how to resolve it.

#### Design

- **Ordering:** derived from the existing convention already in use — the numeric filename prefix of each file directly under `content/cloudflare/` (`01-`, `02-`, … `07-`). No separate manifest file is needed; adding a new section is just adding a new numbered file.
- **Section title:** the first `#` H1 line of each file (e.g. `# Durable Objects Guidelines`). Used both as the section heading in the merged doc and to derive its anchor slug.
- **Cross-link convention (already in use):** links to other files in the same collection are plain relative filenames, optionally with a heading fragment: `[04](04-durable-objects-guidelines.md)` or `(some-file.md#some-heading)`.
- **`cloudflare_worker.agent.md`'s role:** its prose (role/objective framing) becomes the intro section of the merged document. Its hand-written numbered list is **not** carried over as-is — the build regenerates the table of contents from the actual files on disk, so it can never drift out of sync the way the current hand-written one has.
- **Build-time merge script (`scripts/build-content.ts`):** run before/during `wrangler deploy` (via a `build` step in `wrangler.toml` or an npm `predeploy` hook). It:
  1. Reads `content/cloudflare_worker.agent.md` for the intro prose, and every `content/cloudflare/NN-*.md` file in numeric-prefix order.
  2. Extracts each file's H1 as its title, derives a kebab-case anchor slug from it, and generates a table of contents at the top of the merged document linking to each section anchor.
  3. Rewrites every intra-collection link (`(some-file.md)` / `(some-file.md#fragment)`) found in any source file to point at the corresponding in-document anchor (`(#slug)` / `(#fragment)`) — links to anything outside `content/cloudflare/` (external URLs, etc.) are left untouched.
  4. Concatenates the intro plus all sections into a single Markdown string, written to `content/generated/cloudflare-learnings.md`.
  5. **Fails the build loudly** if a same-collection link points at a file that doesn't exist, or a fragment that doesn't resolve to a real heading — broken cross-links should never ship silently. (Run today, this would fail on the `sections/cloudflare/` paths above — resolving that is a prerequisite for the first successful build.)
- `content/index.html` is a separate, standalone file (the browser landing page) — not part of the multi-file merge; it doesn't exist yet, see the open question below on what it should contain.
- Wrangler module rules (or build-time `import ... from "..." with { type: "text" }` / esbuild raw-text loader) bundle the *generated* merged Markdown and `index.html` into the Worker as string constants at build time.
- A `CONTENT_VERSION` constant (bumped by hand, or derived from a hash of all source files) is bundled alongside them.
- The Durable Object seeds its storage from these constants whenever the stored version doesn't match — so a normal `wrangler deploy` after editing any learnings file or the HTML is sufficient to update what's served. No separate migration or admin step is needed for content changes.

### 5.3 Routing summary

| Path | Method | Handled by | Returns |
|---|---|---|---|
| `/` | GET | DO, direct storage read | `text/html` landing page |
| `/mcp` | GET/POST (Streamable HTTP) | DO, via `McpAgent` | MCP JSON-RPC traffic (initialize, resources/*, tools/*) |

No authentication is applied to either path — the service is public and read-only.

## 6. Data Model (Durable Object storage)

Single DO instance, key/value storage:

| Key | Type | Description |
|---|---|---|
| `content:markdown` | string | Full text of the **merged** Cloudflare Learnings document (generated from `content/learnings/*.md` + `manifest.json` at build time) |
| `content:html` | string | Full text of `index.html` |
| `content:version` | string | Version/hash used to decide whether to reseed on deploy |

The multi-file source layout and cross-link rewriting are purely a build-time/authoring concern (see §5.2) — the Durable Object only ever sees and serves the single, already-merged Markdown string. It has no knowledge of the original file boundaries.

## 7. Functional Requirements

1. A browser `GET` to the Worker's root URL returns `200` with `Content-Type: text/html` and the bundled HTML page body.
2. An MCP client can complete the Streamable HTTP `initialize` handshake against `/mcp`.
3. `resources/list` includes the Cloudflare Learnings resource; `resources/read` on it returns the full **merged** Markdown text (all learnings files combined per the manifest order) with `text/markdown` mime type.
4. `tools/list` includes `get_cloudflare_learnings`; calling it with no arguments returns the same merged Markdown text as tool result content.
5. Cross-links between learnings files (e.g. `workers.md` linking to `durable-objects.md`) resolve as working in-document anchor links in the merged output — no link points at a file that no longer exists once merged.
6. The build fails (deploy does not proceed) if any cross-link in the learnings files can't be resolved against the manifest, so broken links are caught before they ship.
7. Content changes only take effect after editing the source file(s)/manifest and redeploying — there is no runtime mutation path of any kind.
8. On first deploy (cold DO, empty storage), the DO self-seeds from bundled content without requiring a manual step.

## 8. Technical Requirements

- **Language:** TypeScript, strict mode (per global TypeScript conventions).
- **Runtime constraints:** Web Standard APIs only — no Node built-ins (`fs`, `path`, native `crypto`); no `process.env` (use the `env` object passed to `fetch`).
- **MCP SDK:** Cloudflare `agents` package (`McpAgent`) plus `@modelcontextprotocol/sdk` types as needed.
- **Durable Objects:** one class (`LearningsHub`), one singleton instance for the whole service. Needs a `new_sqlite_classes` (or `new_classes`) migration entry in `wrangler.toml`.
- **Content build step:** a standalone script (`scripts/build-content.ts`, run via Node — this runs at build time on the developer/CI machine, not in the Worker runtime, so Node APIs are fine here) that merges `content/learnings/*.md` per `manifest.json`, rewrites cross-links to anchors, validates all links resolve, and writes `content/generated/cloudflare-learnings.md`. Wired in as a `predeploy`/`build` npm script so it always runs before `wrangler deploy`/`wrangler dev`.
- **Bundling:** `wrangler.toml` module rules (or equivalent) to inline the *generated* merged `.md` file and `index.html` as text at build time.
- **CORS:** since MCP clients and browsers both hit this Worker, responses should include permissive CORS headers on the `/mcp` path so browser-based MCP clients aren't blocked; the `/` HTML path doesn't need CORS.

## 9. Verification Plan

Before considering any deploy "done":

1. `node scripts/build-content.ts` (or the wired-up `npm run build:content`) — confirm the merge succeeds, cross-links resolve, and `content/generated/cloudflare-learnings.md` looks correct (headings, anchors, TOC).
2. Deliberately break a cross-link in one of the `content/learnings/*.md` files and confirm the build step fails with a clear error, then revert.
3. `npx wrangler dev` — run locally.
4. Browser check: hit `http://localhost:8787/` and confirm the HTML page renders.
5. MCP check: use MCP Inspector (or equivalent) against `http://localhost:8787/mcp` to confirm `initialize`, `resources/list`, `resources/read`, `tools/list`, and `tools/call` all work and return the expected merged Markdown, with internal links intact.
6. Content-update check: edit one file under `content/learnings/`, bump `CONTENT_VERSION`, redeploy, and confirm the resource/tool output reflects the change (proving the reseed-on-version-mismatch logic works).
7. `wrangler deploy --dry-run` (or a real deploy to a preview environment) before shipping to production.

## 10. Open Questions

- **Stale TOC links in `content/cloudflare_worker.agent.md`:** its links point at `sections/cloudflare/*.md`, but the files live at `content/cloudflare/*.md` directly. Since the build regenerates the TOC from disk anyway (§5.2), the simplest fix is to just drop the hand-written numbered list from that file and keep its intro prose — confirm that's fine, or say if the numbered list (with its per-section one-line descriptions) should be preserved/repaired and fed into the generated TOC instead of discarded.
- What should the HTML landing page actually contain — just a description of the MCP service (name, how to connect), or should it also render the Markdown learnings content for human readers? *(Needs Bob's input before building the HTML; `content/index.html` doesn't exist yet.)*
- Desired Worker subdomain/custom domain, if any (affects `wrangler.toml` `routes`/`workers.dev` config).

## 11. Milestones

1. Scaffold project: `wrangler.toml`, `src/index.ts`, `src/learnings-hub.ts`, `content/learnings/` with placeholder files + `manifest.json`, `content/index.html`.
2. Implement `scripts/build-content.ts` (merge, link rewrite, validation) and wire it as a `predeploy`/`build` step.
3. Implement DO seeding logic + storage schema, reading the generated merged Markdown.
4. Implement MCP resource + tool registration.
5. Implement HTML landing page path.
6. Local verification via `wrangler dev` + MCP Inspector, including the broken-link build-failure case.
7. Deploy to Cloudflare, verify against the live URL.
