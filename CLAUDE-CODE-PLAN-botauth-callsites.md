# Claude Code Session Plan — Wire Web Bot Auth into the audit fetch paths

**Repo:** Marketing-visibility-engine · **Branch:** `feat/web-bot-auth` (create from main)
**Scope:** `web/functions/api/{scan.js, deep-scan.js, bot-access.js}` + verify gate.
**Out of scope this session:** UA change to AIMarkBot/1.0, /bot page, other audit endpoints (render-check, social-visibility, competitor, citation-probe) — follow-up session.

## Mission (read first, keep in context)

Every fetch the scanner makes **to a target site** must go through `signedFetch(env, url, init)` from `web/functions/api/_botauth.js`, which adds RFC 9421 Web Bot Auth headers (Signature-Agent / Signature-Input / Signature). Fail-open: with no `BOTAUTH_PRIVATE_JWK` configured, `signedFetch` behaves exactly like `fetch` — so all edits are zero-risk to current production behavior.

**Two iron rules:**
1. **Sign only target-site fetches.** Never sign: Anthropic/Groq/Kimi API calls (`callClaude`/`_llm.js`), Google PSI (`fetchPSI`, scan.js ~L522), KV/D1 ops, or self-calls to our own origin (`bot-intel.js callLocal`). 
2. **Never sign a fetch that impersonates another bot.** In `bot-access.js`, the per-bot UA simulation fetches (GPTBot, ClaudeBot, PerplexityBot UAs…) MUST stay unsigned — signing while wearing another bot's UA is identity fraud, the opposite of this feature. Only AIMarkBot-identity fetches get signed.

## Phase 0 — Preconditions (verify, don't assume)

1. Confirm these files exist (from the aimark-web-bot-auth package; if missing, STOP and ask):
   - `web/functions/api/_botauth.js`
   - `web/functions/.well-known/http-message-signatures-directory.js`
   - `scripts/generate-botauth-key.mjs`, `scripts/test-botauth.mjs`
2. `node --check web/functions/api/_botauth.js` and `node scripts/test-botauth.mjs` → must pass before any edits.

## Phase 1 — scan.js

Current state: `tryFetchText(url, timeoutMs = 12000)` defined ~L181 with `fetch(url, {...})` at ~L186. Callers at ~L774–779 inside `onRequestPost`, which already has `env` (`const { request, env } = context;` ~L746).

Edits:
1. Add import at top with the existing imports:
   `import { signedFetch } from "./_botauth.js";`
2. Change signature to accept env as FIRST param (matches signedFetch convention):
   `async function tryFetchText(env, url, timeoutMs = 12000)`
   and inside, replace `await fetch(url, {` → `await signedFetch(env, url, {` (keep all existing options: headers/UA, redirect, signal, cf untouched).
3. Update ALL callers (grep `tryFetchText(` — expect 5 call sites ~L774–779):
   `tryFetchText(url)` → `tryFetchText(env, url)`, `tryFetchText(root + "/robots.txt")` → `tryFetchText(env, root + "/robots.txt")`, etc. The conditional one: `alternateHomeUrl ? tryFetchText(env, alternateHomeUrl, 9000) : Promise.resolve(null)`.
4. Do NOT touch `fetchPSI` (~L522) or `callClaude`.

## Phase 2 — deep-scan.js

Current state: `fetchText(url, timeoutMs = 10000)` ~L27, `fetch(url, {...})` ~L31. Handler is `export async function onRequestPost(context)` with `const { request } = context;` — **env is NOT destructured yet**. Callers: ~L102, L103, L104 (Promise.all home/robots/sitemap), ~L114 (sitemap from robots), ~L133 (per-page loop `i === 0 ? home : await fetchText(p)`).

Edits:
1. Add import: `import { signedFetch } from "./_botauth.js";`
2. Change handler destructure to `const { request, env } = context;`
3. `async function fetchText(env, url, timeoutMs = 10000)` and `fetch(url,` → `signedFetch(env, url,` inside.
4. Update all 5 caller sites to pass `env` first. Watch the loop site: `await fetchText(env, p)`.

## Phase 3 — bot-access.js (the careful one)

Current state: `fetchAs(url, ua, timeoutMs = 11000)` ~L44. Handler `const { request } = context;` — env NOT destructured. robots.txt is fetched ~L113 as `fetchAs(root + "/robots.txt", BOTS[0].ua, 8000)` — i.e., wearing the first bot's UA. Per-bot matrix at ~L117 `Promise.all(BOTS.map(...))`.

