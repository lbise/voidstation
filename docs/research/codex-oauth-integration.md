# Codex OAuth integration constraints

Research date: 2026-09-04

## Decision

**Do not ship a Voidstation "Codex OAuth" connection by copying the OAuth client used by Codex CLI, pi, or OpenCode.** OpenAI documents ChatGPT sign-in for its desktop app, Codex CLI, and IDE extension. It does not document a third-party Codex OAuth client registration, public Codex backend contract, or permission to reuse the Codex CLI client identity. That makes the flow technically reproducible but not a documented, maintainable product integration.

Codex OAuth is a hard alpha requirement. Therefore the first deployable alpha is blocked on an OpenAI-provided client registration or written authorization that names Voidstation and defines its redirects, flows, scopes, backend route, supported plans and models, support channel, and change notice. Do not silently substitute an API key and call that requirement met.

Operator-configured OpenAI-compatible endpoints can ship independently with operator-supplied API-key authentication. They must remain a separate connection kind. A ChatGPT subscription token must never be treated as a generic bearer key for one of those endpoints.

## Confidence and source labels

Claims below are marked as follows:

- **[OpenAI docs]** means OpenAI's published documentation.
- **[OpenAI source]** means the `openai/codex` repository, which describes the official client implementation but does not grant third-party authorization.
- **[pi source]** and **[OpenCode source]** mean observations from those third-party implementations.
- **[Gap]** is a question not answered by the sources reviewed.

This note reports the current sources, not a promise that an undocumented endpoint will keep behaving this way.

## What OpenAI actually documents

