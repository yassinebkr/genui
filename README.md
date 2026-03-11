# GenUI Protocol Specification

**Version:** 0.1.0 — Draft  
**Created:** March 11, 2026  
**Authors:** Yassine Benkhira
**License:** MIT  

---

## Abstract

GenUI is a streaming protocol for **agent-generated, self-contained user interfaces**. Unlike declarative rendering protocols where the agent drives every interaction, GenUI produces **standalone applications** — widgets that run client-side logic autonomously after creation. The agent is a programmer, not a puppeteer.

GenUI is designed for AI agents embedded in chat interfaces, but the protocol is transport-agnostic and can be carried over WebSocket, SSE, MCP tool results, or any ordered message stream.

## Design Philosophy

### Agent as Programmer

In most agent-to-UI protocols, the agent generates a declarative UI description, the client renders it, the user interacts, the interaction goes back to the agent, and the agent generates a new UI state. Every click is a round-trip.

GenUI inverts this model. The agent generates a **complete application**: structure (HTML template), style (scoped CSS), data (props/defaults), and behavior (client-side JavaScript). After creation, the widget operates independently. The user can drag cards, toggle checkboxes, sort lists, and filter data — all at native speed, with zero server involvement.

The agent can still read and write widget state via `patch` and `update` operations, but it doesn't need to be in the loop for UI interactions.

### Why This Matters

| Concern | Agent-drives-all | GenUI (standalone) |
|---------|------------------|--------------------|
| Interaction latency | 2–10s (LLM round-trip) | <5ms (local JS) |
| Cost per interaction | Tokens consumed | Zero |
| Determinism | LLM may hallucinate state changes | JS executes deterministically |
| Offline resilience | Dead on disconnect | Keeps working |
| Server load | O(users × clicks) | O(widget creations only) |
| User experience | Form submission circa 2005 | Native app feel |

## Protocol Overview

GenUI uses a unidirectional stream of **operations** (ops) from server to client. Each op is a self-contained JSON object. Ops are processed in order and applied to a persistent canvas state.

The protocol defines 8 operation types:

| Op | Purpose |
|----|---------|
| `upsert` | Create or fully replace a component |
| `patch` | Merge partial data into an existing component |
| `remove` | Remove a component by ID |
| `clear` | Remove all components |
| `define` | Register a custom widget type (Web Component) |
| `undefine` | Unregister a custom widget type |
| `layout` | Change the canvas layout mode |
| `move` | Reposition a component |

### Transport

GenUI is transport-agnostic. Ops are JSON objects delivered in an ordered stream. Supported transports include:

