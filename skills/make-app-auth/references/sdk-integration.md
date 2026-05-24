# SDK Integration

Use `@qfeius/make-app-auth` for Make App authentication and Make backend requests.

## Responsibility Boundary

The SDK owns auth bootstrap, token-mode `Authorization`, unified-login browser state, and `/api/make/**` request helpers.

App code owns page state, user-facing messages, and business feature logic.

## Dependency

Use the public npm package by default:

```json
{
  "dependencies": {
    "@qfeius/make-app-auth": "^0.1.1"
  }
}
```

Install command:

```bash
pnpm add @qfeius/make-app-auth --registry=https://registry.npmjs.org/
```

## Startup Shape

Generated Apps default to token mode and should not auto-redirect to Org unless unified login is explicitly enabled. In published unified-login mode, direct App entry should go to the Org login page instead of showing an App-owned login page.

```js
import {
  createMakeAppAuth,
  MakeAppUnauthorizedError,
  MakeAppForbiddenError
} from '@qfeius/make-app-auth';

const auth = createMakeAppAuth({
  gatewayBaseUrl: makeAuthConfig.serverUrl || makeAuthConfig.gatewayBaseUrl || '/api/make',
  unifiedLogin: false,
  tokenProvider: async () => debugToken
});

const boot = await auth.init({ redirect: false });

if (boot.status === 'authenticated') {
  renderApp({
    auth,
    context: boot.context,
    onLogout: () => auth.logout()
  });
} else if (boot.status === 'forbidden') {
  renderForbidden();
} else {
  renderTokenRequired();
}
```

## Business Requests

Use `auth.api` for Make backend calls. The SDK handles `/api/make`, cookies, JSON request bodies, unified auth errors, and token-mode `Authorization` headers.

All frontend requests to the Make backend must use `auth.api`. This includes schema/meta loading, record list/get/create/update/delete, cell updates, attachment/file APIs, lookup resolution, user candidates, department candidates, and other `/api/make/**` calls. Do not bypass the SDK for "small" helper requests; those requests still need the same Cookie, token-mode Authorization, and 401/403 semantics.

Generated Apps should expose a shared Make API adapter or data-source layer that wraps `auth.api`. UI components, drawers, tables, field editors, and route loaders should call that adapter instead of calling `auth.api` directly. This gives the App one place to handle `MakeAppUnauthorizedError` and `MakeAppForbiddenError`.

`gatewayBaseUrl` is the SDK option for the Make backend API base. In Make tooling this value already exists as `makecli` `server-url` (`makecli configure get server-url`, default `https://dev-make.qtech.cn/api/make`). Reuse that host Make backend config when generating App configuration; do not invent a separate backend URL setting.

For a deployed same-origin unified-login App, prefer the SDK default `/api/make`. For local token-mode debugging, materialize the same Make backend `server-url` into browser-safe config such as `VITE_MAKE_SERVER_URL` or the existing project config, then pass it as `gatewayBaseUrl`. Browser code must not read `~/.make/config` directly.

`gatewayBaseUrl` is not the unified login, Org, or account-center URL.

Business code should pass relative paths to `auth.api`, for example `/data/v1/record`. Do not generate absolute business URLs. If an absolute URL is unavoidable, it must be under the same origin and path scope as `gatewayBaseUrl`; otherwise the SDK rejects it and will not attach token-mode `Authorization`.

```js
const body = {
  app: 'expense',
  entity: 'reimbursement',
  fields: [],
  pagination: { page: 1, size: 10 }
};

if (hasFilterConditions) {
  body.filter = [{ title: { contains: keyword } }];
}

const result = await auth.api.post('/data/v1/record', body, {
  headers: {
    'X-Make-Target': 'MakeService.ListResources',
    'X-Trace-Id': traceId
  }
});
```

Available helpers:

```js
auth.api.get(path, init);
auth.api.post(path, body, init);
auth.api.put(path, body, init);
auth.api.patch(path, body, init);
auth.api.delete(path, init);
auth.api.request(path, init);
```

If a list request has no real filters, omit `filter`. Do not send `filter: []`.

## Shared Adapter Pattern

Keep the request adapter small and central. Business functions may normalize request and response payloads, but auth failures must go through one shared handler.

```js
async function handleMakeRequestError(error) {
  if (error instanceof MakeAppUnauthorizedError) {
    showNeutralLoading();
    try {
      await auth.logout({ redirect: false });
    } catch {
      // Keep recovery moving; a failed cleanup must not strand the user.
    }
    await auth.login({ redirect: true });
    return;
  }

  if (error instanceof MakeAppForbiddenError) {
    renderForbidden();
    return;
  }

  throw error;
}

export async function listRecords(payload) {
  try {
    return await auth.api.post('/data/v1/record', payload, {
      headers: { 'X-Make-Target': 'MakeService.ListResources' }
    });
  } catch (error) {
    return handleMakeRequestError(error);
  }
}
```

Do not leave some Make requests handled by the adapter and others handled ad hoc in UI components. A missed schema, lookup, file, user, or department request can otherwise strand the user in a business error state instead of entering the unified login flow.

## Tests To Add When Touching Auth

- Missing token in token mode.
- Expired or rejected token in token mode.
- 403 forbidden response.
- Unified-login unauthenticated state does not loop redirects.
- Business-request 401 from schema/list/create/update/delete enters the shared expired-session handler.
- Make backend calls are routed through the shared adapter; no raw `window.fetch('/api/make/...')` and no scattered unhandled `auth.api` calls in UI components.
- Authenticated unified-login state exposes a visible logout action wired to `auth.logout()`.
- Logout does not consume or rewrite `orgSsoLogoutUrl` in App code; the SDK calls make-gateway logout and follows gateway `redirectUri`, which should be an App return URL rather than an account-center or Org logout URL.

## Never Generate

- Reading, storing, or forwarding Org access tokens.
- Reading, writing, or deleting `zs_session` or `make_app_session`.
- Browser code that tries to read `~/.make/credentials`.
- Raw `Authorization` header logic outside the SDK.
- Hard-coded Org, unified-login, or account-center domains in App code.
- Passing arbitrary absolute URLs to `auth.api`.
- Constructing Org OAuth URLs, `redirect_uri`, `state`, or `code_challenge`.
- Constructing Org logout URLs or adding App-side fallback logic for `token不能为空`.
- Handling Org OAuth `code` in the App.
- Raw `window.fetch('/api/make/...')` for Make backend calls.
- Monkey-patching `window.fetch`.
- Treating browser context data as server-trusted authorization.
