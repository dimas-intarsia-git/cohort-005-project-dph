# Research: keeping in-app Notifications fresh

Issue: #7. Question: what are the viable ways for Cadence, a React Router app on a single SQLite file read through Drizzle, to show new Notifications (an unread badge and a list) without a full reload, and what does each way cost? This note collects facts and trade-offs only. It does not pick an option.

## Version note (read first)

The ticket says "React Router v7", but this repo pins **React Router 8.3.0** (`react-router`, `@react-router/dev`, `@react-router/node` and `@react-router/serve` in `package.json`, and the installed `node_modules/react-router/package.json`). Every source-code claim below was checked against the installed 8.3.0 build, and the docs quoted are the current reactrouter.com pages, which link to `api.reactrouter.com/v8`. The three mechanisms covered here (loader revalidation, `useRevalidator`/`useFetcher`, and resource routes) work the same way in v7 and v8.

Other versions in use: `better-sqlite3` 12.11.1, `drizzle-orm` 0.45.2, `compression` 1.8.1 (pulled in by `@react-router/serve`).

## Facts about this codebase

- **One connection, WAL mode, synchronous driver.** `app/db/index.ts` lazily opens a single `better-sqlite3` connection to `data.db` and sets `journal_mode = WAL` and `foreign_keys = ON`. All Drizzle queries in the process share that one connection.
- **Server: `react-router-serve` (Express).** `npm start` runs `react-router-serve ./build/server/index.js`. In `@react-router/serve/dist/cli.js` that server calls `app.use(compression())` (line 113, for non-RSC builds) and `app.use(morgan("tiny"))`.
- **SSR is on.** `react-router.config.ts` sets `ssr: true`.
- **The badge would naturally live in `routes/layout.app.tsx`.** It is the pathless layout around every signed-in page and renders `<Sidebar>`. Its loader already runs several queries per request: all users, the current user, recently progressed courses with per-course lesson counts and progress, and a team-admin lookup.
- **No deployment target is checked in.** There is a `.dockerignore` but no Dockerfile, fly.toml, or platform config. Any deployment constraint below is conditional on where the app ends up running.
- **No Notification code exists yet** (`grep -ri notification app/` finds nothing).

## Option A: loader revalidation on navigation (no extra mechanism)

### How it works

