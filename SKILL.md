---
name: drillflow-diagrams
description: Generate hierarchical, drill-down diagrams as DrillFlow documents — a process or system modelled across several levels, where a node opens into its own sub-diagram. Use when the user asks for a nested/multi-level flowchart, wants to break a step down into sub-steps, or asks for a diagram they can open, edit and share as a link. Produces Mermaid for import or native JSON, and can publish a read-only link.
---

# Generating DrillFlow diagrams

DrillFlow is a diagram editor whose defining feature is **drill-down**: a node
can own a whole sub-diagram, and the reader navigates *into* it as its own
canvas, with breadcrumbs and a tree, to any depth. That is what this skill is
for — if the diagram is flat, plain Mermaid in the reply is a better answer than
this skill.

Canonical format reference: <https://drillflow.de/en/format/>
JSON Schema: <https://drillflow.de/schema/drillflow-document.schema.json>

## Choose a path

**Mermaid (default).** Write a `flowchart`, put each level in a `subgraph`, and
let DrillFlow's importer turn every subgraph into a real subflow. Ids, sizes and
layout are handled for you. Use this unless you need something below.

**Native JSON.** Required for colours, layers, rotation, waypoints, notes,
images or freehand ink, and for publishing a link directly. Lossless.

**If the user asked for a link, you must use JSON.** Publishing takes a
DrillFlow document, and the Mermaid importer runs only in the user's browser —
so Mermaid always ends with them pasting it in themselves. Answer with Mermaid
when they want a diagram to look over; answer with JSON when they want something
they can open or send.

## Path 1 — Mermaid

```
flowchart TD
  A[Take order] --> B{In stock?}
  B -->|yes| C[Ship]
  B -->|no| D[Backorder]

  subgraph A[Take order]
    A1[Check customer] --> A2[Validate items]
  end

  subgraph C[Ship]
    C1[Pick] --> C2[Pack] --> C3[Hand to carrier]
  end
```

Each `subgraph` becomes a node in the parent diagram that owns a sub-diagram.
Nest them as deep as the subject deserves. If a subgraph id was already used as
a plain node, that node is promoted into the container rather than duplicated —
which is why `A[Take order]` above appears both as a step and as a subgraph.

Tell the user to import it with **☰ menu → Import → Mermaid…** at
<https://app.drillflow.de> (paste, or open a `.mmd` file).

### Constraints that actually bite

- Node ids must match `[A-Za-z0-9_]+`. No hyphens, dots, spaces or accents —
  put those in the label.
- Shapes: `[x]` `(x)` `([x])` `[[x]]` `[(x)]` `[/x/]` `[\x\]` `>x]` all become a
  rectangle; `((x))` a circle; `{x}` and `{{x}}` a diamond. There is no separate
  hexagon or stadium shape.
- **You cannot produce a dashed edge.** A connector containing `.` is dotted;
  everything else, including `==>`, is solid.
- `style`, `classDef`, `class` and `click` are ignored — **no colours or links
  survive the import**. If the user asked for colour, use Path 2.
- An edge between two different subgraphs is lifted to the nearest diagram
  containing both, or dropped if there isn't one. Keep edges within a level and
  connect *containers* to each other, not their innards.
- Not understood: `~~~`, edge ids (`e1@-->`), `A@{ shape: … }`, animations.

## Path 2 — native JSON

Read the format reference first. The shape, in one breath: one JSON object with
`id`, `title`, `rootDiagramId`, `createdAt`, `updatedAt` (epoch **milliseconds**)
and a flat `diagrams` map; each diagram has `id`, `title`, `nodes`, `edges`; a
node has `id`, `type`, `label` — and optionally `x`/`y`, which you can leave
out entirely (see the rules below). Hierarchy is two references, both required:

```json
{ "id": "n1", "type": "rect", "label": "Take order", "x": 0, "y": 0, "childDiagramId": "d2" }
```
```json
{ "id": "d2", "title": "Take order", "parentNodeId": "n1", "nodes": [], "edges": [] }
```

