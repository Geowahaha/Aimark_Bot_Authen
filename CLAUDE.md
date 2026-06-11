# CLAUDE.md — Aimark_Bot_Authen

## What this repo is
The verified-identity layer of AIMarkBot: RFC 9421 (Web Bot Auth) Ed25519
request signing + the public key directory + the public /bot identity page
(Thai/English). Developed standalone here, then integrated into
`Marketing-visibility-engine` (Cloudflare Pages project `aimark`).

## Layout
- `web/functions/api/_botauth.js` — signing core; `signedFetch(env, url, init)` drop-in; FAIL-OPEN when key unset.
- `web/functions/.well-known/http-message-signatures-directory.js` — JWKS directory (self-signed response, rotation via `BOTAUTH_PUBLIC_JWK_PREV`).
- `web/bot.html` + `web/bot.md` — public bot identity page (TH/EN) + markdown twin.
- `web/_headers` — content-type for /bot.md + Link headers.
- `scripts/generate-botauth-key.mjs` — Ed25519 identity generator (kid = RFC 7638 thumbprint).
- `scripts/test-botauth.mjs` — e2e sign→verify test. MUST pass before any commit: `node scripts/test-botauth.mjs`.
- `BOTAUTH.md` — deploy manual (secrets, env vars, rotation, verification matrix).
- `CLAUDE-CODE-PLAN-botauth-callsites.md` — integration plan for the main engine repo (scan.js / deep-scan.js / bot-access.js).

## Iron rules (never violate)
1. Sign ONLY target-site fetches. Never sign LLM/PSI API calls or self-calls.
2. NEVER sign a request carrying another bot's User-Agent (bot-access simulations stay unsigned — identity fraud otherwise).
3. Private JWK is a Cloudflare encrypted secret only. Never in code, never committed, never logged.
4. Fail-open: missing key ⇒ plain fetch. An audit must never fail because of identity.

## Env contract
- `BOTAUTH_PRIVATE_JWK` (secret) · `BOTAUTH_PUBLIC_JWK` (var) · `BOTAUTH_AGENT_URL` (var, e.g. https://aimark.pages.dev) · optional `BOTAUTH_PUBLIC_JWK_PREV` (rotation overlap).

## Workflow
Small commits, conventional messages (`feat(botauth): …`). Run the e2e test +
`node --check` on touched files before every commit. Windows/PowerShell host.

## workerd JWK rule (proven on production — not a guess)
JWK สำหรับ workerd: ห้ามมี `alg` อื่นนอกจาก `"EdDSA"`; strip `alg`/`use`/`key_ops` ก่อน `importKey` เสมอ.

- `alg: "Ed25519"` → Node ยอมรับ, workerd โยน error ("alg does not match requested Ed25519 curve")
- `alg: "EdDSA"` → ถูกต้องตาม RFC 8037, workerd ยอมรับ
- Fix pattern in `getSigningKey`: `const { alg, use, key_ops, ...jwk } = JSON.parse(env.BOTAUTH_PRIVATE_JWK);`
- e2e test สร้าง key ไม่มี `alg` field จึงไม่จับบั๊กนี้ — จับได้บน workerd เท่านั้น
