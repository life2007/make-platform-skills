---
name: make-app-auth
description: Use when generating, modifying, reviewing, or debugging Make App authentication and authenticated /api/make requests with @qfeius/make-app-auth. Covers local token mode, unified login/OAuth/ngrok mode, 401/403 handling, logout, cookies, sessions, redirect callbacks, and Make App auth troubleshooting.
version: 0.1.1
metadata:
  homepage: https://github.com/qfeius/make-platform-skills/make-app-auth
---

# make-app-auth

Use this skill for **Make App authentication and authenticated Make API access**.

## Scope

This skill covers:

- `@qfeius/make-app-auth` SDK integration
- local token mode with unified login disabled
- unified login, OAuth, SSO, ngrok, cookie, logout, and callback testing
- `/api/make/**` authenticated requests
- 401, 403, and logout behavior
- cookie, session, redirect, and callback troubleshooting

This skill does not cover:

- UI layout, component design, or Make App page structure; use `makeui`.
- DSL modeling or Make resource definitions; use `makedsl`.
- makecli command execution; use `makecli`.
- make-gateway or Org server implementation changes.

## Default Behavior

Default Make App local development uses **token mode** with `unifiedLogin: false`.

Do not require ngrok, Org callback whitelist, or browser OAuth unless the user explicitly asks to test unified login, OAuth, SSO, cookie behavior, logout, or redirect callback behavior.

Mode selection:

- `token`: default for local real-backend development and ordinary feature debugging.
- `unified`: explicit mode for real Org unified login, OAuth, cookies, logout, and redirects.
- `mock`: offline UI preview only; must not call real Make APIs.

## Hard Rules

- Always use `@qfeius/make-app-auth`; do not fork a separate auth implementation.
- Business requests to Make backend must go through `auth.api` under `/api/make/**`.
- All frontend requests to Make backend must go through `auth.api`, including schema/meta, list, get, create, update, delete, attachment/file, lookup, user, and department candidate requests.
- Generated Apps must centralize Make backend access in a shared API adapter or data-source layer that wraps `auth.api` and handles `MakeAppUnauthorizedError` / `MakeAppForbiddenError`. Do not scatter direct `auth.api` calls across UI components without the shared 401/403 handler.
- Do not generate raw `window.fetch('/api/make/...')` for Make backend calls.
- Do not hand-write `Authorization`; token mode must provide tokens through SDK options.
- `gatewayBaseUrl` is the SDK option for the Make backend API base. Reuse the host Make backend config first, especially the `makecli` `server-url` value; do not create a second environment concept for the same URL.
- `gatewayBaseUrl` is not the unified login or account-center URL. Prefer the SDK default `/api/make` for deployed same-origin Apps.
- Do not configure or hard-code unified login, Org, or account-center URLs in generated App code; make-gateway returns those URLs.
- Do not read, write, persist, or delete `zs_session` or `make_app_session` in App code.
- Do not construct Org OAuth URLs, `redirect_uri`, `state`, `code_challenge`, token exchange, or Org logout URLs in generated App code.
- Browser code cannot read `~/.make/credentials`.
- Do not silently switch token mode to unified login because a token is missing or expired.

## Pre-flight Workflow

1. Determine auth mode.
   - If the user asks for normal local development, feature debugging, or backend API integration, use `token`.
   - If the user explicitly asks for unified login, OAuth, SSO, cookie, logout, ngrok, or redirect callback testing, use `unified`.
   - If the user asks for pure UI preview, use `mock`.
2. Read `references/sdk-integration.md` before generating or changing auth code.
3. Read exactly one mode reference unless troubleshooting requires more.
   - Token/default local development: `references/local-token-mode.md`
   - Unified login testing: `references/unified-login-mode.md`
   - 401, 403, logout: `references/logout-and-401.md`
   - Incident/debugging: `references/troubleshooting.md`
4. Keep auth bootstrap thin. Business features must consume the project Make API adapter and auth state, not auth internals.
5. Verify every Make backend request path uses the shared adapter and that the adapter catches `MakeAppUnauthorizedError` / `MakeAppForbiddenError`.
6. When changing generated code, add or update tests for the touched auth path: missing token, expired token, 403, logout, unified-login redirect, or business-request 401 handling.

## Reference Selection

- SDK contract and request wrapper: `references/sdk-integration.md`
- Default local token mode: `references/local-token-mode.md`
- Real unified login mode: `references/unified-login-mode.md`
- 401, 403, and logout behavior: `references/logout-and-401.md`
- Auth incident diagnosis: `references/troubleshooting.md`

## Expected User-Facing Behavior

Token mode:

- Missing token renders a token-required state.
- Expired token renders: `当前调试 Token 已失效，请更新 Token 后重试。`
- 403 renders: `当前 Token 无权限访问该资源。`
- `auth.api` may attach `Authorization`, but only to URLs under the configured Make API gateway base; generated business code should pass relative paths such as `/data/v1/record`.
- Do not redirect to Org and do not call OAuth challenge.

Unified mode:

- Direct App entry should call `auth.init({ redirect: true })` and go to the Org login page; do not show an App-owned login page.
- Do not generate an App-owned login page, login transition page, or signed-out completion page. The only acceptable unauthenticated UI is a neutral loading state while the browser is being redirected.
- Authenticated App shell must expose a visible logout action, normally in the top header near the current user/avatar.
- Login and logout must use SDK APIs.
- Formal unified-login Apps should not pass `accessToken`, `token`, or `tokenProvider`; successful login relies on the App session Cookie written by make-gateway.
- Logout UI should call `auth.logout()` or the project auth wrapper that delegates directly to `auth.logout()`; do not add App-side Org logout URL fallback logic.
- Callback testing requires a registered external domain or ngrok plus Org redirect whitelist.

## Collaboration With makeui

When `makeui` is generating or editing Make App frontend code, this skill owns all authentication decisions. `makeui` may design UI states around auth results, but it must not invent OAuth, cookie, token, logout, or `/api/make/**` request logic.
