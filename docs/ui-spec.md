# gotk UI Spec

## Overview

**Server renders. Frontend orchestrates. Commands are testable functions.**

gotk is a command-based web application toolkit where every behavior — server
and frontend — is a Go function that takes input and returns DOM instructions,
testable with `go test`, no browser required.

You write commands. Commands are Go functions. A command receives a payload,
does its work, and produces a list of instructions: "put this HTML here," "show
that element," "navigate to this URL." A thin JS client (~200 LOC) applies
those instructions to the DOM. That is the entire frontend runtime.

Server commands access databases and render HTML. Frontend commands (compiled to
WASM via TinyGo) handle UI orchestration — opening modals, toggling sidebars,
managing loading states. Both use the same `Context` interface. Both produce the
same instruction types. Both are tested the same way: call the function, inspect
the output.

The architecture targets **web apps**, not websites. It prioritizes testability,
minimal dependencies, zero build steps for the frontend, and a single language
(Go) across the full stack.

### How gotk Compares

| Framework | Who renders | Who holds state | Frontend complexity | Testability |
| --- | --- | --- | --- | --- |
| React/Vue | Client | Client | High (state mgmt, build tools, JSX) | Requires JSDOM/browser |
| LiveView | Server | Server | None (magic diffing) | Tight coupling to framework |
| htmx | Server | Server | Minimal (HTTP attributes) | Test HTTP endpoints |
| **gotk** | **Server** | **Server (data) + DOM (UI)** | **Minimal (Go/WASM orchestration)** | **`go test` for everything** |

**Not React.** The frontend does not render. It orchestrates — show this, hide
that, send this command. No virtual DOM, no state management library, no build
step.

**Not LiveView.** The server does not track the DOM or diff state. It receives a
command, runs a function, and returns explicit instructions. No magic, no
session-bound DOM tree, no framework coupling. Commands are plain functions you
can call in a test.

**Not htmx.** There is a logic layer on the frontend (Go/WASM) for UI
decisions — conditionals, branching, chaining async calls. htmx stops at
declarative HTTP attributes. gotk has a programmable frontend layer when you
need it, but it only orchestrates — it never renders.

### Why This Architecture Is AI-Friendly

gotk is designed so that both AI code generators (writing gotk apps) and AI
agents (using gotk apps) work well with the same architecture.

#### For AI Generating Code

The biggest problem with AI-generated frontend code is not writing it — it is
testing it. React, Vue, Angular, and similar frameworks require browser-based
or JSDOM-based test environments, complex mocking of DOM APIs, and
framework-specific patterns (`act()` wrappers, async rendering, effect
cleanup). AI frequently gets these wrong, producing code that looks correct but
fails in subtle ways — stale closures, missing dependency arrays, race
conditions between renders.

gotk eliminates this class of problems entirely. Every command — server and
frontend — is a pure Go function that produces a data structure. Testing is
asserting on that data structure. No browser, no DOM, no async timing.

**Every command has the same shape.** AI sees three examples and can generate
the hundredth correctly. The pattern never varies:

```go
func Something(ctx *gotk.Context) error {
    x := ctx.Payload.String("x")   // read input
    // do work
    ctx.HTML("#target", html)       // produce instructions
    return nil
}
```

There are no hooks, no lifecycle methods, no component trees, no state
management patterns to choose between. Every command is a standalone function
with no ambient context.

**The instruction set is finite and closed.** There are roughly 10 context
methods to learn: `ctx.HTML`, `ctx.AttrSet`, `ctx.AttrRemove`, `ctx.Navigate`,
`ctx.Focus`, `ctx.Dispatch`, `ctx.Template`, `ctx.Populate`, `ctx.Async`,
`ctx.Error`. Compare this to React where AI must navigate `useState`,
`useEffect`, `useCallback`, `useMemo`, `useRef`, `useContext`, `useReducer`,
`useLayoutEffect`, `useSyncExternalStore`, `useTransition`,
`useDeferredValue` — each with subtle rules about when and how to use them.

**Tests are mechanically derivable.** Given a command, the test writes itself —
create context, set payload, call function, assert on instructions:

```go
func TestDeleteTodo(t *testing.T) {
    app := newTestApp()
    app.db.CreateTodo("Buy milk")

    ctx := gotk.NewTestContext()
    ctx.SetPayload(map[string]any{"id": 1})

    app.DeleteTodo(ctx)

    ins := ctx.Instructions()
    assert.Equal(t, "html", ins[0].Op)
    assert.Equal(t, "remove", ins[0].Mode)
    assert.Equal(t, "#todo-1", ins[0].Target)
}
```

No DOM mocking. No async/await. No `act()` wrappers. No timing issues. The
assertions are on concrete data structures, not on rendered DOM state.

**The error surface is narrow.** The ways a gotk command can be wrong are
limited and testable: wrong instruction type, wrong CSS selector, wrong payload
field name, missing instruction, instructions in wrong order. All caught by
simple assertion tests. Compare to React where bugs can be stale closures,
effect dependency arrays, race conditions between renders, hydration
mismatches, or memory leaks from unsubscribed effects — none caught by simple
data assertions.

**No build configuration to break.** AI-generated React code needs
`package.json`, webpack/vite config, TypeScript config, babel presets, CSS
module configuration. Getting any of these wrong means the code does not
compile. gotk needs a `.go` file and an `.html` template. `go build`. Done.

**The feedback loop is instant.** When AI generates a command + test, you run
`go test`. It passes or fails with a clear assertion error: "expected
instruction Op to be 'html', got 'navigate'." The AI reads this, understands
the problem, and fixes it in one iteration. React test failures often produce
cryptic errors about `act()` or null DOM queries that require framework-specific
knowledge to debug.

**Templates are standard Go `html/template`.** Well-known to every AI model, no
custom DSL to learn.

#### For AI Agents Consuming the App

- **The command protocol is structured JSON.** An agent connects via WebSocket
  and sends `{"cmd": "create-todo", "payload": {"title": "Buy milk"}}`. No
  browser, no screenshots, no simulated clicks.
- **The HTML is self-describing.** `gotk-click="save-user"`,
  `gotk-collect="#form"`, `<input name="email">` — an agent reading the page
  knows what commands exist, what inputs they need, and what the current state
  is.
- **No separate API to build.** The command layer that serves humans serves
  agents. You ship one interface, not two.