- Docs: "In framework mode with SSR, route loaders are automatically revalidated after all navigations and form submissions" ([Route Module: shouldRevalidate](https://reactrouter.com/start/framework/route-module)). The [Revalidation Optimization](https://reactrouter.com/how-to/optimize-revalidation) page adds: "Failed submissions returning a 4xx/5xx status do not trigger revalidations by default". It also says SPA Mode and Data Mode are opt-in for GET navigations, so the "every navigation" behavior depends on `ssr: true`.
- Source confirms this for pathless layouts. The core router's `defaultShouldRevalidate` would be `false` for a pathless layout when only the child path changes (`router.js` `getMatchesToLoad` / `isNewRouteInstance`). But the framework-mode single-fetch strategy overrides it: `singleFetchLoaderNavigationStrategy` in `lib/dom/ssr/single-fetch.js` calls `m.shouldCallHandler(defaultShouldRevalidate)` with `defaultShouldRevalidate = true` unless the action status was 400 or higher. With `ssr: true`, `getShouldRevalidateFunction` in `lib/dom/ssr/routes.js` passes the route's own `shouldRevalidate` through unchanged. **Result: a badge count returned from `layout.app.tsx`'s loader is re-read on every client-side `<Link>` navigation and after every successful `<Form>` or `fetcher.submit`.**
- Single fetch sends one `GET <path>.data` request per navigation, and the server runs all matched loaders. A route that opts out with `shouldRevalidate` causes a `_routes=` parameter to be added (`components.js` lines 254 and 300, `single-fetch.js` `fetchAndDecodeViaTurboStream`).

### Costs

- **Server load:** zero extra requests. The badge query adds to a request that already happens.
- **Complexity:** lowest. One query in an existing loader.
- **Deployment:** none beyond the current setup.
- **SQLite:** one extra indexed read per navigation.
- **Freshness gap:** a user who stays on one page (watching a video, or reading the notifications list itself) sees nothing new until they navigate or submit. That is the main limitation.

## Option B: client-side polling with `useRevalidator` or `useFetcher`

### `useRevalidator().revalidate()`

- Docs, verbatim: "Revalidate the data on the page for reasons outside of normal data mutations like Window focus or polling on an interval" ([useRevalidator](https://reactrouter.com/api/hooks/useRevalidator)). React Router ships no built-in interval. The app writes the `setInterval` or focus listener itself, and the docs example uses a placeholder `useFakeWindowFocus`.
- Source (`lib/hooks.js` `useRevalidator`, `lib/router/router.js` `revalidate()`): it calls `interruptActiveLoads()`, sets `revalidation: "loading"`, and if navigation is idle it starts a navigation to the **current location** with `startUninterruptedRevalidation: true`. **Every matched route loader re-runs.** On a lesson page that means root, `layout.app`, and the lesson loader, all in one `.data` request. Each route can trim this with `shouldRevalidate`, but by default every poll re-runs the whole `layout.app` loader, including the per-course progress queries described above.

### `useFetcher().load(href)` against a dedicated loader

- `fetcher.load` on a route ID sends `GET <href>.data?_routes=<thatRouteId>`, so **only that one loader runs** (`singleFetchLoaderFetcherStrategy` and `fetchAndDecodeViaTurboStream` in `lib/dom/ssr/single-fetch.js`). That is the narrow form of polling: a tiny `notifications` loader returning `{ unreadCount }` or the latest items.
- Fetchers that were loaded with `fetcher.load` are **also revalidated after actions** (`revalidatingFetchers` in `getMatchesToLoad`, `router.js`), so the badge also updates after the user's own mutations without extra code.
- [`useFetcher`](https://reactrouter.com/api/hooks/useFetcher) is the docs-sanctioned way to call a loader without navigating.

### Browser timer behavior that affects any polling

- MDN [Page Visibility API](https://developer.mozilla.org/en-US/docs/Web/API/Page_Visibility_API) lists, as a canonical use case: "An application showing a dashboard of information doesn't want to poll the server for updates when the page isn't visible". `document.visibilityState` and the `visibilitychange` event are the hooks for this.
- Same page: "Timers such as `setTimeout()` are throttled in background/inactive tabs". Throttling starts after 30 s (Firefox) or 10 s (Chrome), using a budget that "regenerates at a rate of 10 ms per second". So a background-tab poll will fire late or irregularly whether or not the app pauses it.

### Costs

- **Server load:** about N_open_tabs / interval requests per second. Example: 50 open tabs polling every 30 s comes to about 1.7 req/s. Each request is a full HTTP round trip through Express, `compression`, `morgan`, and React Router's request handling. With `useRevalidator`, each request also runs every matched loader. With a dedicated `useFetcher` route, it runs one small query.
- **Complexity:** low. A hook with `setInterval` plus visibility and focus handling. No new server infrastructure.
- **Deployment:** none. These are plain short-lived GETs, so the app works behind any proxy or serverless host and with any number of instances.
- **SQLite:** short reads. In WAL mode "readers do not block writers and a writer does not block readers" ([sqlite.org/wal.html](https://www.sqlite.org/wal.html)). better-sqlite3 runs each query synchronously on the Node main thread (README: "Easy-to-use synchronous API"), so request rate times query time is CPU taken from the event loop. Keep the query indexed, for example on `(user_id, read_at)`.
- **Freshness:** bounded by the interval, and worse in background tabs.

## Option C: Server-Sent Events from a resource route

### How it would be built

- A resource route is a route module that "exports a loader or action but does not export a default component", and its loader can return a raw `Response` ([Resource Routes](https://reactrouter.com/how-to/resource-routes)). A loader returning `new Response(readableStream, { headers: { "Content-Type": "text/event-stream" } })` therefore yields an SSE endpoint. The client uses `new EventSource("/api/notifications/stream")` inside a `useEffect`, then either sets local state or calls `revalidate()` / `fetcher.load()` when an event arrives.
- **The installed adapter supports streaming.** `@react-router/express` 8.3.0 `sendRemixResponse` calls `res.flushHeaders()` when the Content-Type matches `text/event-stream`. Its `writeReadableStreamToWritable` (`@react-router/node`) calls `writable.flush()` after **every chunk** when the writable has a `flush` method, which is what the `compression` middleware adds.
- **Compression is the known trap.** The `compression` 1.8.1 README has a "Server-Sent Events" section: "this module does not work out of the box with server-sent events ... You can achieve this by calling `res.flush()`". The adapter's per-chunk `flush()` handles this. Separately, `compression` skips any response whose `Cache-Control` contains `no-transform` (`shouldTransform` in `compression/index.js`). Setting `Cache-Control: no-cache, no-transform` on the SSE response is a belt-and-braces option. This was read from source and **not tested at runtime**.
- **Disconnect detection.** The Express adapter wires `res.on("close", () => controller?.abort())` to `request.signal` (`createRemixRequest`). The loader can listen to `request.signal`'s `abort` event to clear its interval or unsubscribe.

### Where the "new notification" signal comes from

SQLite does not push changes. There are three possible sources:

1. **In-process event emitter.** The code path that inserts a Notification also emits an event, and the SSE handler forwards it. This works only while **exactly one Node process** serves all users. A second instance would never see the other instance's events.
2. **Server-side polling of SQLite** inside each open stream, for example `SELECT max(id)` every few seconds per connection. This moves polling from the browser to the server. It still runs N queries per interval, but saves the HTTP round trips.
3. **`sqlite3_update_hook`** fires "whenever a row is updated, inserted or deleted in a rowid table", but it is registered **per database connection** ([sqlite.org/c3ref/update_hook.html](https://www.sqlite.org/c3ref/update_hook.html)), so it does not see writes from other processes. The installed better-sqlite3 12.11.1 `lib/` exposes no update-hook method (grep for "hook" finds nothing). It is effectively unavailable here.

### Protocol facts from MDN and the WHATWG spec

- **Connection limit:** "When not used over HTTP/2, SSE suffers from a limitation to the maximum number of open connections ... per browser + domain ... (6)" and it is "Won't fix" in Chrome and Firefox. Over HTTP/2 the stream limit is negotiated, with a default of 100 ([MDN: Using server-sent events](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events)). `react-router-serve` speaks HTTP/1.1, so without an HTTP/2-terminating proxy in front, **a user with 6 or more Cadence tabs open would stall further requests to the origin.**
- **Auto-reconnect:** the browser restarts a dropped connection automatically. A `retry:` field sets the delay in ms (MDN). On reconnect the browser sends the last `id:` it saw in the `Last-Event-ID` header ([WHATWG HTML §9.2](https://html.spec.whatwg.org/multipage/server-sent-events.html)), so the server can replay missed notifications.
- **Keepalives:** "Legacy proxy servers are known to, in certain cases, drop HTTP connections after a short timeout", and the spec recommends "a comment line (one starting with a ':' character) every 15 seconds or so" (WHATWG §9.2.7). The same section warns that "HTTP chunking can have unexpected negative effects on the reliability of this protocol".
- **Background tabs:** MDN's Page Visibility page exempts only WebSockets and WebRTC from throttling ("real-time network connections"). It says nothing about EventSource. An open SSE stream is a network connection, not a timer, so it still receives events, but any timers the client runs in response are throttled. This is inferred from MDN, not stated there.

### Costs

- **Server load:** one long-lived HTTP connection and one open stream per open tab, held for the whole session, plus keepalive writes. CPU is idle between events with an in-process emitter, or N queries per interval with server-side polling.
- **Complexity:** highest. It needs a resource route that builds a `ReadableStream`, cleanup on abort, keepalive comments, `id:` and `Last-Event-ID` replay if missed events matter, a client `EventSource` lifecycle per tab, and a decision on how events reach the UI.
- **Deployment constraints:**
  - It needs a host that allows long-lived responses. Serverless and function-style hosts usually cap response duration, and some proxies buffer or time out responses. Check the eventual platform. Nothing is configured in this repo yet.
  - The in-process emitter needs a single instance. SQLite already pushes toward that: WAL "does not work over a network filesystem" and "all processes using a database must be on the same host computer" ([sqlite.org/wal.html](https://www.sqlite.org/wal.html)).
  - It hits the 6-connection HTTP/1.1 limit unless HTTP/2 is terminated in front.
- **SQLite:** with the emitter, no extra reads. With server-side polling, see Option B. Avoid holding a read transaction open for the life of a stream. SQLite warns that "if there is always at least one active reader, then no checkpoints will be able to complete and hence the WAL file will grow without bound" (wal.html §6). better-sqlite3's own docs call it "generally a very bad idea to keep a transaction open across event loop ticks". Short, discrete reads avoid both problems.

## SQLite concurrency facts common to all options

- "there can only be one writer at a time" (wal.html §2.2). The badge is read-only. Marking notifications read, and inserting them, are the only writes, so read freshness does not add write contention.
- WAL "readers do not block writers" is "mostly true", with "some obscure cases where a query against a WAL-mode database can return SQLITE_BUSY" (wal.html §9).
- With one connection in one process, Node runs better-sqlite3 calls one at a time, so there is no intra-process lock contention. The cost shows up as event-loop time instead.

## Comparison

| | A: revalidate on navigation | B1: `useRevalidator` poll | B2: `useFetcher` poll of a small loader | C: SSE resource route |
|---|---|---|---|---|
| Freshness while idle on a page | None | Interval | Interval | Push (near-instant with an emitter) |
| Extra requests | 0 | tabs ÷ interval, each running **all** matched loaders | tabs ÷ interval, each running **one** loader | 1 long-lived per tab |
| New code | Query in layout loader | Hook + interval + visibility | Route + hook + interval + visibility | Resource route + stream + keepalive + reconnect/replay + client lifecycle |
| Works with multiple server instances | Yes | Yes | Yes | Only with server-side DB polling, not with an in-process emitter |
| Proxy or host constraints | None | None | None | Long-lived responses, buffering, HTTP/1.1 6-connection limit |
| Combines with A | (is A) | Yes | Yes | Yes |

These options are not exclusive. Option A comes free with the current `ssr: true` setup, and B or C would add to it.

## Open points for the decision (not answered here)

- Does a notification need to show up while a user sits on one page, such as during a video lesson? If not, Option A alone may be enough.
- What is the deployment target? It decides whether long-lived SSE responses and a single instance are acceptable.
- Should the badge loader live in `layout.app.tsx`, which already runs on every navigation, or in a dedicated route, which is cheaper to poll?

## Sources

- React Router docs: [Route Module (shouldRevalidate)](https://reactrouter.com/start/framework/route-module), [Revalidation Optimization](https://reactrouter.com/how-to/optimize-revalidation), [useRevalidator](https://reactrouter.com/api/hooks/useRevalidator), [useFetcher](https://reactrouter.com/api/hooks/useFetcher), [Resource Routes](https://reactrouter.com/how-to/resource-routes)
- React Router 8.3.0 source (installed `node_modules`): `react-router/dist/development/lib/router/router.js` (`revalidate`, `getMatchesToLoad`, `shouldRevalidateLoader`, `shouldCallHandler`), `lib/hooks.js` (`useRevalidator`), `lib/dom/ssr/single-fetch.js` (navigation and fetcher strategies, `_routes`), `lib/dom/ssr/routes.js` (`getShouldRevalidateFunction`), `@react-router/express/dist/index.js` (`createRemixRequest`, `sendRemixResponse`), `@react-router/node/dist/index.js` (`writeReadableStreamToWritable`), `@react-router/serve/dist/cli.js`
- `compression` 1.8.1 `README.md` (Server-Sent Events) and `index.js` (`shouldTransform`)
- SQLite: [Write-Ahead Logging](https://www.sqlite.org/wal.html), [sqlite3_update_hook](https://www.sqlite.org/c3ref/update_hook.html)
- better-sqlite3: [API docs](https://github.com/WiseLibs/better-sqlite3/blob/master/docs/api.md), installed 12.11.1 `README.md` and `lib/`
- MDN: [Using server-sent events](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events), [Page Visibility API](https://developer.mozilla.org/en-US/docs/Web/API/Page_Visibility_API)
- WHATWG HTML Living Standard: [Server-sent events](https://html.spec.whatwg.org/multipage/server-sent-events.html)
