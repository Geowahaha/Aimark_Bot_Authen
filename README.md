# AIMarkBot — Web Bot Auth (RFC 9421)

AIMarkBot is AI Mark's website audit bot, implementing the Web Bot Auth profile of HTTP Message Signatures (RFC 9421) with Ed25519: every fetch to a target site carries cryptographically verifiable headers that any origin or WAF can authenticate against our published key directory. This repo is the verified-identity layer — signing core, key directory endpoint, and the public `/bot` identity page (Thai/English) — deployed as Cloudflare Pages Functions.

## Repository layout

| Path | Purpose |
|---|---|
| `web/functions/api/_botauth.js` | Signing core: `signedFetch(env, url, init)` drop-in + `botAuthHeaders` + `signDirectoryResponse`. Fail-open when key unset. |
| `web/functions/.well-known/http-message-signatures-directory.js` | JWKS directory endpoint (`GET /.well-known/http-message-signatures-directory`). Self-signs its own response. Supports key rotation via `BOTAUTH_PUBLIC_JWK_PREV`. |
| `web/bot.html` | Public bot identity page (Thai/English toggle). Explains what AIMarkBot is, how to verify its signatures, crawl behavior, and opt-out. |
| `web/bot.md` | Markdown twin of `/bot` (served with `Content-Type: text/markdown`). |
| `web/_headers` | Cloudflare Pages `_headers`: correct `Content-Type` for `/bot.md` and `Link` headers on `/bot`. |
| `scripts/generate-botauth-key.mjs` | Ed25519 key-pair generator. Prints `BOTAUTH_PRIVATE_JWK` and `BOTAUTH_PUBLIC_JWK` (kid = RFC 7638 thumbprint). Run locally; never commit the output. |
| `scripts/test-botauth.mjs` | End-to-end sign → verify test (no network). Must pass before every commit. |
| `BOTAUTH.md` | Full deploy manual: secrets, env vars, rotation procedure, verification matrix. |
| `CLAUDE-CODE-PLAN-botauth-callsites.md` | Integration plan for wiring `signedFetch` into the Marketing-visibility-engine repo (scan.js / deep-scan.js / bot-access.js). |
| `CLAUDE.md` | Codebase instructions and iron rules for Claude Code. |

## Quickstart

```sh
# 1. Generate the Ed25519 identity (run locally — output stays local, NEVER commit)
node scripts/generate-botauth-key.mjs

# 2. Store the private key as an encrypted Cloudflare secret
npx wrangler pages secret put BOTAUTH_PRIVATE_JWK --project-name aimark
# (paste the BOTAUTH_PRIVATE_JWK JSON when prompted)

# 3. Set plain env vars in Cloudflare Dashboard → aimark → Settings → Variables:
#    BOTAUTH_PUBLIC_JWK  = { ...public JWK from step 1 }
#    BOTAUTH_AGENT_URL   = https://aimark.pages.dev

# 4. Deploy (or let Cloudflare Pages auto-deploy on push to main)
```

Full secrets, rotation, and integration details: see **BOTAUTH.md**.

## Verification commands

```sh
# Syntax check
node --check web/functions/api/_botauth.js
node --check web/functions/.well-known/http-message-signatures-directory.js
node --check scripts/generate-botauth-key.mjs
node --check scripts/test-botauth.mjs

# End-to-end sign → verify (must pass before every commit)
node scripts/test-botauth.mjs

# Live directory check after deploy
curl -sI https://aimark.pages.dev/.well-known/http-message-signatures-directory
```

## Iron rules

1. Sign **only** target-site fetches. Never sign LLM/PSI API calls or self-calls.
2. **Never** sign a request carrying another bot's User-Agent (bot-access simulations stay unsigned).
3. `BOTAUTH_PRIVATE_JWK` is a Cloudflare encrypted secret only — never in code, never committed, never logged.
4. Fail-open: missing key → plain fetch. An audit must never fail because of identity.

## Integration into Marketing-visibility-engine

Wiring `signedFetch` into the main engine repo (scan.js, deep-scan.js, bot-access.js) follows the step-by-step plan in **CLAUDE-CODE-PLAN-botauth-callsites.md**.
