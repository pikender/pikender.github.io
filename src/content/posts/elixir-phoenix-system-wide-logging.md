---
title: "System-Wide Logging for Security and Audit in Elixir/Phoenix"
author: "Pikender Sharma"
pubDatetime: 2026-10-27T06:41:29Z
description: "How we added structured access and action logs to a Phoenix/LiveView app — with trace ID correlation across HTTP and WebSocket, Okta identity propagation, and zero per-handler boilerplate."
tags: ["elixir", "phoenix", "liveview", "security", "logging", "observability"]
featured: false
draft: false
---

When you run an internal admin app, knowing _who did what and when_ is not optional. Without a structured audit trail, debugging production incidents turns into log archaeology, and security reviews become guesswork.

This post walks through how we added two structured log streams — **access logs** and **action logs** — to a Phoenix/LiveView application, tied together by a shared trace ID that bridges HTTP requests and WebSocket events.

## The Two Log Types

We wanted clear separation between two concerns:

**Access log** — one entry per HTTP request, at the infrastructure boundary. This answers: what endpoint was hit, by whom, from where, and what was the outcome.

**Action log** — one entry per write operation, at the business logic layer. This answers: what action was taken, on which object, by whom, and did it succeed.

Every entry carries a `trace_id` so you can join across both streams. A single user action in the browser typically produces one access log entry (the page load) and one or more action log entries (the LiveView events).

### Access Log Fields

| Field         | Description                                           |
| ------------- | ----------------------------------------------------- |
| `log_type`    | `"access"`                                            |
| `trace_id`    | Correlation key — matches action log for same request |
| `at`          | UTC ISO8601 timestamp                                 |
| `method`      | HTTP method                                           |
| `path`        | Request path                                          |
| `status`      | HTTP response status                                  |
| `ip`          | Client IP from peer data                              |
| `user_agent`  | `user-agent` request header                           |
| `actor_email` | Identity from Okta JWT                                |

### Action Log Fields

| Field         | Description                                              |
| ------------- | -------------------------------------------------------- |
| `log_type`    | `"action"`                                               |
| `trace_id`    | Same trace ID as the originating page load               |
| `at`          | UTC ISO8601 timestamp                                    |
| `action`      | Atom — e.g. `:add_transfer`, `:generate_migration_batch` |
| `actor_email` | Identity from Okta JWT                                   |
| `object_uuid` | UUID of the entity being acted upon                      |
| `status`      | `:ok` or `:error`                                        |
| `endpoint`    | HTTP path or `"LiveView:ModuleName#event"`               |

## Architecture

```
HTTP Request
    └── Plug.RequestId           (generates trace_id)
    └── Plugs.StoreTraceId       (stores trace_id + actor_email in session)
    └── Plugs.AccessLog          (emits access log on response)

WebSocket connect
    └── LiveHooks.on_mount       (reads session → assigns trace_id, actor_email, ip)

LiveView handle_event
    └── assign(:log_action, ...)
    └── assign(:log_object_uuid, ...)
    └── Telemetry handler        (fires after every handle_event, emits action log)
```

The key insight: `trace_id` originates from `Plug.RequestId` on the HTTP request, gets written into the session cookie during that same request, then is read back into socket assigns when the LiveView mounts. All subsequent WebSocket events for that LiveView session carry the original `trace_id`.

## The Trace ID Bridge

The challenge with LiveView is that WebSocket events have no HTTP request. `Plug.RequestId` sets `Logger.metadata()[:request_id]` — but that metadata is process-local and doesn't survive into the LiveView process.

The solution is two plugs working together in the `:browser` pipeline.

**`StoreTraceId`** runs on every browser request and writes both `trace_id` and `actor_email` into the session cookie:

```elixir
defmodule SystemToolsWeb.Plugs.StoreTraceId do
  @behaviour Plug
  require Logger

  def init(opts), do: opts

  def call(conn, _opts) do
    actor_email = SystemToolsWeb.OktaIdentity.email_from_conn(conn)

    conn
    |> store_trace_id()
    |> Plug.Conn.put_session("actor_email", actor_email)
  end

  defp store_trace_id(conn) do
    case Logger.metadata()[:request_id] do
      nil -> conn
      trace_id -> Plug.Conn.put_session(conn, "trace_id", trace_id)
    end
  end
end
```

**`LiveHooks`** runs as an `on_mount` callback when any LiveView in the affected `live_session` mounts, reading both values out of the session and into socket assigns:

```elixir
defmodule SystemToolsWeb.LiveHooks do
  import Phoenix.Component, only: [assign: 3]

  def on_mount(:default, _params, session, socket) do
    ip =
      case socket.private[:connect_info] do
        %{peer_data: %{address: address}} -> address |> :inet.ntoa() |> to_string()
        _ -> nil
      end

    {:cont,
     socket
     |> assign(:trace_id, session["trace_id"])
     |> assign(:actor_email, session["actor_email"])
     |> assign(:ip, ip)}
  end
end
```

The resulting correlation across log streams looks like:

```
access_log  trace_id=GOIWqqamRcx8K3cAAB_m  method=GET  path=/transfers  actor=user@co.com
action_log  trace_id=GOIWqqamRcx8K3cAAB_m  action=add_transfer  object_uuid=29eb3f61-...
```

## Access Log Implementation

The access log plug uses `register_before_send` — a Phoenix hook that fires after the response is assembled but before it's sent. This gives us the final `conn.status`:

```elixir
defmodule SystemToolsWeb.Plugs.AccessLog do
  @behaviour Plug
  require Logger

  def init(opts), do: opts

  def call(conn, _opts) do
    Plug.Conn.register_before_send(conn, fn conn ->
      trace_id = Logger.metadata()[:request_id]
      actor_email = SystemToolsWeb.OktaIdentity.email_from_conn(conn)

      Logger.info("access_log",
        log_type: "access",
        trace_id: trace_id,
        at: DateTime.utc_now() |> DateTime.to_iso8601(),
        method: conn.method,
        path: conn.request_path,
        status: conn.status,
        ip: conn |> Plug.Conn.get_peer_data() |> Map.get(:address) |> :inet.ntoa() |> to_string(),
        user_agent: conn |> Plug.Conn.get_req_header("user-agent") |> List.first(),
        actor_email: actor_email
      )

      conn
    end)
  end
end
```

The plug is added to the endpoint after `Plug.RequestId`, so `Logger.metadata()[:request_id]` is already set when our callback fires.

## Action Log — Zero Boilerplate via Telemetry

Rather than calling an `ActionLog.log/2` function from each `handle_event`, we use Phoenix's built-in telemetry events. Every `handle_event` call emits a `[:phoenix, :live_view, :handle_event, :stop]` event after completion. We attach a handler to these in `Telemetry`:

```elixir
:telemetry.attach(
  "action-log-liveview-events",
  [:phoenix, :live_view, :handle_event, :stop],
  &SystemToolsWeb.Telemetry.handle_liveview_event/4,
  nil
)

:telemetry.attach(
  "action-log-livecomponent-events",
  [:phoenix, :live_component, :handle_event, :stop],
  &SystemToolsWeb.Telemetry.handle_liveview_event/4,
  nil
)
```

The handler checks the resulting socket assigns for two sentinel values:

```elixir
def handle_liveview_event(_event, _measurements, metadata, _config) do
  socket = result_socket(metadata)

  case socket && {socket.assigns[:log_action], socket.assigns[:log_object_uuid]} do
    {action, object_uuid} when not is_nil(action) and not is_nil(object_uuid) ->
      SystemTools.ActionLog.log(socket, %{
        action: action,
        object_uuid: object_uuid,
        status: :ok,
        liveview_module: metadata[:component] || metadata[:socket].view,
        event: metadata.event
      })

    {nil, object_uuid} when not is_nil(object_uuid) ->
      Logger.warning("action_log skipped: log_object_uuid is set but log_action is missing",
        event: metadata.event
      )

    _ ->
      :ok
  end
end

defp result_socket(%{socket: socket}), do: socket
defp result_socket(_), do: nil
```

The pattern is deliberate: handlers that perform write operations signal intent through assigns rather than function calls. The `handle_event` body itself stays focused on business logic:

```elixir
def handle_event("save", params, socket) do
  case do_the_work(params) do
    {:ok, result} ->
      {:noreply,
       socket
       |> assign(:log_object_uuid, result.institution_uuid)
       |> assign(:log_action, :generate_migration_batch)
       |> put_flash(:info, "Done")}

    {:error, _} ->
      {:noreply, put_flash(socket, :error, "Failed")}
  end
end
```

Events without `log_object_uuid` pass through silently. Only write operations get logged.

## A Bug Worth Calling Out: The Telemetry Detach Problem

An early draft of the handler had a fallback:

```elixir
action: socket.assigns[:log_action] || String.to_existing_atom(metadata.event)
```

