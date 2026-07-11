# Immutable request headers on Cloudflare Workers

_Why `request.headers.set(...)` throws on Cloudflare Workers even after `.clone()`, and how to rewrite headers (e.g. overriding `origin` for cross-origin auth) safely._

> **Context:** Cloudflare Workers (Wrangler, Hono framework) + Better Auth. Surfaces with any auth plugin or middleware that mutates an incoming request's headers — common when a non-browser client (native/desktop app, proxy) needs its `origin` header rewritten before auth runs.

## Summary

Per the Fetch spec, `Request.headers` on an **incoming** request sits in the `"immutable"` guard, and **cloning a Request preserves that guard**. Any code that does `request.clone().headers.set(...)` throws `TypeError: Can't modify immutable headers` on Cloudflare Workers.

Node-based servers are lax and let this slide, so the same code works in local Node dev and only fails once deployed to Workers. A plugin that overrides the `origin` header this way makes every affected `/api/auth/*` call return `500 Internal Server Error` on Workers.

## Root cause

The failing pattern looks like this:

```js
async onRequest(request, _ctx) {
  const overrideOrigin = request.headers.get("x-override-origin");
  if (!overrideOrigin) return;
  const req = request.clone();               // headers still immutable on Workers
  req.headers.set("origin", overrideOrigin); // ← throws on Workers
  return { request: req };
}
```

`.clone()` does not reset the header guard. Workers enforces the spec strictly; Node does not.

## The reusable lesson

**To change headers on an incoming request in a Worker, do not clone-and-mutate. Build a fresh `Headers` object and a new `Request`:**

```js
const headers = new Headers(request.headers);   // mutable copy
headers.set("origin", overrideOrigin);

const req = new Request(request.url, {
  method: request.method,
  headers,
  body: request.method === "GET" || request.method === "HEAD"
    ? undefined
    : request.body,
  redirect: request.redirect,
});
```

A `Headers` constructed from another `Headers` is a plain, mutable copy — it does not inherit the `"immutable"` guard. Re-attaching it via `new Request(...)` gives you a request you fully control.

## Applying it as a wrapper (when you can't patch the plugin)

If the misbehaving code is in a third-party plugin, disable its origin override and do the rewrite yourself in the route handler before calling the auth handler:

```ts
// auth config: turn off the plugin's own origin override
plugins: [somePlugin({ disableOriginOverride: true })],
```

```ts
// Hono route: rebuild the request with a mutable header set
app.on(["GET", "POST"], "/api/auth/*", async (c) => {
  const raw = c.req.raw;
  const overrideOrigin = raw.headers.get("x-override-origin");
  if (!overrideOrigin || raw.headers.get("origin")) {
    return auth.handler(raw);
  }
  const headers = new Headers(raw.headers);
  headers.set("origin", overrideOrigin);
  const rebuilt = new Request(raw.url, {
    method: raw.method,
    headers,
    body: raw.method === "GET" || raw.method === "HEAD" ? undefined : raw.body,
    redirect: raw.redirect,
  });
  return auth.handler(rebuilt);
});
```

With this in place, `getSession`, `signUp.email`, `signIn.email`, and `signOut` all complete successfully.

## Takeaways

- **`.clone()` does not make headers mutable** — the immutable guard is preserved across clones.
- **`new Headers(oldHeaders)` does** — it produces a mutable copy with no guard.
- **Node hides this bug**; only strict runtimes (Cloudflare Workers, and the Fetch spec generally) surface it. Test auth flows on the real runtime, not just local Node dev.
- The same rebuild-the-Request technique applies to any middleware that needs to add or change headers on an inbound request in a Worker.