See the [AI Agent Interface](#ai-agent-interface) section for details.

## Mental Model

Think of the browser as a WebView in a desktop app:

- **Commands** are messages sent from the UI to a handler (like IPC in Electron).
- **Instructions** are responses that tell the UI what to change (like render
  updates).
- **The thin client** is the event loop — it binds events, sends commands, and
  applies instructions.
- **State** lives server-side (business data) or in the DOM (UI state). There is
  no client-side state store.

## Protocol

All communication between the thin client and the server happens over
WebSocket, using JSON messages. The initial page load is a standard HTTP request
that returns a full server-side rendered page.

### Client to Server (Command)

```json
{
  "cmd": "create-todo",
  "payload": { "title": "Buy milk" },
  "ref": "a1b2"
}
```

| Field     | Type   | Description                                      |
| --------- | ------ | ------------------------------------------------ |
| `cmd`     | string | Command name.                                    |
| `payload` | object | Arbitrary data collected from the DOM or hardcoded. |
| `ref`     | string | Client-generated ID to correlate the response.   |

### Server to Client (Instructions)

```json
{
  "ref": "a1b2",
  "ins": [
    { "op": "html", "target": "#todo-list", "html": "<li>...</li>" },
    { "op": "navigate", "url": "/todos" }
  ]
}
```

| Field | Type   | Description                                             |
| ----- | ------ | ------------------------------------------------------- |
| `ref` | string | Correlates to the command `ref`. Empty for server-push. |
| `ins` | array  | Ordered list of instructions to apply sequentially.     |

Server-initiated pushes (no corresponding command) omit `ref`.

## Instructions

Instructions are the atomic units of UI change. The thin client applies them
sequentially.

### `html`

Replace, append, or prepend HTML content in a target element.

```json
{
  "op": "html",
  "target": "#todo-list",
  "html": "<li>New item</li>",
  "mode": "append"
}
```

| Field    | Type   | Default     | Description                              |
| -------- | ------ | ----------- | ---------------------------------------- |
| `target` | string | required    | CSS selector for the target element.     |
| `html`   | string | required    | HTML content.                            |
| `mode`   | string | `"replace"` | One of `replace`, `append`, `prepend`, `remove`. `remove` ignores `html` and removes the target element. |

### `template`

Clone a `<template>` element's content into a target.

```json
{
  "op": "template",
  "source": "#tpl-user-form",
  "target": "#modal"
}
```

| Field    | Type   | Description                              |
| -------- | ------ | ---------------------------------------- |
| `source` | string | CSS selector for the `<template>` element. |
| `target` | string | CSS selector for the target element. Content is replaced. |

### `populate`

Fill named form elements within a container using a key-value map. For each
entry, finds `[name="<key>"]` within the target and sets its value.

```json
{
  "op": "populate",
  "target": "#modal-body",
  "data": { "name": "Alice", "email": "alice@test.com" }
}
```

| Field    | Type   | Description                                    |
| -------- | ------ | ---------------------------------------------- |
| `target` | string | CSS selector for the container.                |
| `data`   | object | Map of `name` attribute to value.              |

### `navigate`

Push a URL to browser history. Optionally includes HTML to update a target
without a server round-trip.

```json
{
  "op": "navigate",
  "url": "/settings",
  "target": "#content",
  "html": "<div>...</div>"
}
```

| Field    | Type   | Description                                     |
| -------- | ------ | ----------------------------------------------- |
| `url`    | string | URL to push via `history.pushState`.             |
| `target` | string | Optional. CSS selector to update.                |
| `html`   | string | Optional. HTML to set on the target.             |

### `attr-set`

Set an attribute on an element.

```json
{ "op": "attr-set", "target": "#sidebar", "attr": "hidden", "value": "" }
```

### `attr-remove`

Remove an attribute from an element.

```json
{ "op": "attr-remove", "target": "#modal", "attr": "hidden" }
```

### `set-value`

Set the value of a form element.

```json
{ "op": "set-value", "target": "#search-input", "value": "query" }
```

### `dispatch`

Dispatch a CustomEvent on an element.

```json
{
  "op": "dispatch",
  "target": "#form",
  "event": "reset",
  "detail": {}
}
```

| Field    | Type   | Description                                     |
| -------- | ------ | ----------------------------------------------- |
| `target` | string | CSS selector.                                    |
| `event`  | string | Event name.                                      |
| `detail` | object | Optional. Passed as `CustomEvent.detail`.        |

### `focus`

Focus an element.

```json
{ "op": "focus", "target": "#title-input" }
```

### `exec`

Execute a registered client-side JS function. Escape hatch for behaviors that
require direct DOM access (animations, scroll position, focus traps).

```json
{ "op": "exec", "name": "lockScroll", "args": {} }
```

### `cmd`

Trigger a server command from within an instruction sequence. Used by the
framework when a frontend command calls `ctx.Async(...)`. The thin client sends
this as a regular WebSocket command.

```json
{ "op": "cmd", "cmd": "edit-user", "payload": { "id": 42 } }
```

## HTML Attributes (Thin Client API)

The thin client scans the DOM on page load and after every `html` or `template`
instruction. It binds behavior based on `gotk-*` attributes.

### `gotk-click`

Send a command when the element is clicked. At init time, the thin client calls
`wasm.listCommands()` once and caches the result as a JS `Set`. On each click,
it checks `localCmds.has(cmd)` — if the command is registered locally, it runs
via WASM. Otherwise, it is sent to the server via WebSocket. No WASM boundary
crossing per interaction. The HTML does not need to know or declare where a
command runs — routing is automatic.

```html
<!-- Runs locally if "toggle-sidebar" is registered in WASM, otherwise via WS -->
<button gotk-click="toggle-sidebar">☰</button>

<!-- Server command — same attribute, routed automatically -->
<button gotk-click="delete-todo" gotk-collect="#todo-42">Delete</button>
```

### `gotk-input`

Send a command on the `input` event.

```html
<input gotk-input="search" gotk-debounce="300" name="q">
```

### `gotk-on`

Send a command on an arbitrary DOM event. Format: `event:command`.

```html
<div gotk-on="dragend:reorder-items" gotk-collect=".sortable">...</div>
```

### `gotk-navigate`

Intercept a link click and send a `navigate` command via WebSocket instead of a
full page load. The `href` is preserved for right-click, new-tab, and
bookmarking.

```html
<a href="/settings" gotk-navigate>Settings</a>
```

### `gotk-poll`

Send a command at a fixed interval.

```html
<div gotk-poll="refresh-notifications" gotk-every="30s" id="notifs">...</div>
```

### `gotk-collect`

Specifies a CSS selector for a container. When a command fires, the thin client
gathers all elements with a `name` attribute within the container and sends
their values as `payload`. Without `gotk-collect`, the payload is empty (unless
`gotk-payload` is used).

```html
<button gotk-click="create-user" gotk-collect="#user-form">Save</button>
```

### `gotk-payload`

Hardcoded JSON payload.

```html
<button gotk-click="delete-todo" gotk-payload='{"id": 42}'>Delete</button>
```

### `gotk-val-*`

Attach payload values directly on elements. Cleaner than `gotk-payload` for simple
key-value pairs. All `gotk-val-*` attributes are collected into the payload. When
used alongside `gotk-collect`, values are merged with `gotk-val-*` taking
precedence.

```html
<button gotk-click="delete" gotk-val-id="42" gotk-val-type="user">Delete</button>
```

Produces payload: `{"id": "42", "type": "user"}`.

### `gotk-debounce`

Debounce interval in milliseconds. Applies to `gotk-input` and `gotk-on`.

```html
<input gotk-input="validate-email" gotk-debounce="300" name="email">
```

### `gotk-throttle`

Throttle interval in milliseconds. Unlike debounce, throttle fires at most once
per interval and guarantees the last event is delivered. Applies to `gotk-input`
and `gotk-on`.

```html
<div gotk-on="scroll:update-position" gotk-throttle="100">...</div>
```

### `gotk-loading`

Text to display while a server command is in flight. The thin client disables
the element and swaps its text content when the command fires, then restores the
original text and re-enables the element when the response arrives. Prevents
double submissions and provides immediate feedback.

```html
<button gotk-click="save-user" gotk-collect="#form" gotk-loading="Saving...">Save</button>
```

## Payload Collection

When a command fires, the thin client assembles the payload from multiple
sources. Sources are merged in the following priority order (highest wins):

1. **`gotk-val-*` attributes** — Explicit key-value pairs on the element.
2. **`gotk-collect` container** — Named elements within the referenced container.
3. **`gotk-payload` JSON** — Hardcoded JSON object.

When sources overlap on a key, `gotk-val-*` wins over `gotk-collect`, which wins
over `gotk-payload`. In practice, most commands use only one source.

## Command Types

### Server Commands

Handled by Go functions on the server. Transported via WebSocket. Have access to
databases, sessions, and server-side state.

```go
func (app *App) CreateTodo(ctx *gotk.Context) error {
    title := ctx.Payload.String("title")
    todo := app.db.Create(title)
    html := ctx.Render("partials/todo-item", todo)
    ctx.HTML("#todo-list", html, gotk.Append)
    ctx.Dispatch("#new-todo", "reset")
    ctx.Focus("#title-input")
    return nil
}
```

Registration:

```go
mux := gotk.NewMux()
mux.Handle("create-todo", app.CreateTodo)
```

### Frontend Commands

Handled by Go functions compiled to WASM via TinyGo. Run locally in the browser.
No server round-trip. Used for UI-only concerns: toggling visibility, opening
modals, managing UI state.

```go
func CloseModal(ctx *gotk.Context) error {
    ctx.AttrSet("#modal", "hidden", "")
    ctx.HTML("#modal", "", gotk.Replace)
    return nil
}
```

Frontend commands use the same `gotk.Context` interface and produce the same
instruction types as server commands.

### Async Chaining

A frontend command can schedule a server command via `ctx.Async(cmd, payload)`.
The framework applies the frontend command's instructions immediately, then
sends the async command over WebSocket. When the server responds, its
instructions are applied.

This enables patterns where the UI updates instantly and data loads
progressively.

```go
func OpenUserModal(ctx *gotk.Context) error {
    ctx.Template("#tpl-user-form", "#modal")
    ctx.AttrRemove("#modal", "hidden")

    if id := ctx.Payload.Int("id"); id > 0 {
        ctx.HTML("#modal-title", "Edit User")
        ctx.AttrSet("#modal-body", "aria-busy", "true")
        ctx.Async("edit-user", map[string]any{"id": id})
    } else {
        ctx.HTML("#modal-title", "New User")
    }

    return nil
}
```

The server command fulfills the async call:

```go
func (app *App) EditUser(ctx *gotk.Context) error {
    user := app.db.GetUser(ctx.Payload.Int("id"))
    ctx.Populate("#modal-body", map[string]any{
        "name":  user.Name,
        "email": user.Email,
    })
    ctx.AttrRemove("#modal-body", "aria-busy")
    return nil
}
```

Timeline for the edit case:

```
click
  └─ OpenUserModal (WASM, instant)
       ├─ template  → clone form into #modal
       ├─ attr-remove → unhide modal
       ├─ html → set title "Edit User"
       ├─ attr-set → aria-busy on #modal-body
       └─ async → "edit-user" {id: 42}     → sent over WS
                    │
                  EditUser (server)
                    ├─ populate → fill form fields
                    └─ attr-remove → clear aria-busy
```

## Error Handling

### Server Command Errors

When a server command encounters an error, it should produce instructions that
surface the error in the UI. The `ctx.Error()` convenience method generates a
standard `html` instruction with a `gotk-error` CSS class:

```go
func (app *App) EditUser(ctx *gotk.Context) error {
    user, err := app.db.GetUser(ctx.Payload.Int("id"))
    if err != nil {
        ctx.Error("#modal-body", "Failed to load user")
        ctx.AttrRemove("#modal-body", "aria-busy")
        return nil
    }
    ctx.Populate("#modal-body", map[string]any{
        "name":  user.Name,
        "email": user.Email,
    })
    ctx.AttrRemove("#modal-body", "aria-busy")
    return nil
}
```

`ctx.Error(target, message)` produces:

```json
{
  "op": "html",
  "target": "#modal-body",
  "html": "<div class=\"gotk-error\">Failed to load user</div>"
}
```

This is not a new instruction type — it is a convenience method that produces a
standard `html` instruction. Applications style `.gotk-error` as needed.

### Async Errors

When a server command called via `ctx.Async()` fails, the error instructions are
applied to the DOM like any other response. The frontend command does not need to
handle the error — the server command decides how to surface it. This keeps the
error handling close to the code that knows what went wrong.

## Context Convenience Methods

The `gotk.Context` provides convenience methods that produce standard
instructions. These methods exist to reduce boilerplate and enforce consistent
patterns. None of them introduce new instruction types.

| Method | Produces | Description |
| --- | --- | --- |
| `ctx.HTML(target, html, mode?)` | `html` | Set/append/prepend HTML content. |
| `ctx.Remove(target)` | `html` (mode: remove) | Remove an element from the DOM. |
| `ctx.Template(source, target)` | `template` | Clone a `<template>` into a target. |
| `ctx.Populate(target, data)` | `populate` | Fill named form elements from a map. |
| `ctx.Navigate(url)` | `navigate` | Push a URL to browser history. |
| `ctx.AttrSet(target, attr, value)` | `attr-set` | Set an attribute. |
| `ctx.AttrRemove(target, attr)` | `attr-remove` | Remove an attribute. |
| `ctx.SetValue(target, value)` | `set-value` | Set a form element value. |
| `ctx.Dispatch(target, event, detail?)` | `dispatch` | Fire a CustomEvent. |
| `ctx.Focus(target)` | `focus` | Focus an element. |
| `ctx.Exec(name, args?)` | `exec` | Call a registered JS function. |
| `ctx.Async(cmd, payload)` | `cmd` | Schedule a server command. |
| `ctx.Error(target, message)` | `html` | Insert error markup with `gotk-error` class. |
| `ctx.Render(template, data)` | (returns string) | Render a Go template to HTML string. |
| `ctx.JSON(data)` | (returns data) | Return structured data (for Async consumers). |

## Server-Side Rendering and Routing

### Initial Page Load

Every URL has a standard HTTP handler that returns a full server-rendered page.
This ensures pages are bookmarkable and work without JavaScript on first load.

```go
router.GET("/todos", func(w http.ResponseWriter, r *http.Request) {
    todos := app.db.ListTodos()
    gotk.RenderPage(w, "layouts/app", "pages/todos", todos)
})
```

The rendered page includes the thin client JS. On load, it establishes the
WebSocket connection and scans the DOM for `gotk-*` attributes.

### Client-Side Navigation

Links with `gotk-navigate` intercept clicks and send a `navigate` command over
WebSocket. The server returns instructions (typically `html` + `navigate`) to
update the page content and push the URL.

### Back/Forward

The thin client listens for `popstate` events and sends a navigate command:

```json
{ "cmd": "navigate", "payload": { "url": "/todos" } }
```

The server renders the appropriate content and returns it.

### Template Reuse

The same Go templates are used for both SSR full-page renders and WebSocket
fragment responses. No duplication.

## Connection Management

### WebSocket Lifecycle

The thin client manages the WebSocket connection automatically, including
reconnection after network interruptions, server restarts, or laptop sleep/wake.

### Reconnection Strategy

On disconnect, the thin client:

1. Applies the `gotk-disconnected` state (see below).
2. Attempts to reconnect with exponential backoff: immediate, 2s, 5s, 10s,
   capped at 30s.
3. On successful reconnect, sends a `navigate` command for the current URL to
   re-sync the page state with the server.
4. Applies the `gotk-connected` state.

### Connection State CSS

The thin client toggles CSS classes on `<body>` to reflect connection state:

- `gotk-connected` — WebSocket is open.
- `gotk-disconnected` — WebSocket is closed, reconnecting.

Applications can use these to provide visual feedback:

```css
body.gotk-disconnected #main { opacity: 0.5; pointer-events: none; }
body.gotk-disconnected .gotk-offline-banner { display: flex; }
```

```html
<div class="gotk-offline-banner" style="display: none;">
  Reconnecting...
</div>
```

## Thin Client Behavior

Pseudocode for the thin client:

```
on page load:
  init WASM module
  localCmds = Set(wasm.listCommands())   // one-time call, cached as a JS Set
  connect WebSocket
  set body.gotk-connected
  scan DOM for gotk-* attributes, bind handlers

on WS close:
  set body.gotk-disconnected, remove body.gotk-connected
  schedule reconnect with exponential backoff

on WS reconnect:
  set body.gotk-connected, remove body.gotk-disconnected
  send {cmd: "navigate", payload: {url: location.pathname}}

on gotk-click / gotk-input / gotk-on:
  collect payload from gotk-collect / gotk-val-* / gotk-payload
  if localCmds.has(cmd):
    call WASM command with payload
    apply returned instructions
    if async calls exist, send them over WS
  else:
    if gotk-loading:
      disable element, swap text to gotk-loading value
      store ref for restoration on response
    send {cmd, payload, ref} over WS

on WS message:
  if ref matches a gotk-loading element:
    restore original text, re-enable element
  for each instruction in ins:
    html       → set innerHTML / append / prepend / remove
    template   → clone <template> content into target
    populate   → set values on named elements in container
    navigate   → history.pushState, optionally update content
    attr-set   → element.setAttribute
    attr-remove → element.removeAttribute
    set-value  → element.value = value
    dispatch   → element.dispatchEvent(new CustomEvent(...))
    focus      → element.focus()
    exec       → call registered JS function
    cmd        → send as new WS command
  re-scan new/changed DOM nodes for gotk-* attributes

on popstate:
  send {cmd: "navigate", payload: {url: location.pathname}}

on gotk-poll:
  setInterval → send command every gotk-every interval
```

After every `html` or `template` instruction, the thin client re-scans the
affected subtree for new `gotk-*` attributes. This ensures dynamically inserted
content is interactive.

## Project Structure

```
gotk/
  context.go         # Context interface — compiles under Go and TinyGo
  instructions.go    # Instruction types — compiles under Go and TinyGo
  server/
    mux.go           # Command routing and WebSocket handling
    ssr.go           # HTTP handlers and full-page rendering
  client/
    wasm.go          # WASM entry point, command registry for frontend commands
  testing/
    context.go       # TestContext with instruction inspection and async mocking
```

The root `gotk` package contains the shared interface. Commands that import only
`gotk` compile under both targets. Commands that import `gotk/server` (for DB
access, sessions) are server-only.

## Testability

### Server Commands

Unit tested with `go test`. Call the handler, inspect instructions.

```go
func TestCreateTodo(t *testing.T) {
    app := newTestApp()
    ctx := gotk.NewTestContext()
    ctx.SetPayload(map[string]any{"title": "Buy milk"})

    err := app.CreateTodo(ctx)
    require.NoError(t, err)

    ins := ctx.Instructions()
    assert.Equal(t, "html", ins[0].Op)
    assert.Equal(t, "#todo-list", ins[0].Target)
    assert.Contains(t, ins[0].HTML, "Buy milk")
}
```

### Frontend Commands

Unit tested with `go test`. Same pattern — call the handler, inspect
instructions and async calls.

```go
func TestOpenUserModal_Edit(t *testing.T) {
    ctx := gotk.NewTestContext()
    ctx.SetPayload(map[string]any{"id": 42})

    OpenUserModal(ctx)

    ins := ctx.Instructions()
    assert.Equal(t, "template", ins[0].Op)
    assert.Equal(t, "Edit User", ins[2].HTML)
    assert.Equal(t, "attr-set", ins[3].Op)

    async := ctx.AsyncCalls()
    assert.Len(t, async, 1)
    assert.Equal(t, "edit-user", async[0].Cmd)
    assert.Equal(t, 42, async[0].Payload["id"])
}
```

### Templates

Go templates can be tested independently by rendering with test data and
asserting on the HTML output.

### Integration

Send a WebSocket message, assert on the response JSON. No browser required.

### What Doesn't Need Testing

The thin client is ~200 lines of stable code. It maps instructions to DOM
operations. It is tested once and rarely changes. No Playwright or Selenium
required for the vast majority of application testing.

## Design Principles

1. **Explicit over implicit.** Commands explicitly declare what to change in the
   DOM via instructions. There is no automatic re-rendering, no diffing, no
   virtual DOM. The developer controls what updates and when.

2. **Server is the source of truth for data.** Business state lives in
   databases, accessed by server commands. The DOM holds only UI state (what's
   visible, what's focused, what's loading).

3. **Single language.** Go for server commands, Go (via TinyGo/WASM) for
   frontend commands, Go templates for HTML rendering. `go test` covers all
   layers except the thin client.

4. **Commands are just functions.** A command takes a context and returns an
   error. It produces instructions by calling methods on the context. This makes
   commands trivially testable — call the function, inspect the context.

5. **The thin client is stable infrastructure.** It should rarely change. All
   application behavior is expressed through commands and instructions, not
   client-side code.

6. **Progressive enhancement of interactivity.** Initial page load is full SSR
   over HTTP. WebSocket adds real-time interactivity. WASM adds instant frontend
   commands. Each layer is optional — an app could work with SSR + WS alone.

## Summary Table

| Concern              | Where it runs | How it's tested        |
| -------------------- | ------------- | ---------------------- |
| Business logic       | Server (Go)   | `go test`, unit        |
| Data access          | Server (Go)   | `go test`, unit        |
| HTML rendering       | Server (Go templates) | `go test`, render + assert |
| UI state (modals, toggles) | Browser (WASM) | `go test`, unit   |
| Event binding        | Browser (thin client) | Manual / stable   |
| Instruction application | Browser (thin client) | Manual / stable |

## AI Agent Interface

### Why the Command Layer Is Not Just Another REST API

A common question: if commands are structured JSON in/out over WebSocket, how is
this different from building a REST API?

For a pure backend-to-backend integration, it is not meaningfully different. A
REST API would serve just as well. The difference is: **you don't build one.**

With a typical web app, you build the UI layer AND a separate REST API, then
maintain both, keep them in sync, and test both. With gotk, the command layer
that serves the UI *is* the programmatic interface. There is nothing extra to
build, document, or maintain.

The second difference is **contextual discovery**. A REST API gives you a flat
list of endpoints (via OpenAPI, Swagger, etc.). The gotk HTML gives you
state-aware actions in context:

```html
<!-- This only appears when the user has items -->
<button gotk-click="delete-todo" gotk-val-id="42">Delete</button>

<!-- This shows what fields are needed, with current values -->
<input name="name" value="Alice">
<input name="email" value="alice@test.com">
<button gotk-click="save-user" gotk-collect="#user-form">Save</button>
```

An agent reading this page knows: "right now, I can delete todo 42, and I can
save a user whose current name is Alice." A REST API endpoint like
`DELETE /todos/:id` tells you the shape but not the current state.

### How an AI Agent Uses gotk

An agent does not need a browser. It connects via WebSocket and sends commands
directly:

```json
{"cmd": "create-todo", "payload": {"title": "Buy milk"}, "ref": "a1"}
```

It receives structured instructions in response:

```json
{"ref": "a1", "ins": [{"op": "html", "target": "#todo-list", "html": "<li>Buy milk</li>", "mode": "append"}]}
```

No screenshots, no DOM parsing, no simulated clicks. The agent can also read the
HTML fragments in `html` instructions to understand what changed in
human-readable terms.

### Agent Interaction Levels

An agent can choose its level of interaction depending on capability:

| Level | What the agent does | Requires |
| --- | --- | --- |
| Command-only | Sends commands, reads instruction JSON | WebSocket connection |
| HTML-aware | Reads SSR pages to discover available commands and current state | HTTP GET + WebSocket |
| Full context | Parses `gotk-*` attributes, `name` fields, and page structure | HTML parsing |

All three levels use the same command protocol. No separate API surface needed.

### The Self-Describing UI

The `gotk-*` attributes on HTML elements serve as a machine-readable description
of the UI's capabilities. An agent that can read HTML understands:

- `gotk-click="create-user"` — "I can create a user by triggering this command."
- `gotk-collect="#user-form"` — "I need to provide data from these fields."
- `<input name="email">` — "email is a required/available input."

The HTML IS the API documentation. The rendered page shows what actions are
available, what inputs they expect, and what the current state is.

## Implementation Details

This section defines the exact Go types, conventions, and behaviors needed to
implement the framework without ambiguity.

### Go Types

#### Handler

```go
// HandlerFunc is the signature for all command handlers — server and frontend.
type HandlerFunc func(ctx *Context) error
```

#### Payload

```go
// Payload wraps the command payload with typed accessors.
// All accessors parse from the underlying map[string]any.
// String values from the DOM (e.g., gotk-val-id="42") are coerced:
// Int/Float parse strings via strconv. Bool treats "true"/"1" as true.
// Missing keys return zero values (no error).
type Payload struct {
    data map[string]any
}

func (p Payload) String(key string) string
func (p Payload) Int(key string) int
func (p Payload) Float(key string) float64
func (p Payload) Bool(key string) bool
func (p Payload) Map() map[string]any  // returns the raw map
```

#### Instruction

```go
// Instruction is a single DOM operation. All fields are optional except Op.
// Only the fields relevant to each Op are serialized to JSON.
type Instruction struct {
    Op      string         `json:"op"`
    Target  string         `json:"target,omitempty"`
    HTML    string         `json:"html,omitempty"`
    Mode    string         `json:"mode,omitempty"`     // replace (default), append, prepend, remove
    Source  string         `json:"source,omitempty"`   // template source selector
    Attr    string         `json:"attr,omitempty"`
    Value   string         `json:"value,omitempty"`
    Event   string         `json:"event,omitempty"`
    Detail  map[string]any `json:"detail,omitempty"`
    URL     string         `json:"url,omitempty"`
    Name    string         `json:"name,omitempty"`     // exec function name
    Args    map[string]any `json:"args,omitempty"`
    Cmd     string         `json:"cmd,omitempty"`      // async command name
    Payload map[string]any `json:"payload,omitempty"`
    Data    map[string]any `json:"data,omitempty"`     // populate data
}
```

A single struct with optional fields, not separate types per op. This keeps
JSON serialization simple and allows the thin client to use a single switch on
`op`.

#### Context

```go
// Context is passed to every command handler. It provides access to the
// command payload and methods to produce instructions.
// Both the server implementation and TestContext implement this as a concrete
// struct (not an interface) with the same method set. The server Context wraps
// a WebSocket connection. The TestContext stores instructions in a slice.
type Context struct {
    Payload Payload
    // internal: instructions []Instruction, asyncCalls []AsyncCall
}

// Instruction producers — each appends to the internal instructions slice.
func (c *Context) HTML(target, html string, mode ...string)
func (c *Context) Remove(target string)
func (c *Context) Template(source, target string)
func (c *Context) Populate(target string, data map[string]any)
func (c *Context) Navigate(url string, targetAndHTML ...string)
func (c *Context) AttrSet(target, attr string, value ...string)
func (c *Context) AttrRemove(target, attr string)
func (c *Context) SetValue(target, value string)
func (c *Context) Dispatch(target, event string, detail ...map[string]any)
func (c *Context) Focus(target string)
func (c *Context) Exec(name string, args ...map[string]any)
func (c *Context) Async(cmd string, payload map[string]any)
func (c *Context) Error(target, message string)

// Template rendering — delegates to the registered template engine.
func (c *Context) Render(name string, data any) string

// JSON — returns structured data. Used when the server command is called
// via ctx.Async and the caller wants data instead of instructions.
// Sets the response data on the context; the framework includes it in
// the response alongside any instructions.
func (c *Context) JSON(data map[string]any)
```

#### TestContext

```go
// TestContext is used in tests. Same method set as Context.
// Provides inspection methods not available on the server Context.
type TestContext struct {
    Context // embeds Context
}

func NewTestContext() *TestContext
func (tc *TestContext) SetPayload(data map[string]any)
func (tc *TestContext) Instructions() []Instruction
func (tc *TestContext) AsyncCalls() []AsyncCall

type AsyncCall struct {
    Cmd     string
    Payload map[string]any
}
```

#### Mux

```go
// Mux routes commands to handlers.
type Mux struct { /* internal */ }

func NewMux() *Mux
func (m *Mux) Handle(name string, handler HandlerFunc)

// ServeWebSocket upgrades an HTTP request to a WebSocket connection
// and starts reading commands / writing instruction responses.
func (m *Mux) ServeWebSocket(w http.ResponseWriter, r *http.Request)
```

### HTML Mode `replace` — innerHTML

The `html` instruction with `mode: "replace"` (default) sets the **innerHTML**
of the target element, not outerHTML. The target element itself is preserved.
This means the element's attributes, event bindings, and identity remain
stable. To remove the element entirely, use `mode: "remove"`.

### Template Loading and Rendering

Templates are loaded using Go's standard `template.ParseGlob` or
`template.ParseFS` at application startup. The developer registers a
`*template.Template` with the framework:

```go
tmpl := template.Must(template.ParseGlob("templates/**/*.html"))
mux := gotk.NewMux()
mux.SetTemplates(tmpl)
```

`ctx.Render("partials/todo-item", data)` calls `tmpl.ExecuteTemplate` with the
given name and data, returning the rendered HTML string.

`gotk.RenderPage(w, "layouts/app", "pages/todos", data)` renders a full page
by executing the layout template. The layout uses Go's `{{template "content" .}}`
or `{{block "content" .}}` to include the page template. The thin client
`<script>` tag is injected automatically at the end of `<body>` by the
framework.

### Thin Client Serving

The thin client JS is embedded in the Go binary via `//go:embed`. The
framework registers an HTTP handler at `/gotk/client.js` that serves it.
`gotk.RenderPage` automatically includes `<script src="/gotk/client.js"></script>`
at the end of `<body>`. The WASM binary, if used, is served similarly at
`/gotk/app.wasm`.

### `ref` Generation

The thin client generates `ref` values using an incrementing integer counter
starting at 1. Simple, unique within a connection lifetime, and small on the
wire. Format: `"1"`, `"2"`, `"3"`, etc.

### `gotk-navigate` — Built-in Command

`navigate` is a framework-provided command registered automatically on the mux.
When received, it calls the application's router to determine which page to
render for the given URL, renders the page content (not the full layout), and
returns `html` + `navigate` instructions.

The developer registers a navigate handler that maps URLs to content:

```go
mux.HandleNavigate(func(ctx *gotk.Context, url string) error {
    // Route the URL and render the appropriate content.
    // Return instructions to update the page.
    switch url {
    case "/todos":
        todos := app.db.ListTodos()
        html := ctx.Render("pages/todos", todos)
        ctx.HTML("#content", html)
        ctx.Navigate(url)
    }
    return nil
})
```

### Element Not Found

When an instruction's `target` (or `source`) selector matches no element, the
thin client logs a warning to the console and skips the instruction. It does
not throw, halt instruction processing, or send an error back to the server.
Remaining instructions in the batch continue to execute.

### `gotk-collect` — Collection Rules

The thin client collects values from all elements with a `name` attribute
within the `gotk-collect` container:

| Element type | How value is read |
| --- | --- |
| `<input type="text">`, `<textarea>` | `.value` |
| `<input type="number">` | `.value` (string — Payload accessors coerce) |
| `<input type="checkbox">` | `.checked` (boolean) |
| `<input type="radio">` | `.value` of the checked radio in the group |
| `<select>` | `.value` (selected option's value) |
| `<select multiple>` | Array of `.value` for all selected options |
| `[contenteditable]` | `.innerText` |
| Elements with `gotk-val-*` | Collected via dataset, not `.value` |

**Duplicate names:** If multiple elements share the same `name` (e.g.,
checkboxes), values are collected as an array.

### `gotk-every` Format

Accepts a number followed by a unit suffix:

- `ms` — milliseconds: `500ms`
- `s` — seconds: `30s`
- `m` — minutes: `5m`

Plain numbers without suffix are treated as milliseconds: `5000` = 5 seconds.
Minimum interval: `1000ms` (1 second). The thin client clamps lower values.

### `gotk-poll` Cleanup

When an `html` or `template` instruction replaces a subtree that contains
`gotk-poll` elements, the thin client clears the associated intervals before
replacing the content. The re-scan logic works in two phases:

1. **Teardown:** Before applying `html` (replace mode) or `template`, collect
   all `gotk-poll` elements within the target and clear their intervals.
2. **Setup:** After applying the instruction, scan the new subtree for `gotk-*`
   attributes and bind handlers (including new `gotk-poll` intervals).

This also applies to `gotk-on` and `gotk-input` event listeners.

### Local Routing for All Event Types

Automatic local/server routing (check `localCmds.has(cmd)`) applies to all
command-triggering attributes: `gotk-click`, `gotk-input`, `gotk-on`, and
`gotk-poll`. The routing logic is the same for all — WASM first, WebSocket
fallback.

### `gotk-loading` and Local Commands

`gotk-loading` only applies to commands routed to the server (WebSocket). For
local commands (WASM), the command executes synchronously and instructions are
applied immediately — there is no "in flight" period.

When a local command uses `ctx.Async`, the `gotk-loading` state is **not**
applied. The local command should use instructions (e.g., `ctx.AttrSet` on the
element) to manage its own loading UI if needed.

### `exec` Function Registration

Client-side JS functions for the `exec` instruction are registered via a
global registry on the thin client:

```html
<script>
  gotk.register("lockScroll", function(args) {
    document.body.style.overflow = "hidden";
  });

  gotk.register("unlockScroll", function(args) {
    document.body.style.overflow = "";
  });
</script>
```

`gotk.register(name, fn)` adds the function to an internal map. The `exec`
instruction calls `registeredFns[name](args)`. Unknown function names log a
console warning and are skipped.

### WASM ↔ JS Bridge

The WASM module exports two functions accessible from JS:

```js
// Called once at init. Returns a JSON string: ["cmd-a", "cmd-b", ...]
const cmdsJSON = wasmInstance.exports.listCommands();
const localCmds = new Set(JSON.parse(cmdsJSON));

// Called per command. Takes command name + payload JSON string.
// Returns a JSON string: { "ins": [...], "async": [...] }
const resultJSON = wasmInstance.exports.execCommand(cmdName, payloadJSON);
const result = JSON.parse(resultJSON);
```

Communication is via JSON strings across the WASM boundary. The thin client
deserializes the result and processes `ins` (instructions to apply) and `async`
(commands to send over WebSocket) separately.

### Session and Authentication

WebSocket connections are authenticated via the HTTP upgrade request. The
server's WebSocket handler has access to the standard `http.Request`, including
cookies and headers. The framework does not impose an authentication mechanism
— the developer uses middleware or checks credentials in the upgrade handler:

```go
router.GET("/ws", authMiddleware(mux.ServeWebSocket))
```

Each WebSocket connection is independent. The framework does not provide
cross-connection state or session storage — the developer uses their existing
session mechanism (cookies, JWTs, database sessions).

### Server Push

The server can push instructions to a specific connection at any time. The mux
provides a connection reference that handlers can store:

```go
mux.HandleConnect(func(conn *gotk.Conn) {
    // Store conn for later push
    app.connections[conn.ID()] = conn
})

mux.HandleDisconnect(func(conn *gotk.Conn) {
    delete(app.connections, conn.ID())
})

// Later, push instructions to a specific connection:
conn.Push([]gotk.Instruction{
    {Op: "html", Target: "#notifications", HTML: "<li>New message</li>", Mode: "append"},
})
```

For broadcast (push to all connections), the developer iterates their
connection map. The framework does not provide built-in pub/sub — this is
application-level logic.

### Error Response for Unknown Commands

When the mux receives a command name that has no registered handler, it
responds with an instruction sequence containing an error:

```json
{
  "ref": "a1b2",
  "ins": [{"op": "exec", "name": "console.warn", "args": {"message": "unknown command: foo"}}],
  "error": "unknown command: foo"
}
```

The `error` field is a top-level string on the response. The thin client logs
it to the console. No DOM changes are made unless the developer registers an
error handler.

### Concurrent Commands

Multiple commands can be in flight simultaneously. Each has a unique `ref`.
Responses may arrive in any order — the thin client uses `ref` to correlate
responses to their originating elements (for `gotk-loading` restoration). The
server mux handles commands sequentially per connection (one goroutine per
connection reads messages, dispatches handlers). If a handler is slow, it
blocks subsequent commands on that connection. Long-running handlers should use
goroutines internally.

### Re-scan Scope

After an `html` (any mode) or `template` instruction, the thin client re-scans
**only the target element and its subtree** for new `gotk-*` attributes. After
`populate` or `set-value`, no re-scan occurs (values changed, not structure).

### `ctx.JSON` Usage

`ctx.JSON` is reserved for the future callback/continuation pattern described
in the Future Considerations section (JSON-First Server Commands). In the
current design, it is not used — server commands called via `ctx.Async` return
instructions, not raw data. It is included in the Context interface for forward
compatibility but should not be used in Phase 1–4 implementations.

## Implementation Plan

The plan is structured so that each phase produces a usable system and an
existing Go web application can migrate incrementally — one page, one
interaction at a time — without rewriting anything upfront.

### Phase 1 — Core: Context, Instructions, and TestContext

**Goal:** The command/instruction model exists as Go code. Commands can be
written and tested. Nothing runs in a browser yet.

**Build:**

- `gotk.Context` interface — `Payload`, `HTML()`, `AttrSet()`, `AttrRemove()`,
  `Navigate()`, `Focus()`, `Dispatch()`, `Template()`, `Populate()`, `Exec()`,
  `Error()`, `Render()`.
- Instruction types — Go structs for each instruction op.
- `gotk.TestContext` — captures instructions and async calls for assertion in
  tests.

**What works after this phase:**

```go
func (app *App) CreateTodo(ctx *gotk.Context) error {
    title := ctx.Payload.String("title")
    todo := app.db.Create(title)
    html := ctx.Render("partials/todo-item", todo)
    ctx.HTML("#todo-list", html, gotk.Append)
    return nil
}

func TestCreateTodo(t *testing.T) {
    app := newTestApp()
    ctx := gotk.NewTestContext()
    ctx.SetPayload(map[string]any{"title": "Buy milk"})

    app.CreateTodo(ctx)

    ins := ctx.Instructions()
    assert.Equal(t, "html", ins[0].Op)
    assert.Contains(t, ins[0].HTML, "Buy milk")
}
```

Commands are plain Go functions. Tests run with `go test`. No browser, no
WebSocket, no frontend. An existing app can start writing commands alongside
its current handlers immediately.

**Migration step:** Write commands for existing interactions. Test them. Don't
wire them to anything yet.

### Phase 2 — Server: Mux, WebSocket, and Thin Client

**Goal:** Commands run on the server, triggered from the browser via WebSocket.
The thin JS client handles the `gotk-*` attributes and applies instructions.

**Build:**

- `gotk.Mux` — command router. `mux.Handle(name, handler)`.
- WebSocket handler — accepts connections, reads command JSON, routes to
  handlers, writes instruction JSON back.
- Thin client JS (~200 LOC) — connects WebSocket, scans DOM for `gotk-*`
  attributes, collects payloads, sends commands, applies instructions,
  re-scans after DOM changes.
- SSR helpers — `gotk.RenderPage()` for full-page renders. Includes the thin
  client script automatically.
- `gotk-click`, `gotk-collect`, `gotk-payload`, `gotk-val-*` — core attribute
  set.
- `gotk-navigate` — client-side navigation via WS commands.
- `popstate` handling — back/forward sends a navigate command.
- Connection management — reconnection with exponential backoff,
  `gotk-connected` / `gotk-disconnected` CSS classes on `<body>`.

**What works after this phase:**

A fully functional server-rendered web app with WebSocket-driven interactions.
Pages load via HTTP (SSR). User interactions fire commands via WebSocket.
Server commands return instructions that update the DOM. Pages are
bookmarkable.

```html
<!-- Add to an existing page -->
<script src="/gotk/client.js"></script>

<!-- Add gotk-click to a button that previously used JS or a form -->
<button gotk-click="delete-todo" gotk-val-id="42">Delete</button>
```

```go
// Existing HTTP handler stays for SSR
router.GET("/todos", app.TodosPage)

// New: register a command
mux.Handle("delete-todo", app.DeleteTodo)

// Mount WebSocket alongside existing routes
router.GET("/ws", mux.ServeWebSocket)
```

**Migration step:** Add the thin client script to your layout. Pick one
interaction (a button, a form submission), add `gotk-click`, write a server
command. The rest of the page continues to work as before. Migrate one
interaction at a time.

### Phase 3 — Interactivity: Loading States, Polling, Events

**Goal:** The attribute set is complete. The thin client handles all the
interaction patterns needed for a production app.

**Build:**

- `gotk-loading` — disable element + swap text while command is in flight,
  restore on response.
- `gotk-debounce` / `gotk-throttle` — rate limiting for input and event
  commands.
- `gotk-input` — send command on input events.
- `gotk-on` — send command on arbitrary DOM events (`event:command` format).
- `gotk-poll` / `gotk-every` — periodic command execution.
- `template` instruction — clone `<template>` elements into targets.
- `populate` instruction — fill named form elements from a data map.

**What works after this phase:**

Real-time updates (polling), search-as-you-type (debounced input), drag-and-drop
reordering (custom events), optimistic UI (loading states). All driven by
server commands.

**Migration step:** Replace remaining frontend JS with `gotk-*` attributes.
Polling replaces `setInterval` + `fetch`. Input events replace `addEventListener`
\+ `fetch`. Loading states replace manual button disabling.

### Phase 4 — Frontend Commands: WASM

**Goal:** UI-only logic (modals, toggles, conditional UI) runs locally in the
browser via TinyGo-compiled WASM, using the same Context/instruction pattern.

**Build:**

- WASM entry point — `client/wasm.go`, exports `listCommands()` and a command
  dispatcher.
- `wasm.listCommands()` — called once at init, returns the list of locally
  registered commands. Thin client caches as a JS `Set`.
- Automatic command routing — `gotk-click` checks `localCmds.has(cmd)` before
  sending over WebSocket.
- `ctx.Async(cmd, payload)` — frontend command schedules a server command.
  Thin client applies frontend instructions immediately, sends async command
  over WS, applies server response when it arrives.
- TinyGo build integration — Makefile or build script that compiles
  `client/wasm.go` to `gotk.wasm`.

**What works after this phase:**

Modals open instantly (WASM command clones a `<template>`, sends async to load
data). Sidebar toggles are local. UI state decisions (which template to show,
whether to show a loading state) happen without a server round-trip.

```go
func OpenUserModal(ctx *gotk.Context) error {
    ctx.Template("#tpl-user-form", "#modal")
    ctx.AttrRemove("#modal", "hidden")

    if id := ctx.Payload.Int("id"); id > 0 {
        ctx.HTML("#modal-title", "Edit User")
        ctx.AttrSet("#modal-body", "aria-busy", "true")
        ctx.Async("edit-user", map[string]any{"id": id})
    } else {
        ctx.HTML("#modal-title", "New User")
    }
    return nil
}
```

**Migration step:** Identify interactions that don't need server data (toggle
sidebar, close modal, switch tabs). Write them as WASM commands. They
automatically route locally — no HTML changes needed beyond what was done in
Phase 2. For interactions that show a modal then load data, use `ctx.Async` to
chain a server command.

### Phase 5 — AI Agent Interface

**Goal:** AI agents can discover and use the app programmatically via the same
command protocol.

**Build:**

- `GET /gotk/commands` — JSON endpoint listing all registered commands with
  optional metadata (description, payload schema).
- `gotk.CmdMeta` — optional metadata struct: description, expected payload
  fields, types, required flags.
- `mux.Handle` accepts optional `CmdMeta` for documentation.
- Agent documentation in SSR pages — `gotk-*` attributes already serve as
  self-describing UI. No additional work needed for HTML-aware agents.

**What works after this phase:**

An AI agent connects via WebSocket, fetches the command list, and sends
commands — no browser needed. The same interface serves human users and agents.

**Migration step:** Add `CmdMeta` to existing commands. This is optional
metadata — commands continue to work without it. Add it progressively as
needed.

### Phase Summary

| Phase | What it delivers | Depends on | Migration effort |
| --- | --- | --- | --- |
| 1 — Core | Context, instructions, TestContext | Nothing | Write commands alongside existing code, test with `go test` |
| 2 — Server | Mux, WebSocket, thin client, SSR | Phase 1 | Add script tag + `gotk-click` to one page at a time |
| 3 — Interactivity | Loading, polling, events, debounce | Phase 2 | Replace remaining frontend JS with `gotk-*` attributes |
| 4 — WASM | Frontend commands, async chaining | Phase 2 | Move UI-only logic to WASM commands, no HTML changes |
| 5 — AI Agent | Command discovery, metadata endpoint | Phase 2 | Add optional metadata to existing commands |

Phases 3, 4, and 5 are independent of each other and can be done in any order
or skipped entirely. Phase 2 alone delivers a fully functional system. Phase 1
alone delivers testable command logic.

---

## Future Considerations

### `gotk-do` — Inline Actions (Locality of Behavior)

**Status:** Needs further design work.

**Problem:** Simple UI operations like toggling a sidebar or closing a modal
currently require a WASM command — a Go function in a separate file. Looking at
the HTML, you see `gotk-click="toggle-sidebar"` but have no idea what it does
without finding the Go function. This violates the Locality of Behavior
principle: the behavior of a unit of code should be as obvious as possible by
looking only at that unit of code.

**Idea:** A `gotk-do` attribute for simple DOM manipulations executed directly by
the thin client, no WASM or server involved:

```html
<!-- Toggle an attribute -->
<button gotk-do="attr-toggle #sidebar hidden">☰</button>

<!-- Set an attribute (close modal) -->
<button gotk-do="attr-set #modal hidden">Close</button>

<!-- Remove an attribute (open modal) -->
<button gotk-do="attr-remove #modal hidden">Open</button>

<!-- Remove an element (dismiss notification) -->
<button gotk-do="remove #notification-5">Dismiss</button>

<!-- Toggle a CSS class -->
<button gotk-do="class-toggle #menu active">Toggle Menu</button>

<!-- Focus an element -->
<button gotk-do="focus #search-input">Search</button>
```

**Chaining** with `;`:

```html
<button gotk-do="attr-remove #modal hidden; focus #modal-title">Open</button>
```

**Combining** with server commands — `gotk-do` runs first (instant), then the
command fires:

```html
<button gotk-do="attr-set #save-btn disabled" gotk-click="save-user" gotk-collect="#form">Save</button>
```

**Supported actions** (intentionally small and fixed — not a scripting
language):

| Action | Syntax | Effect |
| --- | --- | --- |
| `attr-set` | `attr-set <target> <attr> [value]` | Set an attribute. |
| `attr-remove` | `attr-remove <target> <attr>` | Remove an attribute. |
| `attr-toggle` | `attr-toggle <target> <attr>` | Toggle an attribute. |
| `class-add` | `class-add <target> <class>` | Add a CSS class. |
| `class-remove` | `class-remove <target> <class>` | Remove a CSS class. |
| `class-toggle` | `class-toggle <target> <class>` | Toggle a CSS class. |
| `remove` | `remove <target>` | Remove element from DOM. |
| `focus` | `focus <target>` | Focus an element. |

No conditionals, no data access, no loops. If you need logic, use a WASM
command (via `gotk-click`).

**The LoB spectrum** this creates:

| Attribute | Where it runs | LoB | Use when |
| --- | --- | --- | --- |
| `gotk-do` | Thin client (inline) | Full | Mechanical DOM changes: show, hide, toggle, focus. |
| `gotk-click` (WASM) | WASM (Go function) | Partial | Conditional logic, state-dependent UI decisions. |
| `gotk-click` | Server (Go function) | Minimal | Business logic, data access, rendered HTML. |

**Open questions:**

- Should `gotk-do` support an event trigger other than click? e.g.,
  `gotk-do="..." gotk-on="mouseenter"`.
- Should `gotk-do` actions mirror the instruction names exactly, or use a
  simplified syntax?
- How does `gotk-do` interact with `gotk-loading`? Should inline actions run before
  or after the loading state is applied?
- Is the syntax `action target attr` clear enough, or would a structured format
  like `action(target, attr)` be less ambiguous?

### Component Pattern — Co-located Templates and Commands

**Status:** Usage pattern, not a framework feature. Only framework change
needed: `mux.Local()` / `mux.Server()` registration methods.

**Problem:** A command like `gotk-click="open-user-modal"` tells you a command
name, but not where to find the code. The template lives in one directory, the
frontend commands in another, the server commands in a third. Understanding the
full behavior of a UI element requires jumping between files.

**Idea:** Co-locate a component's template, frontend commands, and server
commands in a single Go package. Use a struct with interface dependencies so
everything compiles under both Go and TinyGo.

```
app/
  usermodal/
    template.html         # the HTML template
    usermodal.go          # struct, interface, all commands, Register()
    usermodal_test.go     # tests for ALL commands (frontend + server)
```

The struct uses interfaces for server-side dependencies (DB, etc.) so it
compiles under TinyGo without importing concrete implementations:

```go
package usermodal

import (
    _ "embed"
    "github.com/esnunes/gotk"
)

//go:embed template.html
var templateHTML string

type Store interface {
    GetUser(id int) (User, error)
    SaveUser(user User) error
}

type UserModal struct {
    Store Store // nil in WASM, real DB on server, mock in tests
}
```

All commands are methods on the struct — both frontend and server:

```go
// Frontend command — does not use m.Store
func (m *UserModal) Open(ctx *gotk.Context) error {
    ctx.Template("#tpl-user-form", "#modal")
    ctx.AttrRemove("#modal", "hidden")
    if id := ctx.Payload.Int("id"); id > 0 {
        ctx.HTML("#modal-title", "Edit User")
        ctx.AttrSet("#modal-body", "aria-busy", "true")
        ctx.Async("user-modal.edit", map[string]any{"id": id})
    } else {
        ctx.HTML("#modal-title", "New User")
    }
    return nil
}

// Server command — uses m.Store
func (m *UserModal) Edit(ctx *gotk.Context) error {
    user, err := m.Store.GetUser(ctx.Payload.Int("id"))
    if err != nil {
        ctx.Error("#modal-body", "Failed to load user")
        ctx.AttrRemove("#modal-body", "aria-busy")
        return nil
    }
    ctx.Populate("#modal-body", map[string]any{
        "name":  user.Name,
        "email": user.Email,
    })
    ctx.AttrRemove("#modal-body", "aria-busy")
    return nil
}
```

Registration declares which commands run where:

```go
func (m *UserModal) Register(mux *gotk.Mux) {
    mux.Template("user-modal", templateHTML)
    mux.Local("user-modal.open", m.Open)     // frontend (WASM)
    mux.Local("user-modal.close", m.Close)   // frontend (WASM)
    mux.Server("user-modal.edit", m.Edit)    // server (WS)
    mux.Server("user-modal.save", m.Save)    // server (WS)
}
```

The WASM mux ignores `Server()` calls. The server mux handles both. Concrete
dependencies are wired at the application level:

```go
// Server
modal := &usermodal.UserModal{Store: postgres.NewUserStore(db)}
modal.Register(serverMux)

// WASM
modal := &usermodal.UserModal{}
modal.Register(wasmMux)
```

HTML uses dot-namespaced commands to tie elements to their component:

```html
<button gotk-click="user-modal.open" gotk-val-id="42">Edit User</button>
<button gotk-click="user-modal.close">Cancel</button>
<button gotk-click="user-modal.save" gotk-collect="#modal-body">Save</button>
```

**What this is:** A convention for organizing code around UI components. The
struct, interfaces, `//go:embed`, and namespaced commands are all plain Go.

**What the framework provides:** `mux.Local()` and `mux.Server()` registration
methods, and `mux.Template()` for registering embedded HTML templates. Everything
else is a usage pattern.

**Open questions:**

- Should the framework provide a `gotk.Component` interface to formalize the
  pattern, or leave it as a convention?
- How does `mux.Template()` inject the `<template>` element into SSR pages?
  Does the layout have a designated slot, or does the framework append them to
  `<body>` automatically?

### JSON-First Server Commands with Frontend Rendering

**Status:** Blocked by TinyGo limitations. Needs alternative template engine
or different approach.

**Problem:** Server commands currently return pre-rendered HTML via `ctx.HTML`.
This means AI agents receive opaque HTML strings rather than structured data,
and the server must know the DOM structure of every UI fragment it produces.

**Idea:** Server commands default to returning JSON data. Templates are
available on the frontend and executed there to render dynamic changes. The
server becomes a pure data layer. On the backend, templates are still used for
SSR (initial page load). On the frontend, the same templates render dynamic
updates from command responses.

This would make the system more AI-agent-friendly (agents get structured data
by default) and cleanly separate data from presentation.

**Example — how it would work:**

```go
// Server command — returns data only
func (app *App) LoadUser(ctx *gotk.Context) error {
    user := app.db.GetUser(ctx.Payload.Int("id"))
    ctx.JSON(map[string]any{
        "name":  user.Name,
        "email": user.Email,
        "role":  user.Role,
    })
    return nil
}

// Frontend command — receives data, renders template locally
func OpenUserModal(ctx *gotk.Context) error {
    ctx.Template("#tpl-user-form", "#modal")
    ctx.AttrRemove("#modal", "hidden")

    if id := ctx.Payload.Int("id"); id > 0 {
        ctx.HTML("#modal-title", "Edit User")
        ctx.AttrSet("#modal-body", "aria-busy", "true")
        ctx.Async("load-user", map[string]any{"id": id}, func(data gotk.Data) {
            html := ctx.Render("partials/user-form-body", data.Map())
            ctx.HTML("#modal-body", html)
        })
    } else {
        ctx.HTML("#modal-title", "New User")
    }
    return nil
}
```

**Why it's blocked:** Go's `html/template` package relies heavily on
reflection, which TinyGo does not fully support. Templates with `{{if}}`,
`{{range}}`, and method calls cannot be executed in WASM under TinyGo. A simple
`populate` (find `[name]` elements, set values) works for flat forms but fails
for templates with conditionals, loops, or nested structures.

**Possible paths forward:**

- **Custom template engine for TinyGo.** Build a minimal template engine that
  works with `map[string]any` and avoids reflection. Supports `{{if}}`,
  `{{range}}`, and variable interpolation only. Compiles under both Go and
  TinyGo. Downside: two template syntaxes (standard `html/template` for SSR,
  custom for WASM), or migrate SSR to the custom engine too.

- **Use templ (code-generation-based templates).** The templ library generates
  plain Go code with no reflection, which compiles under TinyGo. However, this
  significantly increases complexity: the WASM binary carries all templates,
  frontend commands need callbacks to receive server data before rendering,
  there are two rendering paths to reason about (SSR vs WASM), and the
  architecture drifts toward a LiveView-style model that the current design
  intentionally avoids. Viable if frontend rendering becomes a hard
  requirement, but a substantial departure from the current simplicity.

- **Wait for TinyGo improvements.** TinyGo's reflection support is expanding.
  `html/template` may become viable in future versions. Downside: no timeline,
  and `html/template` pulls in a large dependency chain.

- **Use a `render` instruction.** A middle ground — the server command sends a
  `render` instruction with template ID + data. The thin client (JS) handles
  template execution using `<template>` elements and a simple JS-side rendering
  strategy. The Go command code stays clean, the thin client does the
  rendering, and WASM never needs to execute templates. Downside: the JS
  thin client grows in complexity, and `<template>` elements with `populate`
  only cover flat fills, not conditionals or loops.

- **Accept HTML from server for complex fragments.** Keep `ctx.HTML` for
  fragments that need conditionals/loops (server renders them). Use `populate`
  for simple data fills. Use `render` instruction for cases where a `<template>`
  + data map suffices. This hybrid approach is pragmatic but adds cognitive load
  (three rendering strategies).

**Current recommendation:** Keep server-rendered HTML (`ctx.HTML`) as the
default for now. The `populate` instruction handles the common case of filling
form fields with data. Revisit frontend template execution when TinyGo's
reflection support matures or if a lightweight custom template engine proves
worthwhile.
