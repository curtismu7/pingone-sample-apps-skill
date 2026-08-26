---
name: pingone-sample-apps
description: "Build, run, and troubleshoot the PingOne devdocs sample apps (custom-admin-role, m2m-credentials, mfa-demo, user-registration × go/js/python/react/angular). Use when picking a sample, running setup.sh, wiring worker-app roles, hitting 401/403/406/415 from PingOne, or explaining the response_mode=pi.flow native-flow architecture. Trigger on 'devdocs-sample-apps', 'PingOne sample app', 'pi.flow', 'custom-admin-role', 'm2m-credentials', 'mfa-demo', 'user-registration'."
---

# PingOne devdocs sample apps

4 use cases × 5 stacks = 20 apps. Repo: `ping-rocks/devdocs-sample-apps` (Ping Identity EMU — a personal GitHub account cannot see it; you need an EMU account with SSO authorized).

Everything below was verified by running all 20 apps, not by reading the docs. Where this file and the repo's `AGENTS.md` disagree, this file is right and the reason is given.

## 1. Pick the sample

| Need | Use case | Can an agent finish it unattended? |
|---|---|---|
| Service-to-service OAuth, no user | `m2m-credentials` | yes |
| Management API + delegated admin | `custom-admin-role` | yes |
| Self-service signup + email verify | `user-registration` | **no** — needs a real inbox |
| Sign-in with email MFA | `mfa-demo` | **no** — needs a real inbox |

Stacks: `go`, `js`, `python` are flat. `react` and `angular` split into `client/` + `server/` — **run `setup.sh` from the stack root**, never from inside `client/` or `server/`.

Start with `m2m-credentials` if you just want to see the platform work: it is the only one that needs no inbox, creates nothing, and demonstrates full token validation.

## 2. Roles — the repo's AGENTS.md is wrong here

`AGENTS.md` contradicts the per-project READMEs in two places. **The READMEs are correct.**

| Use case | Correct (README + code) | `AGENTS.md` says |
|---|---|---|
| `custom-admin-role` | `Environment Admin` | `Organization Admin` — **over-privileged, do not grant** |
| `m2m-credentials` | `Identity Data Read Only` + `Environment Admin` | `Identity Data Read Only` + `PingOne Protect` — **no such role exists** |
| `mfa-demo` | `Identity Data Admin` + `Environment Admin` | (agrees) |
| `user-registration` | `Identity Data Admin` + `Environment Admin` | (agrees) |

The `custom-admin-role` case matters most because granting `Organization Admin` **succeeds** — nothing errors, the tenant is just over-privileged org-wide. The sample was deliberately built to avoid needing it; `custom-admin-role/js/index.js` hardcodes the Environment Admin role ID with this comment:

> Referencing it directly in `canBeAssignedBy` avoids a separate `GET /roles` lookup — which an Environment Admin worker cannot perform anyway, since listing platform roles requires Organization Admin.

Never grant `Organization Admin` for these samples.

## 3. Bootstrap, once per PingOne environment

Two different environments are involved, and mixing them is the most common setup failure:

- **Administrators environment** — used *only* to mint the bootstrap token in step 2.
- **Sandbox environment** — where the sample worker app and all sample resources live. This is both `PINGONE_ENV_ID` and `PINGONE_WORKER_ENV_ID` (normally the same value).

```bash
# 1. region — .ca / .eu / .asia / .com.au for other regions
export PINGONE_AUTH_PATH="https://auth.pingone.com"
export PINGONE_API_PATH="https://api.pingone.com"     # NO trailing /v1 — samples append it
export adminEnvId=... adminClientId=... adminClientSecret=... envId=...

# 2. bootstrap token from the ADMIN environment (expires in 1 hour)
curl -sS -X POST "$PINGONE_AUTH_PATH/$adminEnvId/as/token" \
  -H "Authorization: Basic $(printf '%s' "$adminClientId:$adminClientSecret" | base64 | tr -d '\r\n')" \
  --data-urlencode "grant_type=client_credentials"
export accessToken="<access_token from response>"

# 3. create the sample worker app in the SANDBOX environment
curl -sS -X POST "${PINGONE_API_PATH%/}/v1/environments/$envId/applications" \
  -H "Authorization: Bearer $accessToken" -H "Content-Type: application/json" \
  -d '{"name":"Sample Worker App","enabled":true,"type":"WORKER",
       "protocol":"OPENID_CONNECT","grantTypes":["CLIENT_CREDENTIALS"],
       "tokenEndpointAuthMethod":"CLIENT_SECRET_BASIC"}'
export appId="<id from response>"

# 4. read its secret
curl -sS "${PINGONE_API_PATH%/}/v1/environments/$envId/applications/$appId/secret" \
  -H "Authorization: Bearer $accessToken"

# 5. assign the roles from the table above, scoped to the sandbox env
curl -sS "${PINGONE_API_PATH%/}/v1/roles" -H "Authorization: Bearer $accessToken" \
  | jq '[._embedded.roles[] | select(.name=="Environment Admin") | {name,id}]'
curl -sS -X POST "${PINGONE_API_PATH%/}/v1/environments/$envId/applications/$appId/roleAssignments" \
  -H "Authorization: Bearer $accessToken" -H "Content-Type: application/json" \
  -d "{\"role\":{\"id\":\"<role id>\"},\"scope\":{\"id\":\"$envId\",\"type\":\"ENVIRONMENT\"}}"
```

