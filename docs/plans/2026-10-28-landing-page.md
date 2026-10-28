# Landing page and ChatGPT route Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Publish the landing page from `selfwise-labs/releases` through GitHub Pages, and let ChatGPT reach the MCP Worker with the token in the URL.

**Architecture:** Task 1 is a static `index.html` plus three SVG marks in the public repo, converted from the approved mockup file. Task 2 adds `POST /mcp/:token` to the Hono app of `selfwise-mcp`, sharing the existing `/mcp` handler. The two tasks touch different repositories and are independent.

**Tech Stack:** plain HTML/CSS (Google Fonts, no build); Hono on Cloudflare Workers, vitest, the in-process MCP client in `test/mcpClient.ts`.

## Global Constraints

- Design: `docs/2026-10-28-landing-page-design.md` in `selfwise-labs/releases`.
- The page shows no e-mail address and no contact form.
- MCP endpoint `https://selfwise-mcp.cristian-matei-toader.workers.dev/mcp`; the ChatGPT form is `https://selfwise-mcp.cristian-matei-toader.workers.dev/mcp/<token>`.
- Links on the page, exactly: TestFlight `https://testflight.apple.com/join/hzXY9tcd` (Selfwise), `https://testflight.apple.com/join/EQMKsmwa` (Sagax), `https://testflight.apple.com/join/mnCAbn61` (Sapor); APKs `https://github.com/selfwise-labs/releases/releases/download/<app>-latest/<app>.apk`; releases list `https://github.com/selfwise-labs/releases/releases`; repo `https://github.com/selfwise-labs/releases`.
- Commit messages end with exactly:

  ```
  Co-Authored-By: Claude Fable 5.1 <noreply@anthropic.com>
  Claude-Session: https://claude.ai/code/session_01JvRgBstZnu4BJJYoFY72Ci
  ```

- Stage files by explicit path. Use the Edit/Write tools for file content, never shell heredocs or Python for source files (the Windows Bash tool corrupts escapes). Keep source files ASCII except where the mockup already has `&middot;`-style entities (keep entities, not raw characters).
- One PR per repo, branch `feat/landing-page` (releases) and `feat/mcp-token-in-url` (selfwise-mcp), from an up-to-date `main`. Do not push and do not open PRs; the controller does.

---

### Task 1: The page in `selfwise-labs/releases`

**Files:**
- Create: `index.html`, `assets/selfwise.svg`, `assets/sagax.svg`, `assets/sapor.svg`, `.nojekyll`
- Modify: `README.md` (first paragraph: link to the page)
- Already on the branch (do not edit): `docs/2026-10-28-landing-page-design.md`, `docs/plans/2026-10-28-landing-page.md`

**Interfaces:** none; the page links only to existing URLs.

- [ ] **Step 1: Copy the marks**

Copy, byte for byte (Bash `cp`), into `assets/`:

| From | To |
| --- | --- |
| `C:\Users\crist\development\cmt-agentic\workspace\selfwise\selfwise-app\assets\brand\owl-glyph.svg` | `assets/selfwise.svg` |
| `C:\Users\crist\development\cmt-agentic\workspace\sagax\sagax-app\assets\brand\stare-glyph.svg` | `assets/sagax.svg` |
| `C:\Users\crist\development\cmt-agentic\workspace\sapor\sapor-app\assets\brand\bear-glyph.svg` | `assets/sapor.svg` |

Create an empty `.nojekyll` at the repo root.

- [ ] **Step 2: Convert the mockup to `index.html`**

Source: `C:\Users\crist\AppData\Local\Temp\claude\C--Users-crist-development-cmt-agentic\39c28473-80bc-45f3-a0f9-0b3e4af36ec0\scratchpad\landing\project\Main.dc.html` (read it whole). It is an artboard file; the page is the same markup as a plain HTML document:

1. Keep `<!doctype html>`, `<html lang="en">`, `<meta charset="utf-8">`, `<title>Selfwise Labs</title>`. Add `<meta name="viewport" content="width=device-width, initial-scale=1">` and `<meta name="description" content="Selfwise, Sagax and Sapor: a journal and planner, a training log and a food diary for iPhone and Android, with a connector for Claude and ChatGPT.">`. Drop the `<script src="./support.js">` line.
2. Move the `<link>` and `<style>` lines from inside `<helmet>` into `<head>`; drop the `<x-dc>`, `<helmet>` wrappers and the whole `<script type="text/x-dc" ...>` block at the end. The body is the single `<div style="max-width: 1120px; ...">` and its contents, unchanged.
3. Replace the three asset URLs everywhere: `/_blob/6d9fd0e73fd498c21f414bb6ed03ec85` becomes `assets/selfwise.svg`, `/_blob/66f1b3d7c0157f5b072d7a1543a0a1f7` becomes `assets/sagax.svg`, `/_blob/3f282a1b402b6e6db2e97bc6ac13f671` becomes `assets/sapor.svg`.
4. Phone width. Add `class="grid3"` to the apps grid `<div style="display: grid; grid-template-columns: repeat(3, minmax(0, 1fr)); ...">`, `class="grid2"` to the two-column grid of the Claude / ChatGPT cards and to the "Things to ask" grid, and append to the stylesheet:

```css
@media (max-width: 760px){
  .grid3,.grid2{grid-template-columns:1fr !important}
  h1{font-size:38px !important}
  h2{font-size:28px !important}
}
```

5. Make the `<nav>` links and the "Source of this page" link work as plain anchors (they already are). Keep `href="#top"` on the wordmark.

- [ ] **Step 3: Check the page**

From the repo root, render it headless and look at the screenshot (the Read tool shows PNGs):

```
"C:\Program Files (x86)\Microsoft\Edge\Application\msedge.exe" --headless=new --disable-gpu --hide-scrollbars --window-size=1300,2400 --screenshot=C:\Users\crist\AppData\Local\Temp\claude\C--Users-crist-development-cmt-agentic\39c28473-80bc-45f3-a0f9-0b3e4af36ec0\scratchpad\landing-desktop.png file:///C:/Users/crist/development/cmt-agentic/workspace/releases/index.html
"C:\Program Files (x86)\Microsoft\Edge\Application\msedge.exe" --headless=new --disable-gpu --hide-scrollbars --window-size=390,3600 --screenshot=C:\Users\crist\AppData\Local\Temp\claude\C--Users-crist-development-cmt-agentic\39c28473-80bc-45f3-a0f9-0b3e4af36ec0\scratchpad\landing-phone.png file:///C:/Users/crist/development/cmt-agentic/workspace/releases/index.html
```

Expected: the three marks render (if they are missing, the `assets/` paths are wrong); at 390 px the cards are one column and nothing overflows sideways. Also check: `grep -c "_blob" index.html` is 0; `grep -c "mailto\|outlook" index.html` is 0; `python -c "import html.parser"`-level sanity is not needed, but every `<details>` has a `</details>`.

- [ ] **Step 4: README**

In `README.md`, after the first paragraph, add one line: `The landing page: https://selfwise-labs.github.io/releases/ (this repository's \`index.html\`, served by GitHub Pages).`

- [ ] **Step 5: Commit**

```bash
git add index.html assets/selfwise.svg assets/sagax.svg assets/sapor.svg .nojekyll README.md
git commit -m "feat: the landing page, served by GitHub Pages

index.html with the three app marks: what the apps are, where to get them,
the beta invite note, and how to connect Claude (header) or ChatGPT (token
in the URL) to the MCP server. .nojekyll so Pages serves the files as they
are.

Co-Authored-By: Claude Fable 5.1 <noreply@anthropic.com>
Claude-Session: https://claude.ai/code/session_01JvRgBstZnu4BJJYoFY72Ci"
```

---

### Task 2: `POST /mcp/:token` in `selfwise-mcp`

**Files:**
- Modify: `src/app.ts` (the `/mcp` handler), `test/app.test.ts`, `test/mcpClient.ts`, `README.md` (line 5-6), `CLAUDE.md` ("Clients" section)
- Test: `test/app.test.ts`

**Interfaces:**
- Consumes: `createApp(options)` from `src/app.ts`; `connectClient(app, token?)` from `test/mcpClient.ts`.
- Produces: `connectClient(app, token?, { tokenInUrl?: boolean })`.

- [ ] **Step 1: Failing tests**

In `test/mcpClient.ts` give `connectClient` a third parameter and use it for the URL:

```ts
export async function connectClient(app: Hono, token?: string, options: { tokenInUrl?: boolean } = {}): Promise<Client> {
  const client = new Client({ name: 'test-client', version: '0.0.0' });
  const url = options.tokenInUrl && token !== undefined ? `http://mcp.test/mcp/${token}` : 'http://mcp.test/mcp';
  const transport = new StreamableHTTPClientTransport(new URL(url), {
    fetch: async (url, init) => app.request(url, init),
    requestInit: token === undefined || options.tokenInUrl ? {} : { headers: { Authorization: `Bearer ${token}` } },
  });
  await client.connect(transport);
  return client;
}
```

In `test/app.test.ts` add, inside `describe('app')`, using the same `createApp` options as the existing tests (and `fakeBackend` from `./fakeBackend` if `list_tables` needs a backend; read `test/mcp.test.ts` for how a client test is set up there and copy that setup):

```ts
  it('takes the token from the URL for clients that cannot send a header (ChatGPT)', async () => {
    // same app + backend setup as the first client test in test/mcp.test.ts
    const client = await connectClient(app, 'tok-1', { tokenInUrl: true });
    const tools = await client.listTools();
    expect(tools.tools.map((t) => t.name)).toContain('list_tables');
  });

  it('answers GET /mcp/<token> with 405 and /mcp/ with 404', async () => {
    const app = createApp({ backendUrl: 'http://backend.test', seed: EMPTY_SEED_TEXT, fetch: noFetch, store: memoryStore() });
    expect((await app.request('http://mcp.test/mcp/tok-1')).status).toBe(405);
    expect((await app.request('http://mcp.test/mcp/')).status).toBe(404);
  });
```

Run: `npm test -- test/app.test.ts`. Expected: the first new test fails (404 on `/mcp/tok-1` through the client), the second fails on the 405 expectation.

- [ ] **Step 2: The route**

In `src/app.ts`, register the handler for both paths and read the token from the header first, then the path:

```ts
  const mcp = async (c: Context) => {
    const auth = c.req.header('authorization') ?? '';
    const token = auth.startsWith('Bearer ') && auth.length > 7 ? auth.slice(7) : (c.req.param('token') ?? '');
    if (token === '') {
      return c.json({ error: 'missing bearer token; use the token from the app\'s Account screen' }, 401, { 'WWW-Authenticate': 'Bearer' });
    }
    if (c.req.method !== 'POST') {
      return c.json({ error: 'method not allowed; MCP messages are POSTed' }, 405, { Allow: 'POST' });
    }
    // ... the existing body from `let waitUntil` on, unchanged, using `token`
  };
  app.all('/mcp', mcp);
  app.all('/mcp/:token', mcp);
```

Import `Context` from `hono` (`import { Hono, type Context } from 'hono'`). Delete the old `const token = auth.slice(7);` line. Add a comment above the `/mcp/:token` registration: `// ChatGPT's custom connectors cannot send a header; the token travels in the path instead.`

Run: `npm test -- test/app.test.ts`, then `npm test` and `npm run typecheck`. Expected: all pass.

- [ ] **Step 3: Docs**

`README.md` lines 5-6 become:

```
Endpoint: `https://selfwise-mcp.cristian-matei-toader.workers.dev/mcp` (Streamable HTTP, header
`Authorization: Bearer <token>`), or `/mcp/<token>` for clients that cannot send a header (ChatGPT).
```

`CLAUDE.md`, "Clients" section, add a bullet after the Claude Code one:

```
- ChatGPT (web; custom MCP connectors take OAuth or no authentication, never a header): chatgpt.com/plugins → Add custom MCP server, URL
  `https://selfwise-mcp.cristian-matei-toader.workers.dev/mcp/<token>`, No authentication. The token is in the URL, so the URL is the secret; revoke it from the Account screen if it leaks. Codex: `[mcp_servers.selfwise]` with `url` and `bearer_token_env_var` in `~/.codex/config.toml`.
```

- [ ] **Step 4: Commit**

```bash
git add src/app.ts test/app.test.ts test/mcpClient.ts README.md CLAUDE.md
git commit -m "feat: accept the token in the URL (POST /mcp/<token>) for ChatGPT

ChatGPT's custom MCP connectors support OAuth or no authentication and
cannot send a request header, so the same handler also serves /mcp/:token
with the token read from the path; a header, when present, wins.

Co-Authored-By: Claude Fable 5.1 <noreply@anthropic.com>
Claude-Session: https://claude.ai/code/session_01JvRgBstZnu4BJJYoFY72Ci"
```

Leave the untracked `docs/2026-10-28-rate-limiting-design.md` alone: it is another session's file.