The intent was to use the event name when `log_action` wasn't explicitly set. The problem: `String.to_existing_atom/1` raises `ArgumentError` if the atom hasn't been interned yet. Telemetry's contract is that **any handler that raises gets permanently detached** for the lifetime of the node.

One LiveView with a missing `log_action` assign would silently disable all action logging for every subsequent request — no error, no warning, just missing logs.

The fix: remove the fallback entirely. When `log_object_uuid` is set but `log_action` is missing, emit a `Logger.warning` so the developer sees it immediately. Explicit is better than accidentally catastrophic.

## IP Address — The Session Cookie Budget Problem

IP for HTTP actions is straightforward: `Plug.Conn.get_peer_data/1`. For LiveView, the naive approach is to store IP in the session alongside `trace_id` and `actor_email`.

We didn't do that. Phoenix's cookie session store has a **4KB limit**, and the session already carries `trace_id`, `actor_email`, CSRF token, and Phoenix LiveView internal state. We'd already seen session overflow errors in QA on other work.

Instead, Phoenix LiveView's WebSocket supports `connect_info`, which can carry `peer_data` directly from the WebSocket upgrade handshake. Configure it in the endpoint:

```elixir
socket "/live", Phoenix.LiveView.Socket,
  websocket: [connect_info: [:peer_data, session: @session_options]]
```

Then read it in the `on_mount` hook:

```elixir
ip =
  case socket.private[:connect_info] do
    %{peer_data: %{address: address}} -> address |> :inet.ntoa() |> to_string()
    _ -> nil
  end
```

The pattern match matters. An earlier version used bracket access: `socket.private[:connect_info][:peer_data]`. This crashed in QA because behind the load balancer, `connect_info` was `nil` — and `nil` does not implement the `Access` behaviour. The explicit `case` handles it gracefully.

## Scoped `live_session` vs. Global `on_mount`

We needed `LiveHooks.on_mount` to run only for authenticated routes, not every LiveView in the app. Two options:

**Option A:** Register the hook in the `live_view` macro in `system_tools_web.ex`. This applies globally — every LiveView unconditionally depends on session keys that may not be present on public or unauthenticated routes.

**Option B:** Wrap affected routes in named `live_session` blocks in the router:

```elixir
live_session :migrate, on_mount: [SystemToolsWeb.LiveHooks] do
  live "/institutions/:institution_uuid/migrate/product/:uuid", MigrateInstitutionLive.Index, :index
  # ...
end

live_session :transfers, on_mount: [SystemToolsWeb.LiveHooks] do
  live "/transfers", TransferLive.Index
end
```

Option B is explicit and scoped. Only the routes that need logging context get it. Other routes are unaffected.

## Output Format

In development, logs are human-readable with all metadata fields visible:

```
[info] access_log log_type=access trace_id=GOIWqq... at=2026-10-27T... method=GET path=/transfers status=200 ...
```

In production, we use `LoggerJSON.Formatters.Basic` for structured JSON lines:

```json
{
  "message": "access_log",
  "level": "info",
  "trace": "GOIWqqamRcx8K3cAAB_m",
  "metadata": {
    "log_type": "access",
    "at": "2026-10-27T14:23:01Z",
    "method": "GET",
    "path": "/transfers",
    "status": 200,
    "actor_email": "user@company.com",
    "ip": "10.0.1.45"
  }
}
```

`LoggerJSON` promotes `trace_id` from metadata to the top-level `"trace"` field, which most log aggregation tools recognize automatically.

## Process Boundary Gotcha

Logger metadata is **process-local**. The `trace_id` in `Logger.metadata()[:request_id]` is set by `Plug.RequestId` in the HTTP process. If you call `ActionLog.log/2` from a spawned `Task` or another process, `trace_id` will be `nil`.

This is why LiveView action logs don't use `Logger.metadata()` at all — they use `socket.assigns[:trace_id]`, which is a plain value in the socket struct and available regardless of which process reads it. The telemetry handler fires synchronously in the LiveView process, so it has full access to assigns.

One known gap in this approach: when `Logger.info/2` fires from the LiveView process, the `request_id` field that LoggerJSON auto-adds to the envelope will be absent (the LiveView process has no Logger metadata). This doesn't affect our `trace_id` field — that's set explicitly. The practical rule: **use `trace_id` for correlation, not `request_id`**, and don't query action logs by `request_id`.

## What's Next

Error paths are intentionally not logged automatically. If you want failed write attempts recorded, set `log_object_uuid` in the error branch of your `handle_event` too.

The pattern scales well: any future LiveView write action gets logging for free by setting two assigns. The telemetry handler does the rest.