Then fill the **root** `.env` once — every `setup.sh` reads it, so you set it once for all 20 apps:

```bash
cp .env.example .env && chmod 600 .env
```

`setup.sh` parses that file as literal data and never executes shell code: only `PINGONE_*` assignments, no expansion, quotes required around values containing spaces.

**Trap:** an already-exported variable beats the file — *including an exported empty one*, which silently shadows a correct file value. If a value seems ignored, run `env | grep PINGONE_` first.

## 4. Run

```
go       cd <uc>/go       && go run .                                    # :3000
js       cd <uc>/js       && npm install && npm start                    # :3000
python   cd <uc>/python   && pip install -r requirements.txt && python app.py   # :3000
react    server: cd <uc>/react/server   && npm install && npm start      # :3000
         client: cd <uc>/react/client   && npm install && npm run dev    # :5173
angular  server: cd <uc>/angular/server && npm install && npm start      # :3000
         client: cd <uc>/angular/client && npm install && npm start      # :4200
```

Requires `jq` and `curl` (every `setup.sh` uses them), plus the toolchain for your stack. Go apps declare `go 1.25.6`; Node stacks state 18+ and run clean on 26.

**Port 3000 is hardcoded in all 20 apps** — there is no `PORT` variable. Free the port or edit the source. All apps bind `0.0.0.0`, not localhost (`AGENTS.md` says otherwise and is wrong), so they are reachable from your local network while holding a worker client secret. Don't run them on untrusted networks.

**Production-mode serving is inconsistent across React.** Only `mfa-demo/react` and `user-registration/react` serve the built bundle with an SPA fallback. `m2m-credentials/react` serves static but has no fallback, so deep links 404. `custom-admin-role/react` serves **nothing** — `:3000` returns 404 by design and you must use the Vite dev server. All 4 Angular servers do it correctly.

## 5. The pi.flow architecture — explain this before anyone copies it

`user-registration` and `mfa-demo` use PingOne **Native Flows** (`response_mode=pi.flow`), which is *not* a browser redirect flow:

1. The server calls `GET /as/authorize?...&response_mode=pi.flow` → JSON with a flow ID + `ST`/`ST-NO-SS` cookies
2. `POST /flows/{id}` with vendor content types to submit credentials, then the OTP
3. `GET /as/resume?flowId=` → authorization code
4. `POST /as/token` → `grant_type=authorization_code`

What follows from that, and what to tell anyone reusing this code:

- **There is no `/callback` handler in any of the 20 apps.** The `redirect_uri=http://localhost:3000/callback` in the authorize URL is a required parameter echo for the token exchange, not a live endpoint.
- **The authorize URL has no `state`, no `nonce`, and no PKCE.** That is defensible *here* — there is no browser redirect to protect and the client is confidential. It is **not** defensible in a normal redirect-based app. Anyone lifting this URL template into a redirect flow inherits a request with no CSRF and no code-interception protection, and nothing in the sample warns them.
- `/flows/{id}` is a management-plane API needing **both** an admin worker Bearer token **and** the replayed session cookies. Omitting either returns 401.

## 6. Failure modes

| Symptom | Cause / fix |
|---|---|
| `401`/`403` during setup | Wrong role (see table), wrong environment ID, or auth/API region mismatch. Both paths must use the same region. |
| `406 Not Acceptable` on `/flows/` | `Accept` restricted to `application/json`. Must be `Accept: */*` — PingOne uses `application/vnd.pingidentity.*+json`. |
| `415 Unsupported Media Type` | Sent `application/json` to `/flows/{id}`. Use the vendor type, e.g. `application/vnd.pingidentity.usernamePassword.check+json`. |
| Setup worked earlier, now 401 | The bootstrap token expired (1 hour). Re-mint and re-export `accessToken`. |
| `UNIQUENESS_VIOLATION` on role assign | Already assigned. Continue. |
| `Pairing device type EMAIL is not allowed` | `mfa-demo` only. Enable Email in PingOne Console → Authentication → Policies → default device auth policy. |
| OTP never arrives | `PINGONE_TEST_EMAIL` must be a real reachable mailbox. There is no mock — `mfa-demo` and `user-registration` cannot be completed unattended. |
| `Missing required environment variables` | No `.env` in the stack dir. Run `setup.sh` from the stack root first. |
| `Failed to get admin token at startup: invalid_client` | `mfa-demo` only — it validates credentials at boot; the other samples don't. The message carries a PingOne **Correlation ID** — quote it in support tickets. |
| `jq: command not found` | `brew install jq`. Every `setup.sh` needs it. |
| Frontend loads, API calls fail | Backend not on :3000, or the proxy is misconfigured. React and Angular need two processes. |
| macOS: `NotOpenSSLWarning ... LibreSSL` | System Python 3.9 noise, harmless. Use a venv on 3.11+ to silence it. |

## 7. Teardown

Not implemented. `setup.sh` is idempotent and reuses resources by name, so leakage is bounded but non-zero — assume every run leaves resources behind in the sandbox environment. Clean up manually in the PingOne console.

`custom-admin-role` is the one to watch: it creates a custom role **and** a demo worker app when the app runs, not during setup.

## 8. Verifying your work

`setup.sh` exits 0 on success and 1 on any failure, with `error: <step> failed — <code>: <message>` on stderr. There is no JSON output mode; parse stderr.

The repo has **no CI and no application tests** — the only test is `scripts/test-load-env.sh`, covering the `.env` parser. Never assume a sample works because it is committed. Run it.
