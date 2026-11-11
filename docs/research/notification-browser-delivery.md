# How should Notifications reach the browser within 10–15s?

Research for [#2](https://github.com/fattung79/cohort-005-project-fork/issues/2) (map: [#1](https://github.com/fattung79/cohort-005-project-fork/issues/1)). Researched 2026-11-11.

## Recommendation

**Poll with a fetcher: a dedicated resource route, loaded every ~10s from `layout.app.tsx` with `useFetcher({ key })`, only while the tab is visible.**

- Add a resource route, for example `api/notifications/unread-count`. It has a `loader` and no default export. The loader returns the Recipient's unread count, and the newest few Notifications if the UI needs them.
- In `layout.app.tsx`, call `useFetcher({ key: "notifications" }).load(...)` on a ~10s interval. Skip the call while `document.visibilityState === "hidden"`, and fire one load straight away on `visibilitychange` back to `visible`.
- **Do not use `useRevalidator` for this.** It re-runs every loader on the page, including the layout loader.

Why polling: it fits the stack with no new infrastructure, and the 10–15s budget is far longer than one HTTP round trip. Server-Sent Events (SSE) and WebSockets buy latency that #1 says we don't need. Each costs a long-lived connection per tab, plus either an in-process pub/sub (which only works because Cadence is one process, per ADR 0006) or a server-side poll of SQLite anyway.

## The stack, as it is today

- React Router **8.3.0** in framework mode with `ssr: true`, served by `react-router-serve` (`package.json`, `react-router.config.ts`).
- `@react-router/serve` 8.3.0 is a plain Express app with the `compression` middleware. There is no custom server and no WebSocket library in `node_modules` (`node_modules/@react-router/serve/dist/cli.js`, lines 111–127).
- One SQLite file through synchronous `better-sqlite3` and Drizzle, run as one server process: "the app cannot run on more than one machine against the same data" (ADR 0006, `docs/adr/0006-one-sqlite-file-through-drizzle.md`). WAL mode is on (`app/db/index.ts`).
- Every signed-in page renders under `routes/layout.app.tsx`. Its loader is already heavy: all users, plus recent-course progress computed with several queries per course.

## Options

### 1. Polling with `useRevalidator`

- **How it fits.** The docs present this as an intended use: "Revalidate the data on the page for reasons outside of normal data mutations like Window focus or polling on an interval." ([useRevalidator](https://reactrouter.com/api/hooks/useRevalidator); the same JSDoc is in `node_modules/react-router/dist/development/lib/hooks.d.ts`). The unread count would live in the `layout.app.tsx` loader.
- **Cost per tab.** In framework mode, revalidation re-runs the active loaders on the page ([Optimize revalidation](https://reactrouter.com/how-to/optimize-revalidation)). Every 10s, each open tab would re-run the layout loader (all users, course progress) plus the current page's loader. You can limit this with `shouldRevalidate` on each route. The docs warn that always returning `false` "can leave this route's UI out of sync" and blocks explicit `useRevalidator` reloads ([Route Module: shouldRevalidate](https://reactrouter.com/start/framework/route-module)). Doing that across ~25 routes is fragile.
- **Failure modes.** It is heavy for what it delivers. A revalidation can also replace page data while the user is editing, for example in Instructor forms.
- **Verdict.** It works, but it is the wrong scope for this job.

### 2. Polling a resource route with `useFetcher` (recommended)

- **How it fits.** "A route becomes a resource route by convention when its module exports a loader or action but does not export a default component" ([Resource routes](https://reactrouter.com/how-to/resource-routes)). `fetcher.load(href)` runs only that route's loader. `useFetcher({ key })` lets any component share the same fetcher's state, such as a sidebar badge and a dropdown ([useFetcher](https://reactrouter.com/api/hooks/useFetcher)).
- **Free freshness after actions.** "`fetcher.load` only revalidates by default after action submissions and explicit `useRevalidator` calls" ([Optimize revalidation](https://reactrouter.com/how-to/optimize-revalidation)). After the user's own actions, such as marking a Notification read, the count refreshes with no extra code.
- **Server and hosting.** None needed. The poll is an ordinary short GET through the existing Express/`react-router-serve` stack. It works the same in `react-router dev`.
- **Cost per open tab.** One small request per ~10s while the tab is visible, and none while it is hidden. Each request runs one indexed SQLite query, such as `count(*) WHERE recipient_id = ? AND read_at IS NULL`. `better-sqlite3` is synchronous (ADR 0006), so the query blocks the event loop for its duration. An indexed count is in the microsecond range. Rough load: 1,000 visible tabs at 10s ≈ 100 req/s of trivial work. Add an index on (recipient, read state) when the Notification model is designed.
- **Visibility gating.** MDN gives this exact case: "an application showing a dashboard of information doesn't want to poll the server for updates when the page isn't visible" ([Page Visibility API](https://developer.mozilla.org/en-US/docs/Web/API/Page_Visibility_API)). Browsers throttle hidden-tab timers anyway: Chrome checks them once per second, and after 5 minutes hidden only once per minute ("intensive throttling"). Firefox Android may hold them off for 15 minutes ([setTimeout: Timeouts in inactive tabs](https://developer.mozilla.org/en-US/docs/Web/API/Window/setTimeout#timeouts_in_inactive_tabs)). So the poll should not depend on hidden-tab timers. Stop polling when the tab is hidden, and fetch once on `visibilitychange` to `visible`. The user then sees a fresh count the moment they look.
- **Worst-case latency.** One interval plus one round trip. At 10s that is ≈10s, within the 10–15s budget.
- **Failure modes.**
  - A failed poll shows the previous count until the next tick succeeds. Nothing has to reconnect.
  - When the session expires, the loader should return a count of 0 or a 401 rather than throw, so no error boundary fires every 10s. A thrown or returned `Response` "is considered a successful execution and will not trigger handleError" ([Resource routes](https://reactrouter.com/how-to/resource-routes)).
  - A slow response can overlap the next tick. Skip the tick while `fetcher.state !== "idle"`.

### 3. Server-Sent Events (`EventSource`) from a resource route

- **How it fits.** A resource route `loader` can return a `Response` whose body is a `ReadableStream`, with `Content-Type: text/event-stream` (required, per [MDN: Using server-sent events](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events)). React Router has no SSE-specific API. The browser side uses a plain `EventSource` in a `useEffect`, outside the loader/fetcher model.
- **Plumbing checks (verified in source).**
  - The `compression` README warns that it "does not work out of the box with server-sent events" because it buffers, and says to call `res.flush()` ([compression README](https://github.com/expressjs/compression#server-sent-events)). `@react-router/node` 8.3.0 calls `writable.flush()` after every chunk it writes (`node_modules/@react-router/node/dist/index.js`, `writeReadableStreamToWritable`), so events would get through. `Cache-Control: no-transform` also makes `compression` skip the response entirely.
  - `@react-router/express` aborts `request.signal` on `res.on("close")`. The loader can detect a closed tab and drop its subscription.
- **Where events come from.** Something must tell the open stream that a Notification was created.
  - (a) An in-process `EventEmitter` that actions emit to. This works only because Cadence is one process (ADR 0006). It breaks the moment there are two processes, and it misses rows written outside the server, such as by `scripts/seed.ts`.
  - (b) Each stream polls SQLite on a timer. That is option 2 moved to the server, plus a held-open connection.
- **Cost per open tab.** One open HTTP connection per tab for as long as the tab is open, plus periodic keep-alive comment lines. MDN: "a server can send a comment periodically to keep the connection alive". Hidden tabs keep their connection.
- **Failure modes.**
  - Without HTTP/2, browsers cap SSE at **6 connections per browser per domain, across all tabs**. Chrome and Firefox marked this "Won't fix" (MDN, same page). A user with 6+ Cadence tabs open would stall all other requests to the site. Avoiding this needs HTTP/2 in front of the Node server, a hosting concern that `react-router-serve` does not handle.
  - The browser reconnects automatically, delayed by the `retry` field (MDN). Proxies and load balancers with idle timeouts cut quiet streams.
  - Every deploy or restart drops every stream at once, and they all reconnect together.
- **Verdict.** It works, but costs a held connection per tab and adds the 6-connection risk and proxy tuning, to deliver latency we have said we don't need.

### 4. WebSockets

- **How it fits.** It doesn't. A WebSocket is "a two-way interactive communication session" ([MDN: WebSockets API](https://developer.mozilla.org/en-US/docs/Web/API/WebSockets_API)). The `Upgrade` handshake never reaches route loaders or actions. `react-router-serve` creates its Express server internally and exposes no hook to attach a WebSocket server. We would have to replace `react-router-serve` with a custom server, add a library such as `ws`, and solve the same "where do events come from" problem as SSE.
- **Cost per open tab.** One persistent socket per tab.
- **Failure modes.**
  - Reconnection, heartbeats and auth on upgrade are all hand-rolled.
  - There is no backpressure: "The WebSocket interface doesn't support backpressure".
  - "If a page has an open WebSocket connection, the browser may not add it to the bfcache" (MDN, same page).
- **Verdict.** It is built for two-way, low-latency traffic. Notifications are one-way and may lag 10–15s, so it is the wrong tool here.

### 5. Other options, briefly

- **Refresh on navigation only.** Put the count in the layout loader and do nothing else. Framework-mode loaders revalidate "after all navigations and form submissions" ([Route Module](https://reactrouter.com/start/framework/route-module)). This is free, but a user reading a long lesson would never see a new Notification, which fails the 10–15s goal. As a side effect it still applies to option 2, since the fetcher reloads after actions.
- **Long polling.** An SSE-like held request that re-opens on every event. It has the downsides of both options and no advantage here.
- **Web Push / service workers.** These deliver outside the open tab and need permission prompts and a push service. They are out of scope: #1 is in-app only.

## Comparison

| | Fits the loader/action model | New server/hosting work | Cost per open tab | Main failure mode |
|---|---|---|---|---|
| `useRevalidator` poll | Yes | None | Every page loader every ~10s | Heavy; can disturb page data mid-edit |
| **Fetcher poll of a resource route** | **Yes** | **None** | **1 tiny indexed query / ~10s while visible** | **Stale until next tick** |
| SSE | Partly (loader returns a stream; client is a raw `EventSource`) | Event source plus HTTP/2 and proxy timeouts | 1 held connection per tab, always | 6-connection cap on HTTP/1.1; mass reconnects |
| WebSockets | No | Custom server and `ws` | 1 held socket per tab | Hand-rolled reconnect/auth; bfcache |

## Notes for the PRD

- Keep the poll interval in one constant, around 10s. It is the only setting that controls lag.
- The poll endpoint is the natural "read" side of the Notification model. Shape its response so that a later push channel (SSE, or the future unified fan-out) could deliver the same payload.
- If Cadence ever runs on more than one machine (ADR 0006's stated limit), polling keeps working unchanged. SSE or WebSockets with in-process events would not.
