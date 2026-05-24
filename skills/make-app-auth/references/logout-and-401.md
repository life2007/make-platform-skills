# Logout And 401

Use this reference when implementing or reviewing user-facing behavior for auth failures and logout.

## Token Mode

401 means the user-provided debug token is missing, expired, malformed, or rejected.

Behavior:

- Show update-token state.
- Do not redirect to Org.
- Do not call `/challenge`.
- Do not build OAuth URLs.

Recommended message:

```text
当前调试 Token 已失效，请更新 Token 后重试。
```

403 means the token is valid but lacks permission:

```text
当前 Token 无权限访问该资源。
```

Logout in token mode should only clear local debug state managed by the App or SDK. It should not claim to clear Org browser login.

## Unified Mode

401 means browser session is missing or expired.

Behavior:

- Show only a neutral loading state while the browser is being redirected.
- Use `auth.login({ redirect: true })` to enter the Org login page.
- If the current App session may be stale or broken, first clear the App session with `auth.logout({ redirect: false })`, then call `auth.login({ redirect: true })`. Ignore logout failures in this recovery path so the browser is not stranded.
- Do not render an App-owned login page, login transition page, or signed-out completion page.
- Do not silently redirect on every API call.
- Do not leave business views in a schema/list/create/update/delete error state for 401. Route 401 through the shared expired-session handler.

Logout:

```js
await auth.logout();
```

Do not construct Org logout URLs in generated App code. make-gateway and Org own global logout behavior.

The SDK calls make-gateway logout and follows the gateway-provided App `redirectUri`. Generated App code must not rebuild this flow, consume deprecated `orgSsoLogoutUrl` directly, or patch wrong logout URLs in UI code. After the App loads again, `auth.init({ redirect: true })` decides whether the user should enter the Org login page.

## Error Handling Pattern

Handle 401/403 in one Make API adapter or data-source layer. Every frontend request to the Make backend, including schema/meta, records, lookup, user, department, and file APIs, must go through that shared handler.

```js
async function handleMakeRequestError(error) {
  if (error instanceof MakeAppUnauthorizedError) {
    if (authMode === 'token') {
      renderTokenExpired({ message: '当前调试 Token 已失效，请更新 Token 后重试。' });
      return;
    }
    renderLoading();
    try {
      await auth.logout({ redirect: false });
    } catch {
      // Recovery must continue to login; logout failure should not strand the user.
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

try {
  await auth.api.post('/data/v1/record', payload);
} catch (error) {
  await handleMakeRequestError(error);
}
```

## Anti-patterns

- Redirecting to Org automatically on every 401 in token mode.
- Handling 401 only in App bootstrap while business requests use unhandled `auth.api` calls.
- Calling `auth.api` directly from scattered UI components without the shared 401/403 handler.
- Using raw `window.fetch('/api/make/...')` for any Make backend request.
- Rebuilding Org authorize/logout URLs in App code.
- Hard-coding Org, unified-login, or account-center environment domains in App code.
- Clearing `zs_session` or `make_app_session` from App code.
- Treating 403 as a login-expired state.
