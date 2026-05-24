# Troubleshooting

Use this when diagnosing Make App auth failures.

Goal: quickly decide whether the issue is App code, SDK usage, make-gateway, Org, browser cookie state, or local proxy configuration.

## First Split: Token Or Unified

Ask or infer the active mode first.

- `token`: no Org redirect should happen.
- `unified`: real browser, external domain/ngrok, Org whitelist, cookies, and callback routing matter.

Most false diagnoses come from applying unified-login assumptions to token-mode local development.

## Evidence To Collect First

- Active auth mode: `token`, `unified`, or `mock`.
- Request URL and status for the failing `/api/make/**` call.
- Whether the request was made through the shared Make API adapter that wraps `auth.api`.
- Browser request headers: especially Cookie and Authorization presence, without exposing full token values in reports.
- User-facing message shown by the App.
- make-gateway response body, request ID, or trace ID when available.

## Token Mode Checklist

For 401:

- Confirm the SDK was created with `unifiedLogin: false`.
- Confirm token was provided via `accessToken`, `token`, or `tokenProvider`.
- Confirm `gatewayBaseUrl` matches the intended Make backend API base. It should normally reuse the host Make backend config / `makecli` `server-url`, not a separately invented URL.
- For local dev, prefer `/api/make` plus dev-server proxy unless explicitly using a direct environment gateway.
- Confirm browser code is not trying to read `~/.make/credentials`.
- Confirm the shared Make API adapter uses `auth.api`, and that `auth.api` is adding `Authorization`; App code should not hand-write it.
- Confirm business code passes relative paths to `auth.api`; arbitrary absolute URLs are rejected and must not receive `Authorization`.
- Confirm the token is valid against Org verify endpoint or a known protected Make API.

Expected user-facing result:

```text
当前调试 Token 已失效，请更新 Token 后重试。
```

## Unified Mode Checklist

For redirect/callback failures:

- Confirm the App is reachable through a registered external HTTPS domain or ngrok.
- Confirm Org whitelist contains the exact callback `redirect_uri`.
- Confirm `/api/make/**` routes to make-gateway from that domain.
- Confirm every schema/meta/list/create/update/delete/file/user/department request goes through the shared Make API adapter, not scattered unhandled `auth.api` calls or raw fetch.
- Confirm browser accepts and sends cookies for the App domain.
- Confirm the page does not auto-loop login on every 401.

For cookie problems:

- Inspect browser DevTools Application/Cookies for App domain.
- Check whether duplicate session cookies exist.
- Check request Cookie header sent to `/api/make/**`.
- Confirm make-gateway logs for session verification and challenge generation.
- Confirm the request is same-origin or has the expected credentials behavior.

For logout problems:

- Confirm App calls `auth.logout()`, not a hand-built Org URL.
- Confirm make-gateway returns the SDK-expected logout result.
- Confirm make-gateway returns the App return URL as `redirectUri`; account-center or Org logout URLs should not be exposed to generated App code.
- Confirm token-mode logout is not expected to clear Org browser cookies.
- In unified mode, verify post-logout browser state with a real browser, not curl only.

## Quick Decision

- 401 in token mode with no `Authorization`: App/SDK usage problem.
- 401 in token mode with valid `Authorization` format: token likely expired or Org rejected it.
- 401 in unified mode with no Cookie: browser cookie/session problem or exchange did not set cookie.
- 401 in unified mode with Cookie present: make-gateway or Org token verification problem.
- Repeated redirects after login: callback/exchange/cookie persistence problem, not UI layout.

## Fast Root-Cause Labels

- `token_missing`: no token in token mode.
- `token_expired`: 401 in token mode with a previously valid token.
- `token_forbidden`: 403 in token mode.
- `redirect_uri_not_whitelisted`: Org rejects callback.
- `api_proxy_missing`: frontend loads, but `/api/make/**` does not reach make-gateway.
- `api_adapter_missing`: Make backend requests bypass the shared adapter, so business-request 401 is not routed into the login recovery flow.
- `cookie_not_set`: exchange/login succeeded but browser has no App session cookie.
- `cookie_not_sent`: browser stores cookie but request does not include it.
- `logout_contract_mismatch`: App expects redirect/link but gateway returns a different shape.