OpenAI says Codex supports two sign-in methods: ChatGPT for subscription access and API key for usage-based access. It names the supported local clients as the ChatGPT desktop app, Codex CLI, and IDE extension. The browser returns credentials to Codex after the ChatGPT sign-in. [OpenAI docs: Authentication](https://developers.openai.com/codex/auth/)

That page says ChatGPT sign-in follows the workspace's permissions, RBAC, retention, and residency settings. API-key usage follows the Platform organization's retention and data-sharing settings. It also says API keys are the recommended default for automation and says not to expose Codex execution in untrusted or public environments. [OpenAI docs: Authentication](https://developers.openai.com/codex/auth/)

The current pricing page says Codex is included in Free, Go, Plus, Pro, Business, Edu, and Enterprise plans, with availability and limits varying by plan, workspace, region, model, complexity, and fair-use limits. The same page says API-key usage is charged at API rates. [OpenAI docs: Codex pricing](https://developers.openai.com/codex/pricing/)

OpenAI documents device-code sign-in as **beta**, specifically for Codex CLI users whose browser callback does not work. A personal user must enable it in ChatGPT security settings, or a workspace admin must enable it. [OpenAI docs: Authentication](https://developers.openai.com/codex/auth/#login-on-headless-devices)

OpenAI documents local credential caching for the CLI and extension at `~/.codex/auth.json` or an OS credential store. It says the file contains access tokens, should be treated like a password, and that active ChatGPT sessions refresh before expiry. [OpenAI docs: Authentication](https://developers.openai.com/codex/auth/#credential-storage)

None of those documents publishes an OAuth application-registration process for an independent product, an allowed third-party callback list, public OAuth scopes or audience, a public contract for `chatgpt.com/backend-api/codex`, or permission to use the Codex CLI public client ID. The absence is material. It is not evidence of a prohibition, but it prevents Voidstation from claiming official support. **[Gap]**

## Observed protocol

### Client identity and browser authorization

The official Codex client sets the client ID to `app_EMoamEEZ73f0CkXaXp7hrann`. It uses `https://auth.openai.com/oauth/authorize`, authorization-code flow, PKCE with S256, random state, and `http://localhost:1455/auth/callback`. Its current source requests `openid profile email offline_access api.connectors.read api.connectors.invoke`, adds `id_token_add_organizations=true`, `codex_cli_simplified_flow=true`, and an `originator` value. [OpenAI source: authorization URL](https://github.com/openai/codex/blob/8e6a44b428e31f91b21edc97904fcdf4f0931ade/codex-rs/login/src/server.rs#L565-L611)

No `audience` parameter appears in that official client code. That is an observation, not a statement that the authorization server has no audience rules. **[OpenAI source]**

The official client exchanges the code at `https://auth.openai.com/oauth/token` and expects `id_token`, `access_token`, and `refresh_token`. [OpenAI source: code exchange](https://github.com/openai/codex/blob/8e6a44b428e31f91b21edc97904fcdf4f0931ade/codex-rs/login/src/server.rs#L698-L801)

pi 0.84.4 and OpenCode's current `dev` branch use the same client ID and the same authorization and token hosts. Both request the narrower `openid profile email offline_access` scope. pi uses `originator=pi`; OpenCode uses `originator=opencode`. [pi source: OAuth implementation](https://github.com/earendil-works/pi/blob/v0.84.4/packages/ai/src/auth/oauth/openai-codex.ts) [OpenCode source: Codex plugin](https://github.com/anomalyco/opencode/blob/70f74112e3f4a33ea1af8209c979a5060d7d2a36/packages/opencode/src/plugin/openai/codex.ts)

That matching client ID explains why these flows may work. It does **not** establish that OpenAI registered pi, OpenCode, or Voidstation as authorized relying parties. Neither repository can make that claim for Voidstation.

### Callback and headless behavior

The official docs identify `localhost:1455` as the normal browser callback and prescribe device code as the preferred fallback for remote, headless, or callback-blocked environments. [OpenAI docs: headless login](https://developers.openai.com/codex/auth/#login-on-headless-devices)

pi listens on `127.0.0.1:1455/auth/callback` by default, validates state, and also lets the user paste the final redirect URL or code. [pi source](https://github.com/earendil-works/pi/blob/v0.84.4/packages/ai/src/auth/oauth/openai-codex.ts)

OpenCode listens on port 1455 at `/auth/callback`, validates state, and waits up to five minutes. [OpenCode source](https://github.com/anomalyco/opencode/blob/70f74112e3f4a33ea1af8209c979a5060d7d2a36/packages/opencode/src/plugin/openai/codex.ts)

For device code, the official client POSTs the client ID to `/api/accounts/deviceauth/usercode`, opens `https://auth.openai.com/codex/device`, and polls `/api/accounts/deviceauth/token`. A successful poll returns an authorization code and PKCE verifier, which the client exchanges with redirect URI `https://auth.openai.com/deviceauth/callback`. The official client bounds polling to 15 minutes. [OpenAI source: device code](https://github.com/openai/codex/blob/8e6a44b428e31f91b21edc97904fcdf4f0931ade/codex-rs/login/src/device_code_auth.rs)

pi follows that observed device flow and uses a 15-minute timeout. OpenCode follows the same endpoint pattern but its plugin's polling loop has no visible overall timeout. [pi source](https://github.com/earendil-works/pi/blob/v0.84.4/packages/ai/src/auth/oauth/openai-codex.ts) [OpenCode source](https://github.com/anomalyco/opencode/blob/70f74112e3f4a33ea1af8209c979a5060d7d2a36/packages/opencode/src/plugin/openai/codex.ts)

### Tokens, account selection, and refresh

The official client parses the ID-token JWT and expects the namespaced `https://api.openai.com/auth` claims to contain `chatgpt_plan_type`, `chatgpt_user_id`, and `chatgpt_account_id`. It also parses `exp` from the access-token JWT. This is source-code evidence about token shape, not a published OpenAI JWT schema or a validation contract for Voidstation. [OpenAI source: token data](https://github.com/openai/codex/blob/8e6a44b428e31f91b21edc97904fcdf4f0931ade/codex-rs/login/src/token_data.rs)

The official client refreshes with `grant_type=refresh_token`, its client ID, and the refresh token. It persists a rotated ID token, access token, and refresh token. It treats `refresh_token_expired`, `refresh_token_reused`, `refresh_token_invalidated`, and `invalid_grant` as terminal re-login conditions, and aims to refresh JWT access tokens five minutes before expiry. [OpenAI source: refresh](https://github.com/openai/codex/blob/8e6a44b428e31f91b21edc97904fcdf4f0931ade/codex-rs/login/src/auth/manager.rs#L1575-L1658) [OpenAI source: refresh window](https://github.com/openai/codex/blob/8e6a44b428e31f91b21edc97904fcdf4f0931ade/codex-rs/login/src/auth/manager.rs#L2919-L2942)

pi requires `access_token`, `refresh_token`, and numeric `expires_in` in the token response, saves an absolute expiry, and requires an account ID parsed from the access-token JWT. OpenCode uses `expires_in` when present and otherwise assumes one hour. Those choices differ from the official client, which derives the early-refresh decision from JWT expiration and does not show `expires_in` in its exchange type. This is a concrete maintainability risk, not a harmless implementation detail. [pi source](https://github.com/earendil-works/pi/blob/v0.84.4/packages/ai/src/auth/oauth/openai-codex.ts) [OpenCode source](https://github.com/anomalyco/opencode/blob/70f74112e3f4a33ea1af8209c979a5060d7d2a36/packages/opencode/src/plugin/openai/codex.ts) [OpenAI source: exchanged tokens](https://github.com/openai/codex/blob/8e6a44b428e31f91b21edc97904fcdf4f0931ade/codex-rs/login/src/server.rs#L698-L801)

OpenAI's official storage supports plaintext `auth.json` mode with Unix `0600`, OS keyring mode, or automatic keyring with file fallback. The docs say to treat the file as a password. [OpenAI source: storage](https://github.com/openai/codex/blob/8e6a44b428e31f91b21edc97904fcdf4f0931ade/codex-rs/login/src/auth/storage.rs) [OpenAI docs: credential storage](https://developers.openai.com/codex/auth/#credential-storage)

pi stores OAuth credentials in `~/.pi/agent/auth.json`; its docs say that file is `0600`. OpenCode stores `refresh`, `access`, expiry, and optional account ID in its data-directory `auth.json`, also with `0600`. [pi source: provider docs](https://github.com/earendil-works/pi/blob/v0.84.4/packages/coding-agent/docs/providers.md) [OpenCode source: auth store](https://github.com/anomalyco/opencode/blob/70f74112e3f4a33ea1af8209c979a5060d7d2a36/packages/opencode/src/auth/index.ts)

### Inference route compatibility

The official Codex client uses the ChatGPT Codex backend. The official repository names `https://chatgpt.com/backend-api/codex/responses` in its CLI diagnostics and test fixtures. [OpenAI source](https://github.com/openai/codex/blob/8e6a44b428e31f91b21edc97904fcdf4f0931ade/codex-rs/cli/src/doctor.rs#L3648-L3666)

pi defines a separate `openai-codex-responses` transport with base URL `https://chatgpt.com/backend-api`. It sends the bearer token, `chatgpt-account-id`, `originator: pi`, user agent, experimental Responses headers, and a Codex-specific request shape to `/codex/responses`. It explicitly notes that this backend rejects `store: true`. [pi source: provider](https://github.com/earendil-works/pi/blob/v0.84.4/packages/ai/src/providers/openai-codex.ts) [pi source: transport](https://github.com/earendil-works/pi/blob/v0.84.4/packages/ai/src/api/openai-codex-responses.ts#L380-L470) [pi source: headers](https://github.com/earendil-works/pi/blob/v0.84.4/packages/ai/src/api/openai-codex-responses.ts#L1560-L1649)

OpenCode removes the ordinary Authorization header, substitutes the OAuth bearer token, adds `ChatGPT-Account-Id` when it can parse one, and rewrites requests intended for `/v1/responses` or `/chat/completions` to `https://chatgpt.com/backend-api/codex/responses`. It filters its OAuth model list and sets zero cost. [OpenCode source](https://github.com/anomalyco/opencode/blob/70f74112e3f4a33ea1af8209c979a5060d7d2a36/packages/opencode/src/plugin/openai/codex.ts)

This is **not** normal OpenAI-compatible API routing. A configurable endpoint may support Chat Completions or the public Responses API with an API key, but the observed Codex subscription backend, extra headers, account selection, model allow-list, and response behavior are a different connection. Do not point a Codex OAuth credential at an operator endpoint. Do not point an operator API key at the ChatGPT Codex backend.

## Risks that block a maintainable release

1. **Client-identity risk.** pi and OpenCode copy the official Codex client ID. Voidstation would need its own OpenAI-authorized identity and redirect registration. A copied ID can stop working, be restricted, or make the consent and support story misleading.
2. **Undocumented backend risk.** `/backend-api/codex/responses`, its headers, account claim, device endpoints, model set, and response details are implementation observations. They are not a published third-party API contract.
3. **Entitlement risk.** pi and OpenCode label the flow "Plus/Pro", while the current OpenAI pricing page lists a broader and changing set of plans. Workspace membership, role, region, plan, model, fair-use limit, and admin settings all affect access. The server, not Voidstation, owns those decisions.
4. **Refresh-rotation risk.** The official client recognizes one-time or invalidated refresh-token cases. A stale concurrent writer can strand a user after rotation. Store updates must be atomic and serialized per connection, and terminal errors must revoke the local connection and demand re-authentication.
5. **Secret and privacy risk.** These are bearer and refresh credentials for a ChatGPT workspace, not ordinary endpoint keys. The docs attach workspace retention and residency controls to ChatGPT sign-in. Sending them to a self-hosted server changes the threat model and needs explicit user consent, encryption at rest, redacted logs, export/delete behavior, and an operator trust boundary.
6. **Device-code risk.** OpenAI calls it beta and requires user or admin enablement. Voidstation must not offer it as a universally available fallback.
7. **Protocol drift risk.** pi's and OpenCode's token-expiry and model-filter behavior already diverge from the official client. Tracking their code means tracking three moving implementations without an OpenAI compatibility commitment.

## First alpha plan

### Release gate

Before implementation begins, obtain an OpenAI answer that explicitly permits Voidstation to offer ChatGPT subscription sign-in. The answer must provide a Voidstation client ID and registered redirect URI set, or say that OpenAI wants a different integration path. Record the answer and any terms in the release decision.

If that answer is absent, the alpha cannot truthfully meet the hard Codex OAuth requirement. Ship neither a copied client ID nor an auth-cache import as a workaround.

### Connection model

Implement two isolated connection kinds:

- `openai_compatible_api_key`: operator config includes HTTPS base URL, protocol flavor (`chat_completions` or `responses`), credential reference, and explicitly configured models. This uses only the endpoint's own documented API-key contract.
- `codex_subscription_oauth`: disabled unless the OpenAI release gate is satisfied. Its credentials and request transport must be dedicated to the OpenAI contract OpenAI provides. It never accepts an arbitrary base URL.

Persist only a secret reference in normal connection records. Keep the secret payload in an encrypted, per-user secret store. Treat access and refresh tokens, callback codes, device codes, and authorization URLs with query values as secrets. Redact them from logs, events, errors, diagnostics, backups, and issue reports. Delete all credential material on disconnect or permanent refresh failure.

### OAuth implementation, only after authorization

1. Use the client ID, exact redirect URI, scopes, audience requirements, device-code permission, and token endpoint supplied by OpenAI. Do not copy pi/OpenCode constants.
2. Use authorization code plus PKCE S256 and a cryptographically random, single-use state. Bind the state, PKCE verifier, user, and connection attempt. Reject wrong, missing, reused, or expired state.
3. Register only the approved loopback or HTTPS callback. For an SSH or server-hosted alpha, use device code only if OpenAI authorizes it for Voidstation. Bound polling, honor `slow_down`, and make cancellation stop the attempt.
4. Parse only the claims OpenAI documents for Voidstation. If an account or workspace selector is required, render it only from a verified, documented claim or API. Do not parse an unverified JWT merely to make a routing decision.
5. Save a token bundle atomically after the initial exchange and after every refresh. Serialize refreshes by connection, retain the newest rotated refresh token, refresh before the documented expiry, and disconnect on a terminal invalid-grant or revoked-token response.
6. Build a dedicated Codex transport from OpenAI's supplied route and headers. It must not share the OpenAI-compatible endpoint transport. Keep model availability server-owned and display backend errors and usage limits without inventing local quota calculations.
7. Add a connection screen that states whether use draws from ChatGPT subscription limits or API billing. Link users to OpenAI's current usage page. Show the active workspace only when OpenAI provides a supported way to obtain it.

### Acceptance tests

Use a local fake authorization server and recorded, redacted fixtures. No test account token belongs in the repository.

- Authorization-code success, denied consent, callback timeout, cancel, missing/wrong/reused state, and PKCE-verifier mismatch.
- Browser callback and approved headless/device-code paths. Device polling must stop on expiry, cancellation, terminal error, and `slow_down`.
- Initial persistence is private. Refresh rotation is atomic under two simultaneous requests. Expired, reused, invalidated, and invalid-grant refresh failures clear the connection and require a fresh login.
- Credential redaction covers structured logs, HTTP error text, telemetry, backups, export, and UI errors.
- The Codex transport cannot accept an operator URL. The OpenAI-compatible connection cannot load a Codex subscription token. Each connection makes only its configured protocol calls.
- Contract tests run only against the documented OpenAI contract supplied in the release gate. A change in client registration, callback, token fields, claims, route, headers, supported models, or entitlement response blocks promotion rather than falling back to copied behavior.

## Questions to take to OpenAI

1. Will OpenAI register Voidstation as a supported third-party Codex/ChatGPT OAuth client? If yes, what client type, client ID, redirect URIs, scopes, audience, and consent copy are approved?
2. Is device-code authorization available to this client? Which personal security and managed-workspace admin settings govern it, and what polling, expiry, and error contract applies?
3. What endpoint and headers may Voidstation call with its subscription tokens? Is `chatgpt.com/backend-api/codex/responses` supported for third parties, or is there another supported interface?
4. Which token fields and verified claims are contractual? How should Voidstation select or display a workspace, honor residency, and handle refresh-token rotation and revocation?
5. Which ChatGPT plans, regions, workspace roles, and models support Voidstation? What is the machine-readable entitlement and usage-limit contract?
6. What data handling, retention, audit, support, deprecation notice, and incident-revocation obligations apply when a self-hosted operator stores and uses these credentials?

## Sources reviewed

- [OpenAI Codex authentication documentation](https://developers.openai.com/codex/auth/)
- [OpenAI Codex pricing](https://developers.openai.com/codex/pricing/)
- [OpenAI Codex CLI repository, pinned commit `8e6a44b`](https://github.com/openai/codex/tree/8e6a44b428e31f91b21edc97904fcdf4f0931ade)
- [pi 0.84.4 source and documentation](https://github.com/earendil-works/pi/tree/v0.84.4)
- [installed pi coding agent 0.84.4, including bundled `@earendil-works/pi-ai` 0.84.4]
- [OpenCode source, pinned commit `70f7411`](https://github.com/anomalyco/opencode/tree/70f74112e3f4a33ea1af8209c979a5060d7d2a36)