- **WebSocket** — real-time bidirectional (recommended)
- **SSE** — server-sent events for unidirectional streaming
- **MCP tool results** — ops embedded in `_canvas_ops` field of tool responses
- **Code fences** — ops embedded in LLM text output within ` ```genui ` fences (for non-tool-calling models)

### State Model

The client maintains a **canvas state**: an ordered list of active ops. State persists across reconnections. On reconnect, the server replays the current state.

Components are identified by unique string IDs. IDs should be meaningful (e.g., `"weather-paris"`, `"kanban-board"`, `"task-list"`).

## Operations

### `upsert` — Create or Replace

Creates a new component or fully replaces an existing one.

```json
{
  "op": "upsert",
  "id": "weather-paris",
  "type": "weather",
  "data": {
    "city": "Paris",
    "temp": 18,
    "condition": "Partly Cloudy",
    "icon": "⛅"
  }
}
```

**Fields:**
- `op` (string, required): `"upsert"`
- `id` (string, required): Unique component identifier
- `type` (string, required): Component type (from the component catalog or a custom `define`d type)
- `data` (object, required): Component-specific data
- `layout` (object, optional): Positioning hints (`zone`, `order`)

### `patch` — Partial Update

Merges partial data into an existing component. Only the specified fields are updated; all other fields retain their current values.

```json
{
  "op": "patch",
  "id": "weather-paris",
  "data": { "temp": 21, "condition": "Sunny" }
}
```

### `remove` — Remove Component

```json
{ "op": "remove", "id": "weather-paris" }
```

### `clear` — Remove All

```json
{ "op": "clear" }
```

### `define` — Register Custom Widget

Registers a new component type as a Web Component with Shadow DOM isolation. This is the core of GenUI's standalone widget model.

```json
{
  "op": "define",
  "id": "kanban-board",
  "component": {
    "html": "<div class=\"board\">{{#each columns}}<div class=\"col\" data-action=\"drop\" data-column=\"{{id}}\">...</div>{{/each}}</div>",
    "css": ".board { display: flex; gap: 1rem; }",
    "props": ["columns"],
    "defaults": { "columns": [] },
    "actions": [
      { "name": "dragstart", "emits": "card-drag" },
      { "name": "drop", "emits": "card-drop" }
    ],
    "js": "if (action === 'card-drop') { /* move card locally */ render(); return true; }"
  }
}
```

**Component fields:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `html` | string | yes | Mustache-like template (see [Template Syntax](#template-syntax)) |
| `css` | string | no | Scoped CSS (Shadow DOM, no style leaks) |
| `props` | string[] | no | Reactive property names |
| `defaults` | object | no | Default values for props |
| `actions` | Action[] | no | Interactive action declarations |
| `js` | string | no | Client-side action handler (see [Action Handlers](#action-handlers)) |

### `undefine` — Unregister Custom Widget

```json
{ "op": "undefine", "id": "kanban-board" }
```

Existing DOM instances remain visible but stop receiving updates.

### `layout` — Change Layout Mode

```json
{ "op": "layout", "mode": "dashboard" }
```

Modes: `auto`, `dashboard`, `focus`, `columns`, `rows`.

### `move` — Reposition Component

```json
{ "op": "move", "id": "weather-paris", "layout": { "zone": "sidebar", "order": 0 } }
```

## Template Syntax

Custom widget templates use a Mustache-like syntax:

| Syntax | Description |
|--------|-------------|
| `{{prop}}` | HTML-escaped variable binding |
| `{{{prop}}}` | Raw (unescaped) HTML binding |
| `{{#each items}}...{{/each}}` | Array iteration. Inside: `{{@index}}`, `{{@first}}`, `{{@last}}` |
| `{{#if prop}}...{{/if}}` | Conditional rendering (truthy check) |
| `{{#unless prop}}...{{/unless}}` | Inverse conditional |

Templates are compiled once at `define` time and re-evaluated on each `render()` call. The template compiler operates on the widget's data object — no external state access.

## Action System

GenUI uses a `data-action` attribute system for interactivity. Elements with `data-action="name"` become interactive.

### Built-in Action Types

| `data-action` | Behavior | Payload |
|----------------|----------|---------|
| `"dragstart"` | Element becomes draggable. Runtime auto-sets `draggable="true"`, adds visual `.dragging` class. | `dragId` from `data-card-id` or `data-item-id` |
| `"drop"` | Element becomes a drop zone. Runtime handles `dragover` (preventDefault) and `dragleave`. | `dragId` from dataTransfer + all `data-*` attributes |
| Any other value | Click handler | All `data-*` attributes as key-value pairs |

### Action Handlers (the `js` field)

The `js` field is a JavaScript function body executed client-side on every action. It receives 5 parameters:

```
function(action, payload, data, render, root) { ... }
```

| Parameter | Type | Description |
|-----------|------|-------------|
| `action` | string | The `emits` name from the action declaration (or the raw `data-action` value) |
| `payload` | object | All `data-*` attributes from the triggering element, plus `dragId` for DnD |
| `data` | object | The widget's current state. **Mutable** — modify in-place. |
| `render` | function | Call after modifying `data` to re-render the template. |
| `root` | ShadowRoot | The widget's Shadow DOM root (for direct DOM queries if needed). |

**Return value:**
- `return true` — Action handled locally. No server/agent involvement.
- `return false` or `undefined` — Action propagates to the agent via a `widget-action` event.

**Execution model:** The handler is compiled via `new Function()` and called **fresh per action**. Local variables do NOT persist between calls. To persist state across actions, store it on `data` (e.g., `data._draggedId = id`).

**Example — Kanban drag-and-drop:**

```javascript
// Dragstart: store card ID (already in payload.dragId via runtime)
if (action === 'card-drag') return true;

// Drop: move card between columns
if (action === 'card-drop') {
  const cols = ['todo', 'doing', 'done'];
  let card;
  for (const c of cols) {
    const i = (data[c] || []).findIndex(x => x.id === payload.dragId);
    if (i >= 0) { card = data[c].splice(i, 1)[0]; break; }
  }
  if (card && payload.column) {
    data[payload.column].push(card);
    render();
  }
  return true;
}
return true; // all actions handled locally
```

## Security Model

### Shadow DOM Isolation

Custom widgets render inside a closed Shadow DOM. Styles and DOM structure are fully isolated from the host page and from other widgets.

### HTML Sanitization

Widget HTML templates are sanitized before rendering:
- `<script>` tags are stripped
- Inline event handlers (`onclick`, `ondragover`, etc.) are stripped
- `javascript:` URLs are blocked
- `data:text/html` URLs are blocked

Interactive behavior is achieved exclusively through `data-action` attributes and the `js` handler — never through inline HTML event attributes.

### CSS Sanitization

- `@import` rules are blocked
- External `url()` references are blocked

### JS Handler Scope

The `js` handler runs in the browser's main thread with the same origin. It has access to standard Web APIs but is scoped to the widget's Shadow DOM via the `root` parameter. The handler has no direct access to the host page's DOM, other widgets' state, or the agent's conversation history.

### Size Limits

| Limit | Value |
|-------|-------|
| Widget ID | 2–49 characters, `[a-z][a-z0-9-]+` |
| HTML + CSS size | 50 KB max |
| Max active widget types | 30 per session |

## Built-in Component Catalog

GenUI includes 34 built-in component types that render without `define`. These are optimized for chat-embedded data presentation.

### Data Display

| Type | Key Fields | Description |
|------|-----------|-------------|
| `card` | `title, text, icon` | Simple prose card (1–3 sentences) |
| `stats` | `title, items:[{label,value}]` | Grid of label/value pairs |
| `kv` | `title, items:[{key,value}]` | Key-value rows |
| `table` | `title, headers:[], rows:[[]]` | Tabular data |
| `code` | `title, language, code` | Syntax-highlighted code block |
| `tags` | `label, items:[{text,color}]` | Tag/badge list |
| `accordion` | `title, sections:[{title,content}]` | Collapsible sections |
| `tabs` | `title, tabs:[{label,content}], active` | Tabbed content |

### Metrics & Visualization

| Type | Key Fields | Description |
|------|-----------|-------------|
| `gauge` | `label, value, max, unit, color` | Circular/arc gauge |
| `progress` | `label, value, max, icon, color` | Progress bar |
| `sparkline` | `label, values:[], color, trend` | Inline trend chart |
| `chart-bar` | `title, labels:[], datasets:[{label,data:[],color}]` | Bar chart |
| `chart-line` | `title, labels:[], datasets:[{label,data:[],color}]` | Line chart |
| `chart-pie` | `title, slices:[{label,value,color}]` | Pie chart |
| `stacked-bar` | `title, items:[{label,value,color}]` | Stacked horizontal bar |
| `rating` | `title, value, max` | Star rating |

### Layout & Status

| Type | Key Fields | Description |
|------|-----------|-------------|
| `hero` | `title, subtitle, icon, badge, gradient` | Header/hero block |
| `alert` | `title, message, severity` | Alert banner (info/warning/error/success) |
| `status` | `title, text, color` | Status indicator |
| `timeline` | `title, items:[{title,text,date,icon,status}]` | Vertical timeline |
| `checklist` | `title, items:[{text,checked}]` | Task/checkbox list |
| `streak` | `title, days:[], active:[]` | Activity streak visualization |

### Interactive

| Type | Key Fields | Description |
|------|-----------|-------------|
| `buttons` | `title, buttons:[{label,action,style}]` | Action button group |
| `chips` | `title, chips:[{text,value,checked}]` | Selectable chip group |
| `toggle` | `label, checked` | Toggle switch |
| `input` | `label, type, placeholder, value` | Text input |
| `slider` | `label, value, min, max` | Range slider |
| `form` | `title, id, fields:[{name,type,label,value}], actions:[{label,action,style}]` | Complete form |
| `form-strip` | `title, desc, icon, fields:[...], action, label` | Compact inline form |

### Media & Links

| Type | Key Fields | Description |
|------|-----------|-------------|
| `image` | `title, src, caption, alt` | Image display |
| `video` | `title, src, caption` | Video player |
| `link-card` | `title, desc, url, icon, color` | Rich link preview |
| `weather` | `icon, city, temp, condition` | Weather card |

## TOON Format (Token-Efficient Encoding)

GenUI supports **TOON** (Token-Oriented Object Notation) as an alternative to JSON for canvas ops. TOON is a compact, human-readable encoding that saves **30–40% tokens** on structured data.

TOON is especially useful when the LLM generates ops directly in its text output (code fence mode).

### Syntax

```
key: value                     # Simple property
key[N]: val1,val2,val3         # Flat array
key[N]{f1,f2}:                 # Tabular array (uniform objects)
  row1f1,row1f2                #   One row per line
  row2f1,row2f2
---                            # Op separator
```

### Rules

- Indentation: 2 spaces for nesting
- Strings: no quotes needed unless they contain `,` `:` or leading/trailing whitespace
- Booleans: `true`/`false`, null: `null`, numbers: as-is
- `---` separates multiple ops in a single block

### Example

**JSON (185 tokens):**
```json
{"op":"upsert","id":"srv","type":"stats","data":{"title":"Services","items":[{"label":"Uptime","value":"14d"},{"label":"Requests","value":"1.2M"},{"label":"Errors","value":"0.03%"}]}}
```

**TOON (110 tokens):**
```
op: upsert
id: srv
type: stats
data:
  title: Services
  items[3]{label,value}:
    Uptime,14d
    Requests,1.2M
    Errors,0.03%
```

## Templates (Instant Render)

For common layouts, GenUI provides **template presets** that render in <5ms with zero model latency.

| Template | Key Fields |
|----------|-----------|
| `dashboard` | `title, subtitle, gauges, stats, sparkline, barChart, pieChart, cards, progress, alert` |
| `status` | `title, subtitle, stats, checklist, timeline, kv, alert` |
| `form` | `title, subtitle, formTitle, formId, fields, actions, alert` |
| `timeline` | `title, subtitle, timelineTitle, items, stats` |
| `chart` | `title, subtitle, bar, pie, line, sparklines, gauges, stacked` |
| `checklist` | `title, subtitle, listTitle, items, stats, alert` |
| `detail` | `title, subtitle, kvTitle, items, tags, actions` |
| `email` | `heading, subtitle, to, subject, body, sendLabel, note` |

Templates are invoked via a template block type (transport-dependent) with a `tpl` field specifying the template name and all data fields inline.

## Comparison with Other Protocols

| Feature | GenUI | A2UI v0.9 | AG-UI | Claude Artifacts |
|---------|-------|-----------|-------|-----------------|
| Client-side logic | ✅ `js` handler | ❌ Agent round-trip | ❌ Agent round-trip | ✅ Full JS (sandboxed iframe) |
| Component catalog | 34 built-in + custom | ~20 basic + custom catalogs | Framework-dependent | Freeform HTML |
| Token efficiency | ✅ TOON format (30-40% savings) | ❌ JSON only | ❌ JSON only | N/A (full code) |
| Shadow DOM isolation | ✅ Per-widget | ❌ Renderer-dependent | ❌ Framework-dependent | ✅ iframe sandbox |
| Template system | ✅ Mustache-like, compiled | ❌ Data binding via JSON Pointer | ❌ None | ❌ None |
| Inline in chat | ✅ Native | ❌ Separate surface | ❌ Framework-dependent | ❌ Side panel |
| Drag-and-drop | ✅ Built-in (`data-action`) | ❌ Not specified | ❌ Not specified | ✅ Manual JS |
| State persistence | ✅ Server-side (survives reconnect) | ✅ Surface model | ❌ Client-side | ❌ Per-artifact |
| Transport | WS, SSE, MCP, code fence | A2A, MCP, SSE, WS, REST | SSE, HTTP | HTTP |
| Standardization | This spec (v0.1) | Google-backed, v0.9 draft | CopilotKit ecosystem | Proprietary |

### Key Differentiator

A2UI and AG-UI model the agent as a **real-time UI controller** — every user interaction flows back to the agent, which decides the next UI state. This creates a responsive, agent-driven experience at the cost of latency, token cost, and reliability.

GenUI models the agent as a **software developer** — it writes an application (structure + style + behavior + data), deploys it to the client, and steps back. The application runs independently. The agent can intervene when needed (via `patch`/`update`), but the default is autonomy.

This is the difference between remote desktop and installing an app.

## Implementation Notes

### Rendering

- Built-in components should be implemented as native UI elements (not iframes).
- Custom widgets (`define`) should use Shadow DOM for style and DOM isolation.
- Templates are compiled once at `define` time and evaluated on each `render()`.
- Ops arriving during an active stream should be buffered and rendered after stream completion.

### Persistence

- The server should persist the current canvas state (list of active ops).
- On client reconnect, the full state is replayed: `define` ops first (sorted), then `upsert`/`patch` ops.
- `clear` resets the persisted state.

### Action Routing

1. User interacts with a `data-action` element.
2. The widget's `js` handler is called (if defined).
3. If the handler returns `true`, the action is handled locally. Done.
4. If the handler returns `false`/`undefined`, or no handler exists, the action is emitted as a `widget-action` event.
5. The transport layer delivers the event to the server/agent.
6. The agent may respond with new ops (e.g., `patch` to update state).

## Future Directions

- **Widget marketplace** — shareable, installable widget definitions
- **Inter-widget communication** — events between widgets on the same canvas
- **Persistent storage API** — widgets can save/load data across sessions
- **Collaboration** — multiple users viewing/interacting with the same canvas
- **A2UI bridge** — bidirectional conversion between GenUI ops and A2UI messages

## References

- [A2UI Protocol v0.9](https://a2ui.org/specification/v0.9-a2ui/) — Google's agent-to-UI protocol
- [AG-UI Protocol](https://docs.ag-ui.com/introduction) — CopilotKit's agent-user interaction protocol
- [MCP (Model Context Protocol)](https://modelcontextprotocol.io/) — Anthropic's tool/resource protocol
- [Web Components / Shadow DOM](https://developer.mozilla.org/en-US/docs/Web/API/Web_components) — W3C standard
- [Mustache Templates](https://mustache.github.io/) — Logic-less template syntax (GenUI subset)

---

*GenUI is developed as part of the [Scratchy](https://github.com/yassinebkr/scratchy) project.*
