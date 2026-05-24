# Local Token Mode

Use token mode for ordinary local development and backend API debugging.

This is the default mode while Make App publishing and registered external domains are not available for every developer.

## When To Use

Use `token` mode when:

- The user is building or debugging normal Make App UI/features locally.
- The App is not deployed.
- There is no ngrok or registered App domain.
- The task is not explicitly about OAuth, unified login, cookies, logout, or redirect callbacks.

Do not require ngrok, Org callback whitelist, or Org login page in this mode.

## Contract

- `unifiedLogin` must be `false`.
- The browser must not read `~/.make/credentials`.
- Missing or expired tokens are local-debug states, not OAuth states.
- Business requests still go through `/api/make/**` and `auth.api`.

## SDK Setup

```js
const auth = createMakeAppAuth({
  gatewayBaseUrl: makeAuthConfig.serverUrl || makeAuthConfig.gatewayBaseUrl || '/api/make',
  unifiedLogin: false,
  accessToken: debugToken
});
```

or:

```js
const auth = createMakeAppAuth({
  gatewayBaseUrl: makeAuthConfig.serverUrl || makeAuthConfig.gatewayBaseUrl || '/api/make',
  unifiedLogin: false,
  tokenProvider: async () => getDebugToken()
});
```

`gatewayBaseUrl` is the SDK option for the Make backend API base. Reuse the same value that Make tooling calls `server-url` (`makecli configure get server-url`, default `https://dev-make.qtech.cn/api/make`). Default to `/api/make` and rely on the local dev server proxy when the App is same-origin with make-gateway.

Browser code cannot read `~/.make/config`; if local token mode must point directly to an environment gateway, expose the existing Make backend `server-url` through browser-safe project config such as `VITE_MAKE_SERVER_URL` rather than inventing a second URL concept.

Token source priority:

1. `accessToken` or `token` passed to `createMakeAppAuth`.
2. `tokenProvider()` passed to `createMakeAppAuth`.
3. Node/local debug credentials from `~/.make/credentials` using profile `default` unless configured otherwise.

Browser code cannot read `~/.make/credentials`. If a browser App needs local credentials, have the dev server or local service read them and pass a token through a controlled debug path, or ask the user to provide a token in local debug config.

Do not generate a browser-only implementation that claims to automatically load `~/.make/credentials`.

## Missing Or Expired Token

Missing token:

- Render a token-required state.
- Message: `请提供本地调试 Token 后重试。`
- Do not redirect to Org.

Expired token:

- `auth.api` receives 401.
- SDK throws `MakeAppUnauthorizedError`.
- Render an update-token state.
- Message: `当前调试 Token 已失效，请更新 Token 后重试。`
- Do not call OAuth challenge and do not auto-login.

403:

- Render forbidden/no-permission state.
- Message: `当前 Token 无权限访问该资源。`

## Local Proxy

Local browser requests should still go through the App's `/api/make/**` route and a local proxy to make-gateway. Do not bypass make-gateway by calling meta/data services directly.

Typical local route:

```text
browser -> local Vite/App dev server -> /api/make proxy -> make-gateway -> Make services
```

If local token mode points directly to an environment gateway, keep the same rule at SDK level: business code calls `auth.api('/data/v1/record')`, and the SDK attaches `Authorization` only under the configured `gatewayBaseUrl`.