Edits:
1. Import `signedFetch` from `./_botauth.js`. Destructure `const { request, env } = context;`.
2. **Leave `fetchAs` and the per-bot Promise.all UNTOUCHED and UNSIGNED.** Add a comment above `fetchAs`: `// UNSIGNED by design: these simulate third-party bot UAs. Never add Web Bot Auth here (identity fraud). AIMarkBot-identity fetches use signedFetch.`
3. Replace the robots.txt fetch with an honest, signed AIMarkBot-identity fetch. Add alongside `fetchAs`:
   ```js
   const AIMARK_UA = "AIMarkBot/1.0 (+https://aimark.pages.dev/bot; site-owner-requested audit)";
   async function fetchAsSelf(env, url, timeoutMs = 11000) {
     const ctrl = new AbortController();
     const t = setTimeout(() => ctrl.abort(), timeoutMs);
     try {
       const r = await signedFetch(env, url, {
         headers: { "User-Agent": AIMARK_UA, "Accept": "text/html,*/*", "Accept-Language": "en,th;q=0.9" },
         redirect: "follow", signal: ctrl.signal, cf: { cacheTtl: 0 },
       });
       return { ok: r.ok, status: r.status, body: await r.text(), finalUrl: r.url, headers: Object.fromEntries(r.headers) };
     } catch (e) {
       return { ok: false, status: 0, body: "", error: String(e).slice(0, 160), finalUrl: url, headers: {} };
     } finally { clearTimeout(t); }
   }
   ```
   Then: `const robotsRes = await fetchAsSelf(env, `${root}/robots.txt`, 8000);` (return shape matches existing usage: `.ok`, `.body`).
4. Add one authoritative signed baseline row to the response: before the per-bot matrix, `const selfRes = await fetchAsSelf(env, url);` and include in the JSON response:
   `aimark_bot: { ua: AIMARK_UA, signed: true, http_status: selfRes.status, fetch: selfRes.ok ? "served" : "blocked_or_error" }`
   (additive field — the front-end ignores unknown keys; do NOT change existing `results`/`robots_only` shapes.)
5. Update the existing `honest_note` string: append `" Our own baseline fetch is made as AIMarkBot with an RFC 9421 Web Bot Auth signature (key directory: /.well-known/http-message-signatures-directory)."`
6. `bot-intel.js`: NO changes (its `callLocal` hits our own origin).

## Phase 4 — Verify gate + npm script

1. `web/package.json` scripts: add `"test:botauth": "node ../scripts/test-botauth.mjs"`.
2. `scripts/aimark-verify-production.mjs`: in the TESTS array (~L140, alongside the `syntax:` entries) add:
   ```js
   ["syntax: _botauth.js", () => run(process.execPath, ["--check", rel("web", "functions", "api", "_botauth.js")])],
   ["syntax: deep-scan.js", () => run(process.execPath, ["--check", rel("web", "functions", "api", "deep-scan.js")])],
   ["syntax: bot-access.js", () => run(process.execPath, ["--check", rel("web", "functions", "api", "bot-access.js")])],
   ["botauth e2e", () => run(process.execPath, [rel("scripts", "test-botauth.mjs")])],
   ```

## Phase 5 — Verification (all must pass before commit)

```powershell
node --check web/functions/api/scan.js
node --check web/functions/api/deep-scan.js
node --check web/functions/api/bot-access.js
node scripts/test-botauth.mjs
# No stale callers left (each must return ZERO lines):
#   scan.js: tryFetchText( not followed by env
#   deep-scan.js: fetchText( not followed by env  (definition line excluded)
grep -n "tryFetchText([^e]" web/functions/api/scan.js
grep -n "fetchText([^e]" web/functions/api/deep-scan.js
# Iron-rule audit: fetchAs( must NOT reference signedFetch anywhere
grep -n "signedFetch" web/functions/api/bot-access.js   # expect: import line + fetchAsSelf only
npm run verify:local   # full gate from web/
# Local smoke (wrangler pages dev): POST /api/scan and /api/deep-scan with a test URL
#   → responses unchanged in shape; POST /api/bot-access → has aimark_bot field.
# Live directory check after deploy:
curl -sI https://aimark.pages.dev/.well-known/http-message-signatures-directory
```

## Acceptance criteria
- [ ] All target-site fetches in the 3 files route through signedFetch with env threaded.
- [ ] Per-bot simulation fetches remain plain fetch, with the iron-rule comment in place.
- [ ] bot-access response gains `aimark_bot` baseline; existing fields byte-compatible.
- [ ] With BOTAUTH secrets unset, behavior identical to main (fail-open verified by smoke test).
- [ ] verify:local fully green including the 4 new gate entries.

## Commit plan (3 commits)
1. `feat(botauth): thread env + signedFetch through scan.js and deep-scan.js`
2. `feat(botauth): signed AIMarkBot baseline in bot-access.js; keep bot simulations unsigned by design`
3. `chore(verify): add botauth e2e + syntax checks to gate`
Then PR `feat/web-bot-auth` → main; deploy; set secrets per BOTAUTH.md; re-run live checks.

## Rollback
Single-purpose branch; revert the merge commit. With secrets unset there is no behavioral change, so rollback urgency is low even post-merge.
