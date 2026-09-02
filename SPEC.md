# DrillFlow document format

DrillFlow is a diagram editor built around **drill-down**: a node can own a
whole sub-diagram, and you navigate *into* it as its own canvas. This document
is the canonical description of the file format, written so that a person or an
AI agent can generate a valid DrillFlow document from scratch.

Everything here describes the format as the app at
[app.drillflow.de](https://app.drillflow.de) reads and writes it. If you only
want the short version: **generate Mermaid, import it** — jump to
[Generating a document](#generating-a-document).

## The shape of a document

A document is a single JSON object. Diagrams live in a **flat map**, and
nesting is expressed by references between them, never by nesting the JSON.

```json
{
  "id": "doc-1",
  "title": "Order handling",
  "rootDiagramId": "root",
  "createdAt": 1754006400000,
  "updatedAt": 1754006400000,
  "diagrams": {
    "root": {
      "id": "root",
      "title": "Order handling",
      "nodes": [
        { "id": "n1", "type": "rect", "label": "Take order", "x": 0, "y": 0, "childDiagramId": "d2" },
        { "id": "n2", "type": "diamond", "label": "In stock?", "x": 0, "y": 140 }
      ],
      "edges": [
        { "id": "e1", "from": { "nodeId": "n1" }, "to": { "nodeId": "n2" }, "label": "then" }
      ]
    },
    "d2": {
      "id": "d2",
      "title": "Take order",
      "parentNodeId": "n1",
      "nodes": [{ "id": "n3", "type": "rect", "label": "Check customer", "x": 0, "y": 0 }],
      "edges": []
    }
  }
}
```

That is a complete, valid document: two diagrams, one of them a subflow of the
node `n1`.

### Whose style a reader sees

Presentation resolves in one order, and only the first two ever live in a file:

1. **What the element says.** Anything set per node or edge — font, colour,
   arrowhead, line style — belongs to that element and travels with the
   document.
2. **`style` on the document.** A snapshot of the sharer's own defaults, taken
   when a link is created. It exists because those defaults live in their
   browser: without it, an author who styled nothing by hand would hand over no
   style at all. DrillFlow never writes it into a working document.
3. **The reader's own defaults**, then the built-ins listed under
   [Defaults](#defaults).

A reader of a shared link may decline 1 and 2 entirely and see the diagram in
their own defaults instead. Emphasis (`bold`, `italic`, `align`) stays either
way: it marks a particular word rather than setting a theme.

`style` accepts the same keys an element does, minus emphasis: `fontFamily`,
`fontSize`, `nodeColor`, `edgeColor`, `colorMode`, `nodeLineStyle`,
`edgeLineStyle`, `arrowType`, `arrowSize`, `edgeStyle`. Colours must not contain
`url(...)`, for the reason given under the element rules.

### What DrillFlow fills in

A document written by a generator — an AI agent, a script — is allowed to leave
out what can only be read one way. DrillFlow completes it on load and the file
you export afterwards carries the completed form:

| Left out | What happens |
|---|---|
| `id` on an edge | An id is assigned. Nothing references an edge by id, so there is only one possible reading. Stable across re-imports of the same file. |
| `type` on a node, or an unknown one | Read as `rect`. |
| `label` on a node | Read as empty. |
| A `labelRuns` entry that does not fit its `label` | Cleaned up: offsets are clamped to the string, and empty, inverted and format-less runs are dropped. |
| `title` on a diagram or the document, `id` on the document | Filled in. |
| `x`/`y` on **every** node of a diagram | The diagram is laid out for you — see [Or let DrillFlow place them](#or-let-drillflow-place-them). |
| An edge endpoint naming a node the diagram does not have | The edge is dropped. The shape is gone, so the arrow that pointed at it is gone — the only other reading would be to refuse the whole document over one arrow. |

What is **not** filled in, because guessing would change the meaning of the
document rather than complete it:

- `id` on a **node**. Edges point at nodes by id; inventing one would silently
  change which shapes an arrow joins. A node without an id is an error.
- Either endpoint of an edge (`from.nodeId` / `to.nodeId`) — a MISSING or
  non-string endpoint is an error. An endpoint that is a well-formed reference
  to a node that is not there is a different case: that edge is dropped (above).
- `x`/`y` on *some* nodes of a diagram — all or none, per the layout rule.

`POST /api/doc` reports the same list in a `warnings` array, so a generator that
shares a document without opening the editor still learns what it left out.

### Document

| Field | Type | Required | Notes |
|---|---|---|---|
| `id` | string | yes | Identity of the document itself. |
| `title` | string | yes | Shown in the editor and used as the export filename. |
| `rootDiagramId` | string | yes | Key into `diagrams`. Every document has exactly one root. |
| `diagrams` | object | yes | Map of diagram id to diagram. Not an array. |
| `createdAt` | number | yes | Unix epoch **milliseconds**. |
| `updatedAt` | number | yes | Unix epoch milliseconds. |
| `style` | object | no | The style the document was **shared** with — see [Whose style a reader sees](#whose-style-a-reader-sees). Written into a shared copy, not into a working document. |

### Diagram

| Field | Type | Required | Notes |
|---|---|---|---|
| `id` | string | yes | Should equal its key in `diagrams`. |
| `title` | string | yes | A subflow's title mirrors its owning node's label. |
| `nodes` | array | yes | May be empty. **Order is paint order** — a later entry draws on top of an earlier one. |
| `edges` | array | yes | May be empty. |
| `parentNodeId` | string | no | Present exactly on sub-diagrams: the id of the node that owns this diagram. |
| `layers` | array | no | Named sheets for showing/hiding groups of elements. Absent means everything is on the implicit base sheet. |
| `showPorts` | boolean | no | Does this sub-diagram show the level above's inputs and outputs? Absent follows the reader's own preference; `true`/`false` is a deliberate choice for this one sub-diagram. Meaningless on the root. See [Boundary ports](#boundary-ports). |

### Node

| Field | Type | Required | Notes |
|---|---|---|---|
| `id` | string | yes | Unique **within its diagram**. |
| `type` | string | yes | One of `rect`, `circle`, `diamond`, `text`, `pin`, `draw`, `image`. An absent or unknown type is read as `rect`, but write the one you mean. |
| `label` | string | yes | May be empty. `\n` makes a line break. An absent label is read as empty. |
| `labelRuns` | array | no | Formatted stretches of `label` — see [Formatting inside a label](#formatting-inside-a-label). |
| `x`, `y` | number | no | Top-left corner in flow units (px at 100% zoom). Y grows downward. Omit on every node of a diagram to have DrillFlow arrange it — see [Or let DrillFlow place them](#or-let-drillflow-place-them). Give both or neither. |
| `w`, `h` | number | no | Size. Omitted means the per-type default below. |
| `childDiagramId` | string | no | Key into `diagrams` — this node owns that sub-diagram. |
| `color` | string | no | Any CSS colour. Must not contain `url(...)`. |
| `colorMode` | string | no | `fill` (default) or `stroke`. |
| `lineStyle` | string | no | Shape outline: `solid` (default), `dashed`, `dotted`, `double`, `none`. |
| `rotation` | number | no | Degrees clockwise, `[0, 360)`. Shape, text and image nodes only. |
| `notes` | string | no | Free text, shown only in the inspector. |
| `layerId` | string | no | Id of a layer in this diagram's `layers`. Absent = base sheet. |
| `fontSize` | number | no | px, default 14. |
| `fontFamily` | string | no | `sketch` (default), `sans`, `serif`, `mono`. |
| `bold`, `italic` | boolean | no | |
| `align` | string | no | `left`, `center` (default), `right`. |
| `labelColor` | string | no | The label's ink. Without it the ink is derived — see [Colours and the reader's theme](#colours-and-the-reader-s-theme). Must not contain `url(...)`. |
| `points` | array | draw only | Ink points `{x, y, p?}` **relative to the node's x/y**. `p` is stylus pressure. |
| `strokeWidth` | string | draw only | `s`, `m` (default), `l`. |
| `src` | string | image only | Inline image as a `data:image/...` URI. Remote URLs are rejected. |
| `lockAspect` | boolean | image only | Keep aspect on resize. Absent = true. |
| `portOf` | string | no | **Boundary port** — the id of an edge in the PARENT diagram that this node stands for. Written by the editor, not by hand; see [Boundary ports](#boundary-ports). |

Node types in practice:

- `rect`, `circle`, `diamond` — the ordinary shapes. These can own subflows.
- `text` — a bare label with no outline. Cannot own a subflow.
- `pin` — a dangling arrow endpoint, created in the editor by dragging an edge
  to empty canvas. Cannot own a subflow. Agents rarely need these.
- `draw` — freehand ink. Requires a non-empty `points` array.
- `image` — a picture, inline in `src`.

### Edge

| Field | Type | Required | Notes |
|---|---|---|---|
| `id` | string | no | Nothing references an edge by id, so DrillFlow assigns one when it is absent — see [What DrillFlow fills in](#what-drillflow-fills-in). Write one if you want stable ids across exports. |
| `from`, `to` | object | yes | `{ "nodeId": "..." }`, optionally with `anchorDeg` or `anchorRel`. |
| `label` | string | no | |
| `labelRuns` | array | no | Formatted stretches of `label` — see [Formatting inside a label](#formatting-inside-a-label). |
| `arrowType` | string | no | `directed` (default), `bidirectional`, `undirected`. |
| `edgeStyle` | string | no | `straight` or `bezier`. Absent follows the viewer's preference. |
| `lineStyle` | string | no | `solid` (default), `dashed`, `dotted`. Note: no `double`/`none`, unlike nodes. |
| `color` | string | no | Stroke and arrowheads. Must not contain `url(...)`. |
| `arrowSize` | number | no | Arrowhead size in px, default 20. |
| `labelColor` | string | no | The label's ink; without it the label follows the reader's theme. Must not contain `url(...)`. |
| `waypoints` | array | no | `{x, y}` points the edge is dragged through. |
| `layerId` | string | no | Must not be an *earlier* sheet than either endpoint. |

Both endpoints of an edge must be nodes **in the same diagram**. There are no
cross-diagram edges; hierarchy is expressed by `childDiagramId`, not by edges.

#### Formatting inside a label

A label is one plain string, and everything that reads it — node search, the
subflow title it syncs with, the breadcrumbs, the Mermaid/Markdown/text exports
— reads that string. `labelRuns` decorates stretches of it without touching it:

```json
{
  "id": "n1", "type": "rect", "label": "Ship it fast",
  "labelRuns": [{ "start": 0, "end": 4, "bold": true }]
}
```

| Field | Type | Required | Notes |
|---|---|---|---|
| `start`, `end` | integer | yes | `[start, end)` offsets into the element's exact `label`. |
| `fontSize` | number | no | px. |
| `fontFamily` | string | no | `sketch`, `sans`, `serif`, `mono`. |
| `bold`, `italic` | boolean | no | `false` is meaningful: it un-bolds a word inside a bold element. |
| `color` | string | no | Any CSS colour, no `url(...)`. |

Rules worth knowing before you write one:

- **`align` is not a run property.** Alignment belongs to the whole label.
- **The array is a paint order.** A later run wins, property by property — so
  "bold the sentence" plus "colour one word" compose.
- **A label rewritten without runs loses them.** Renaming a node from the
  subflow tree, a title sync, an import: anything that writes the string without
  saying what happens to the formatting clears it. Losing the formatting beats
  bolding the wrong syllable once the offsets no longer line up.
- **Plain-text exports lose it too.** Mermaid, Markdown and text read `label`;
  PNG, SVG and PDF photograph the canvas and keep it.
- **A run's colour is absolute** — unlike an element's colour it does not flip
  with the reader's theme, so pick one that works in both, or leave it out.

#### Endpoint attachment

Edges **float** by default: the endpoint is recomputed to the exact point on the
shape's outline facing the other end, so it stays correct when a node is moved,
resized or rotated. Omit `anchorDeg` and you get that behaviour — which is what
a generator wants.

`anchorDeg` pins an end to a fixed point on the outline: a direction from the
node's centre in degrees clockwise from +X, measured in the node's *unrotated*
frame (screen axes, y down). So `0` is the right edge, `90` the bottom, `180`
the left, `270` the top. A pinned point keeps its relative spot when the node is
resized and turns with the node when it is rotated.

`anchorRel` pins an end *inside* the node instead of on its outline:
`{ "x": 0.25, "y": 0.75 }`, fractions of the node's width and height in the same
unrotated frame (centre = `0.5`/`0.5`). It exists because a direction can only
ever name a point on the outline, and an arrow may need to point at something
*within* a picture — a button, a field, a row of a pasted screenshot. It is
resize- and rotation-proof for the same reason `anchorDeg` is, and it **wins
over `anchorDeg`** when both are present.

The editor only ever writes it on an `image` node: on every other shape a drop
in the body leaves the endpoint floating. A reader that does not know the field
draws the arrow to the edge of the node, which is what DrillFlow itself did
before the field existed — an honest degradation, not a broken document.

`handleId` appears in documents written before floating edges. It is still
parsed and ignored — never write it.

### Layer

| Field | Type | Required |
|---|---|---|
| `id` | string | yes |
| `name` | string | yes |
| `visible` | boolean | yes |

`Diagram.layers` is ordered base-first over an implicit base sheet. A node or
edge with no `layerId` sits on that base sheet, which is index -1. The one
invariant: an edge's layer index must be greater than or equal to both of its
endpoints' — an edge may never reference a later sheet than the shapes it
connects.

Layers control **visibility and grouping, not depth.** The order exists for that
invariant and for the panel's own top-to-bottom listing; it does not affect what
covers what. Stacking is the order of `nodes` alone, so a shape on the base
sheet can quite legitimately paint over one on a later sheet.

## Drill-down semantics

This is the part that makes a DrillFlow document different from a flat diagram
file, so it is worth stating precisely.

- A document has exactly one root diagram, named by `rootDiagramId`.
- A node owns a sub-diagram by carrying `childDiagramId`. That sub-diagram
  carries `parentNodeId` pointing back at the node. Write **both**.
- `text` and `pin` nodes cannot own subflows.
- Depth is unlimited. Because `diagrams` is a flat map and a subflow is a key
  lookup, **nesting adds a map entry, never a level of JSON nesting** — 200
  levels of drill-down is still a JSON document about 6 levels deep.
- Throughout this document, **"levels" counts canvases**: a root diagram with no
  subflows is one level, and drilling into one of its nodes reaches level two.
- A subflow's `title` and its owning node's `label` are kept in sync by the
  editor. Generators should just set them to the same string.
- New subflows are empty; there is no auto-created "start" node.

### Boundary ports

Drilling into a subflow answers "what happens inside", but by itself it loses
sight of what *reaches* the node from outside. A **boundary port** is a node
inside the subflow that stands for one edge on the level above, so an arrow in
the subflow can dock to the thing that crosses its boundary — "this input is
handled by that step", and an input may branch to several.

It is an ordinary node carrying one extra field:

```json
{ "id": "p1", "type": "circle", "label": "Order received",
  "portOf": "e17", "x": -284, "y": -12, "w": 24, "h": 24 }
```

- **`portOf` is an edge id in the PARENT diagram** — the diagram whose node owns
  this one. Which side it is on is derived: the parent edge ends at the owner
  node → an input, leaves it → an output.
- **The name is rendered from the peer** on the level above (the node at the
  edge's other end), so renaming that box renames the port. A stored `label` is
  only what a reader falls back to when the parent edge is gone.
- **A port whose parent edge disappeared is kept and marked**, never deleted:
  the arrows somebody drew to it inside the subflow are real work.
- **How it is drawn is the reader's setting**, not the document's — an arrow tag
  or a plain circle. Nothing about the shape is stored.

**For generators: do not write `portOf` yourself.** The editor creates and
maintains these nodes; a hand-written one whose id does not name a real parent
edge is simply an orphan. The field is documented so that a document survives a
round trip through your tool — copy it through untouched, and leave the port's
size and position alone. Everything else about a port is an ordinary node, so a
reader that ignores the field renders a small labelled shape and loses nothing
but the meaning.

Ports are **context, not content**: DrillFlow keeps them out of the Markdown
outline, the Mermaid export, the "unconnected shapes" check and the per-diagram
node count. A tool that summarises a diagram should do the same.

## Defaults

A generator that omits `w`/`h` gets these sizes, and layout code (including
DrillFlow's own auto-layout) assumes them:

| Type | Default w × h |
|---|---|
| `rect` | 140 × 60 |
| `circle` | 100 × 100 |
| `diamond` | 100 × 100 |
| `text` | 140 × 40 |
| `pin` | 12 × 12 |
| `draw` | 40 × 40 |
| `image` | 200 × 150 |

Other defaults: font size 14, font `sketch`, alignment `center`, arrow size 20,
edge `arrowType` `directed`, `lineStyle` `solid`, image `lockAspect` true.

Positions are top-left corners in flow units. A comfortable manual layout is
roughly 80 px of vertical gap between rows and 48 px of horizontal gap between
siblings, which is what DrillFlow's own importer uses.

### Or let DrillFlow place them

`x` and `y` are **optional**. Leave them off every node of a diagram and
DrillFlow arranges that diagram itself — the same top-down layout it uses for an
imported Mermaid flowchart, following your edges to decide what sits below what.

```json
{ "id": "n1", "type": "rect", "label": "Take order", "childDiagramId": "d2" }
```

This is usually the right choice. You know the structure; the layout is
arithmetic, and getting it wrong is the one mistake you cannot see from where
you are standing.

Four things to know:

- **Crossings are minimised, not eliminated.** The layout ranks your nodes into
  rows, orders each row so branches that belong together stay together, and
  routes an edge that skips or loops through a corridor of its own rather than
  through the shapes in between. It does not promise a crossing-free picture —
  for many graphs there is none — so a densely cross-linked diagram may still
  show a few. It never draws an edge through a node that is not its own endpoint.
- **All or nothing, per diagram.** Position every node of a diagram or none of
  them. A mix is rejected rather than half-arranged — it is nearly always a
  forgotten node, and honouring it would mean throwing away the coordinates you
  did supply. Each diagram is judged on its own, so one may be hand-placed while
  another is left to us.
- **Nothing is rewritten.** The document you publish keeps exactly what you
  wrote; positions are computed when it is opened. The layout is deterministic,
  so every reader sees the same picture.
- **Place things yourself when the arrangement carries meaning** — swimlanes,
  a deliberate left-to-right reading order, anything where position is part of
  what you are saying.

### Will the label fit?

**A shape you gave a `w`/`h` never resizes itself to fit its text.** Labels wrap,
and a word longer than the box breaks rather than sticking out — but if the text
needs more room than the box has, it overflows the bottom. It is not clipped and
the box does not grow.

**If you omit both `w` and `h`, DrillFlow derives a size that fits the label**,
including the room a `diamond` or `circle` loses at its corners. Omitting them is
therefore the safest thing an agent can do: you cannot see your own output, and a
size you guessed is a size nobody checked.

Size the box yourself only when you want a specific shape. Measured at the
default 14 px in the default font:

- **about 18 characters per line** in a default 140 px-wide `rect` (the shape's
  own padding takes 14 px from each side, leaving 112 px of text)
- **two lines** inside the default 60 px height
- a `diamond` needs **twice** the text block on both axes, and a `circle` about
  **1.4×**. This is geometry, not a fudge: a text block `tw × th` centred in a
  rhombus `w × h` fits only while `tw/w + th/h ≤ 1`, and in an ellipse while
  `(tw/w)² + (th/h)² ≤ 1`. A label that sits comfortably in a 140×60 rect will
  hang out of a 100×100 diamond on both ends.

A label longer than roughly 36 characters therefore wants an explicit `w`/`h` —
or no size at all, and let the editor work it out. Widening is usually better
than heightening; diagrams read better in rows.

### Colours and the reader's theme

`color` is a **fill**, and DrillFlow has a light and a dark theme. You cannot
know which one the reader uses — a shared link is opened under their preference,
not yours. The label's own colour is chosen for you, black or white, whichever
reads better on your fill, so text stays legible either way. What you cannot
control is how the fill itself sits against their canvas: **mid-tone colours
travel best**, while very pale and very dark fills look right in one theme and
heavy in the other.

`labelColor` overrides that choice for a whole label, and a `labelRuns` entry's
`color` overrides it again for the stretch it covers. Both are absolute: they do
not flip with the reader's theme, so pick something legible in either — that is
the trade you take for saying it yourself. A reader who switches a shared link
to their own style gets the derived ink back, exactly as they do for `color`.

## Generating a document

There are two paths. Pick the first unless you need something it can't express.

One difference decides it more often than any feature: **Mermaid ends in text a
person pastes into the editor, JSON can end in a link.** Publishing takes a
DrillFlow document (see [Publishing a document](#publishing-a-document)) and the
Mermaid importer runs only in the browser, so an assistant asked for a *ready
link* has to write JSON. Asked for a diagram to look over, Mermaid is the
friendlier answer.

### Path 1 — Mermaid flowchart (recommended)

DrillFlow imports Mermaid `flowchart` text and turns every `subgraph` into a
real subflow, so the drill-down structure survives. This is the easiest path for
an LLM: write ordinary Mermaid, let the importer do ids, sizes and layout.

```
flowchart TD
  A[Take order] --> B{In stock?}
  B -->|yes| C[Ship]
  B -->|no| D[Backorder]

  subgraph A[Take order]
    A1[Check customer] --> A2[Validate items]
  end
```

Import it in the editor: **☰ menu → Import → Mermaid…**, then paste or open a
`.mmd` file.

**What the importer supports**

- Header `flowchart` or `graph`, case-insensitive. The direction token
  (`TD`, `LR`, …) is accepted and then ignored — layout is always top-down.
- Node ids matching `[A-Za-z0-9_]+`. No hyphens, dots, spaces or non-ASCII in
  ids; put those in the label instead.
- Shapes:

| Mermaid | Becomes |
|---|---|
| `A[text]`, `A(text)`, `A([text])`, `A[[text]]`, `A[(text)]`, `A[/text/]`, `A[\text\]`, `A>text]` | `rect` |
| `A((text))` | `circle` |
| `A{text}`, `A{{text}}` | `diamond` |
| `A` (bare id, no shape) | `rect`, labelled with the id |

- Edges: `-->`, `---`, `-.->`, `==>` and their variants; arrowheads `>`, `o`,
  `x` at either end (`o` and `x` are treated as plain heads). Two heads give
  `bidirectional`, one gives `directed`, none gives `undirected`. `<--` reverses
  the direction.
- Edge labels in both spellings: `A -->|yes| B` and `A -- yes --> B`.
- Chains (`A --> B --> C`) and `&` fan-out (`A & B --> C & D`).
- `subgraph id [Title]` … `end`, nested arbitrarily deep. A subgraph becomes a
  node in the parent diagram that owns the sub-diagram. Referencing the subgraph
  id in an edge connects to that node. If the id was already used as a plain
  node earlier, that node is promoted into the container rather than duplicated.
- `%%` comments; `;` statement separators.

**What it does not support** — know these before you generate:

- **There is no way to produce a `dashed` edge.** A line containing `.` becomes
  `dotted`; everything else, including thick `==>`, becomes `solid`.
- `style`, `classDef`, `class`, `click` and `direction` are parsed and ignored,
  so **no colours or links come across**.
- `~~~` invisible links, edge ids (`e1@-->`), animations and the Mermaid v11
  `A@{ shape: ... }` syntax are not understood.
- Markdown strings and arbitrary HTML in labels are not rendered; `<br>` becomes
  a line break, and `#quot;`/`&amp;`/`&lt;`/`&gt;` are decoded.
- An edge whose two ends live in different subgraphs is lifted to the nearest
  diagram that contains both. If that is impossible, or if it collapses into a
  self-loop, the edge is dropped — the importer reports how many.

Everything the importer produces has floating endpoints and auto-computed
positions, which is usually what you want.

### Path 2 — native JSON

Write the document object described above and import it: **☰ menu → Open…**, or
publish it through the API. Lossless, and the only way to set colours, layers,
rotation, waypoints, notes, images or ink.

If you take this path, run through the checklist below before handing the file
over — the editor refuses a document that breaks any of it.

## Using an AI assistant

Any assistant can follow Path 1 from this page alone: ask it for a Mermaid
flowchart, put each level in a `subgraph`, and import the result. Nothing needs
to be installed for that.

If you use Claude, there is a ready-made **skill** that teaches it the parts
that actually trip models up — the id and shape constraints, what the importer
silently drops, the validation checklist, and how to publish a link:

- [SKILL.md](https://drillflow.de/skills/drillflow-diagrams/SKILL.md) — drop it
  into your assistant's skills directory (for Claude Code that is
  `.claude/skills/drillflow-diagrams/`)
- Worked examples —
  [minimal.json](https://drillflow.de/skills/drillflow-diagrams/examples/minimal.json)
  (the smallest complete document),
  [order-handling.mmd](https://drillflow.de/skills/drillflow-diagrams/examples/order-handling.mmd)
  (a Mermaid flowchart three levels deep: root → Ship → Pack), and
  [order-handling.json](https://drillflow.de/skills/drillflow-diagrams/examples/order-handling.json)
  (the same diagram as native JSON)
- [llms.txt](https://drillflow.de/llms.txt) — the short index, if you would
  rather point a tool at one URL and let it find the rest

The same files, plus this reference and the JSON Schema, are mirrored at
[github.com/capydev42/drillflow-skill](https://github.com/capydev42/drillflow-skill)
if you prefer to clone them.

A caveat worth setting expectations on: models are good at Mermaid and less
good at the details, so expect the occasional invalid id or an edge drawn
between two different subgraphs. The importer reports how many edges it had to
drop rather than quietly mangling the diagram, so mistakes are visible — and a
generated diagram is a starting point you edit, not a finished one.

## Validation checklist

The editor validates on load and refuses the whole document on the first
problem. In order:

1. The document is an object with a non-empty string `rootDiagramId` and an
   object (not array) `diagrams`, and `diagrams[rootDiagramId]` exists.
2. Every diagram is an object with array `nodes` and array `edges`.
3. If `layers` is present it is an array; every layer has a non-empty string
   `id`, a string `name`, a boolean `visible`; layer ids are unique per diagram.
4. Every `layerId` on a node or edge resolves to a layer in that same diagram.
5. Every node has a non-empty string `id`, unique within its diagram, and
   `x`/`y` that are either both numbers or both absent — and within one
   diagram, either every node is positioned or none is.
6. Every `childDiagramId` is a key of `diagrams`.
7. `points`, if present, is an array of `{x, y}` with finite numbers (`p` finite
   when present); a `draw` node has at least one point.
8. An `image` node has a string `src`, and any `src` is an inline
   `data:image/...` URI — remote URLs are refused.
9. No `color` contains `url(...)`.
10. `strokeWidth` is `s`/`m`/`l`; `rotation` is a finite number; a node's
    `lineStyle` is one of `solid`/`dashed`/`dotted`/`double`/`none`.
11. Every edge has a string `id`, and `from.nodeId`/`to.nodeId` are strings.
    An edge whose endpoint names a node the diagram does not contain is
    **dropped, not refused** — see the healing note below.
12. An edge's layer index is not earlier than either endpoint's.
13. A `parentNodeId` refers to a node that exists somewhere in the document.

Rules 8 and 9 exist because a shared document is rendered in someone else's
browser: an external image URL, or a colour smuggling a CSS `url(...)`, would
report every viewer's IP address back to whoever published the document.

**The loader is more forgiving than this specification.** It never checks
`createdAt` or `updatedAt`, and it completes the fields listed under
[What DrillFlow fills in](#what-drillflow-fills-in) instead of refusing the
document. It also **drops an edge whose `from.nodeId` or `to.nodeId` names a
node that diagram does not contain**, reporting it as a healed field: the shape
is gone, so the arrow that pointed at it is gone too — there is only one
sensible reading, and refusing a whole document over one unreachable arrow
punishes the reader for something the author cannot see. Both endpoints must
still be strings; an endpoint that is not a reference at all is refused. The [JSON Schema](https://drillflow.de/schema/drillflow-document.schema.json)
stays strict on purpose — it describes a complete document, so a generator
validating against it is told about an omission rather than discovering later
that DrillFlow guessed. Generate complete documents; the tolerance is a safety
net, not a shortcut.

## Publishing a document

`POST /api/doc` stores a document and returns a read-only link. It needs no
account and no key — it is the same endpoint the editor's Share button uses.

```bash
curl -X POST https://api.drillflow.de/api/doc \
  -H 'Content-Type: application/json' \
  -d '{"data": { ...document... }}'
```

```json
{ "slug": "aB3xK9pQ", "url": "https://app.drillflow.de/view/aB3xK9pQ" }
```

The returned URL opens a read-only canvas with working drill-down, so an agent
can hand a user a finished diagram without any install or import step.

Limits, all enforced:

- **10 requests per minute** per IP address for publishing (reads allow 120).
- **2 MB** per document. Larger gets `413`.
- The document must pass the structural checks above; a payload that isn't a
  DrillFlow document gets `422`. Ceilings: 5000 diagrams, 20000 nodes and 20000
  edges per diagram, JSON nesting depth 64.
- **Anonymous links expire 90 days after creation** and then return `404`. Sign
  in at [app.drillflow.de](https://app.drillflow.de) before sharing to keep a
  link permanent and to be able to list or revoke it later.

`GET /api/doc/{slug}` returns `{"data": ..., "allowCopy": true}` for a live
link, and `404` for one that is missing, revoked or expired.

Please don't use this as general file hosting — it is a diagram share endpoint.
Abuse reports go to [support@drillflow.de](mailto:support@drillflow.de) and
links are removed on request.

## Reading a document back

Two text forms an agent can consume:

- **Mermaid export** (`☰ → Export → Mermaid…`) is the round-trip companion to the
  import above. Structure survives — shapes, subflows, edges and their labels —
  and re-importing the output reproduces the same hierarchy. It drops colours,
  positions, sizes, rotation, layers, notes, typography, waypoints and pinned
  anchors; `pin`, `draw` and `image` nodes are skipped along with any edge that
  touches one; a `dashed` edge comes back as dotted, and a `text` node as a
  rect.
- **JSON export** is exactly the document object, 2-space indented. Nothing is
  added or stripped, so it is lossless in both directions.

## Notes for tools

- There is no `version` or `$schema` field in a document. Compatibility is
  handled by tolerance: new fields are optional, and unknown ones are ignored.
- Ids are opaque strings. The editor writes UUIDs, but any unique string works,
  and node ids only have to be unique inside their own diagram.
- `createdAt`/`updatedAt` are milliseconds, not seconds.
- The JSON Schema lives at
  `https://drillflow.de/schema/drillflow-document.schema.json` and is the
  machine-readable form of this document.