`examples/minimal.json` in this skill is a complete two-level document to copy.

### Rules to hold in mind while writing

- `diagrams` is a **flat map**. Nesting is by reference; never nest the JSON.
- Node ids are unique **within their diagram**, not globally.
- **Both endpoints of an edge must be in the same diagram.** There are no
  cross-diagram edges.
- Omit `anchorDeg` — edges float and re-attach themselves correctly. That is
  what you want.
- `text` and `pin` nodes cannot own a subflow.
- **Leave `x`/`y` out and DrillFlow arranges the diagram for you**, top-down,
  following your edges — the same layout an imported Mermaid flowchart gets.
  Prefer this: you know the structure, and the geometry is the one part you
  cannot check from where you are. All or nothing per diagram, though: position
  every node of a diagram or none of them, or it is rejected.
  Place things yourself only when the arrangement carries meaning — swimlanes, a
  deliberate reading order. Then `x`/`y` are top-left corners in px, y downward,
  with roughly 80 px between rows and 48 px between siblings.
  The layout orders each row to keep related branches together and routes long
  edges around the shapes they pass, but it minimises crossings rather than
  promising none — a densely cross-linked diagram may still show a few.
- **Omit `w`/`h` and DrillFlow sizes each shape to its own label**, corners
  included. Prefer this for the same reason you leave `x`/`y` out: you cannot
  see your output, so a size you guessed is a size nobody checked. (Sizes only
  fall back to the flat defaults — rect 140×60, circle/diamond 100×100, text
  140×40, image 200×150 — when there is no label to measure.)
- If you *do* set them, remember a `diamond` needs **twice** the text block on
  both axes and a `circle` about **1.4×**: a question that fits a 140×60 rect
  hangs out of a 100×100 diamond at the top and bottom. Size decision nodes
  generously or leave them unsized.
- Node `lineStyle` allows `double`/`none`; edge `lineStyle` does not.
- Image `src` must be an inline `data:image/…` URI; a remote URL is rejected,
  as is any colour containing `url(...)`.

### Validate before you hand it over

Check against the JSON Schema if you can run a validator. Otherwise walk this
list — the editor refuses the whole document on the first failure:

1. `diagrams[rootDiagramId]` exists.
2. Every `childDiagramId` is a key of `diagrams`, and that diagram's
   `parentNodeId` names the owning node.
3. Every edge's `from.nodeId` and `to.nodeId` exist **in that same diagram**.
4. Node ids are unique per diagram. Within one diagram, either every node
   has numeric `x` and `y`, or none does — never a mix.
5. `draw` nodes have at least one ink point; `image` nodes have a
   `data:image/…` src.
6. Every `layerId` names a layer of the same diagram, and an edge's layer is not
   earlier than its endpoints'.
7. `createdAt`/`updatedAt` are milliseconds.

## Publishing a link (optional)

If the user wants a link rather than a file, publish the JSON document:

```bash
curl -X POST https://api.drillflow.de/api/doc \
  -H 'Content-Type: application/json' \
  -d '{"data": { ...document... }}'
```

Returns `{"slug": "...", "url": "https://app.drillflow.de/view/<slug>"}` — a
read-only canvas with working drill-down, no account or install needed.

Say these out loud to the user rather than burying them:

- **Anonymous links expire 90 days after creation.** Signing in at
  app.drillflow.de before sharing makes a link permanent and revocable.
- The link is **public to anyone holding it**. Do not publish anything
  confidential without asking first.
- Limits: 10 publishes per minute per IP, 2 MB per document.

Publishing is a network write of the user's content — ask before doing it, and
prefer handing over the file when the user did not ask for a link.

## Files in this skill

| File | What it is |
|---|---|
| `examples/minimal.json` | Smallest complete document: two levels, one subflow. |
| `examples/order-handling.mmd` | Mermaid with nested subgraphs; the Path 1 example. |
| `examples/order-handling.json` | The same diagram as native JSON. Three levels on the longest path: root → Ship → Pack. |

("Levels" counts canvases, so a root diagram with no subflows is one level.)
