---
name: mermaid-diagrams
description: Create Mermaid diagrams (flowchart, sequence, state, ER, class) in the team's blueprint style and export them as an image file: PNG by default (for Confluence), or SVG or PDF if the user prefers. Use whenever the user asks to draw, generate, sketch, restyle or document something as a diagram, such as architecture, process or data flow, service interactions, state lifecycle or data model. Applies a fixed palette, shape vocabulary and layout rules, and renders with mermaid-cli.
---

# Mermaid diagrams (blueprint style)

Every diagram should look like part of one drawing set: a technical-blueprint feel, almost no color, and color used only when it carries meaning. The diagrams are usually exported as PNG images and placed in Confluence pages, but the user chooses the output format (see "Rendering").

## Workflow

1. **Pick the diagram type.**
    - Architecture, process or data flow: `flowchart`
    - Interactions between components over time: `sequenceDiagram`
    - Lifecycle of one thing (an order, a job): `stateDiagram-v2`
    - Data model: `erDiagram`
    - Code structure: `classDiagram`
2. **Write the source** to a `.mmd` file named in kebab-case (for example `order-bursting-flow.mmd`). Put it next to the documentation it belongs to, or in `docs/diagrams/` if the repository has no convention. Start the file with the standard header below, unchanged.
3. **Choose the output format.** If the user has not said which file type they want, ask once before rendering: PNG (default, for Confluence), SVG, or PDF. If they do not answer, use PNG.
4. **Render it** with the command for that format in the "Rendering" section, and look at the result if you can open images.
5. **Run the checklist** at the end. Fix problems in the `.mmd` and render again.
6. **Report** the `.mmd` path and the path of the rendered file. Keep the `.mmd`, because the rendered file cannot be edited later.

If `mmdc` cannot be run in this environment, give the `.mmd` source and the render command, and say clearly that the diagram was not rendered or checked. Never claim a diagram looks right without having rendered it.

## Standard header

Put this at the very top of every `.mmd` file, with `---` on the first line and no blank line before it. It is safe for all diagram types: the `flowchart:` and `themeCSS` keys only affect flowcharts and are ignored elsewhere.

```yaml
---
config:
  theme: base
  themeVariables:
    fontFamily: "JetBrains Mono, IBM Plex Mono, Consolas, monospace"
    fontSize: "14px"
    background: "#f4f7fb"
    primaryColor: "#ffffff"
    primaryBorderColor: "#1b2a4a"
    primaryTextColor: "#1b2a4a"
    secondaryColor: "#e8eef8"
    tertiaryColor: "#f4f7fb"
    lineColor: "#1b2a4a"
    textColor: "#1b2a4a"
    clusterBkg: "#ffffff"
    clusterBorder: "#8fa3c4"
    edgeLabelBackground: "#f4f7fb"
    noteBkgColor: "#e8eef8"
    noteBorderColor: "#8fa3c4"
    noteTextColor: "#1b2a4a"
    actorBkg: "#ffffff"
    actorBorder: "#1b2a4a"
    actorTextColor: "#1b2a4a"
    actorLineColor: "#8fa3c4"
    signalColor: "#1b2a4a"
    signalTextColor: "#1b2a4a"
    labelBoxBkgColor: "#e8eef8"
    labelBoxBorderColor: "#8fa3c4"
    labelTextColor: "#1b2a4a"
    loopTextColor: "#1b2a4a"
    activationBkgColor: "#e8eef8"
    activationBorderColor: "#2f6fed"
    sequenceNumberColor: "#f4f7fb"
  themeCSS: |
    .cluster-label foreignObject { overflow: visible; }
    .cluster-label div { transform: translateX(calc(50% + 16px)); }
  flowchart:
    curve: linear
    nodeSpacing: 36
    rankSpacing: 56
    padding: 18
    htmlLabels: true
---
```

The two `themeCSS` lines move subgraph titles slightly to the right of the group's center line. Without them, arrows entering a group at its center run through the title text.

## Palette

Use only these colors. Do not introduce new ones.

| Color | Hex | Use |
|---|---|---|
| Navy | `#1b2a4a` | All text, lines and node borders; filled background for external actors and endpoints |
| Paper | `#f4f7fb` | Page background |
| White | `#ffffff` | Default node and group fill |
| Tint | `#e8eef8` | Data stores, notes |
| Slate | `#8fa3c4` | Group borders, secondary lines |
| Blue | `#2f6fed` | The single component the diagram is about. At most one or two nodes per diagram. |
| Amber | `#c2701a` | Failure, rejection, alert and warning paths only |

## Flowchart vocabulary

The same shape always means the same thing, across all diagrams.

| Meaning | Syntax | Class |
|---|---|---|
| External actor or system (user, client, third party) | `ID([Label])` | `terminal` |
| Service or component | `ID[Label]` | (default) |
| Shared or called-out step | `ID[[Label]]` | |
| Decision | `ID{Label?}` | |
| Check, rule or limit | `ID{{Label}}` | |
| Data store | `ID[(Label)]` | `store` |
| Queue, bus or stream | `ID[/Label/]` | `store` |
| Alert or terminal event | `ID((Label))` | `fail` if it is an error |
| Failure or rejection step | `ID[Label]` | `fail` |
| The focus component | `ID[Label]` | `accent` |

| Meaning | Syntax |
|---|---|
| Main path | `A ==> B` |
| Normal flow | `A --> B` |
| Async, optional or failure flow | `A -.-> B` |
| Labeled link | `A -- label --> B`, `A == label ==> B`, `A -. label .-> B` |

## Flowchart rules

- **Direction:** use `flowchart TB`. Confluence shrinks wide images to the page width, so left-to-right diagrams end up with tiny text. Use `LR` only when the diagram is narrow.
- **Size:** at most about 18 nodes per diagram. If it needs more, split it into an overview and detail diagrams.
- **Labels:** one to three words in sentence case (`Order service`, not `The service that handles orders`). Link labels are lowercase single words (`pass`, `fail`, `events`).
- **IDs:** short and uppercase (`GW`, `ORD`, `DB`), unique across the whole diagram.
- **Groups:** use `subgraph id [NN · TITLE]` with an uppercase title and a two-digit number, for example `[01 · INGEST]`. Nest at most one level deep. Number groups in the order the main path crosses them. Keep titles to one or two words: the header shifts each title to the right of the group's center line, so a title longer than about half the group's width spills over the border. If that happens, shorten the title or exempt that group (see Pitfalls).
- **Accent:** one blue focus component per diagram, namely the thing being explained. If everything is highlighted, nothing is.
- **Amber:** only for failure, rejection and alert elements.
- **Styling:** use classes (`classDef`) only. Never style single nodes with `style`, except for the dashed borders on subgraphs.

Always end a flowchart with this block. Remove `classDef` lines for classes the diagram does not use, and add one `style` line per subgraph ID.

```
    classDef default fill:#ffffff,stroke:#1b2a4a,stroke-width:1.2px,color:#1b2a4a
    classDef terminal fill:#1b2a4a,stroke:#1b2a4a,color:#f4f7fb
    classDef accent fill:#ffffff,stroke:#2f6fed,stroke-width:2.4px,color:#2f6fed
    classDef store fill:#e8eef8,stroke:#1b2a4a,stroke-width:1.2px,color:#1b2a4a
    classDef fail fill:#ffffff,stroke:#c2701a,stroke-width:1.6px,stroke-dasharray:4 3,color:#c2701a

    style ingest stroke-dasharray:6 4
    style core stroke-dasharray:6 4
```

### Flowchart example

The standard header goes above this. It shows groups, a decision, a data store, a failure path and the focus component.

```
flowchart TB
    CLIENT([Trading client]):::terminal

    subgraph ingest [01 · INGEST]
        GW[API gateway]
        VAL{Valid order?}
        GW --> VAL
    end

    subgraph core [02 · CORE]
        ORD[Order service]
        BURST[Burst processor]:::accent
        ORD --> BURST
    end

    DB[(Order DB)]:::store
    REJ[Reject + notify]:::fail

    CLIENT ==> GW
    VAL == pass ==> ORD
    VAL -. fail .-> REJ
    BURST --> DB

    classDef default fill:#ffffff,stroke:#1b2a4a,stroke-width:1.2px,color:#1b2a4a
    classDef terminal fill:#1b2a4a,stroke:#1b2a4a,color:#f4f7fb
    classDef accent fill:#ffffff,stroke:#2f6fed,stroke-width:2.4px,color:#2f6fed
    classDef store fill:#e8eef8,stroke:#1b2a4a,stroke-width:1.2px,color:#1b2a4a
    classDef fail fill:#ffffff,stroke:#c2701a,stroke-width:1.6px,stroke-dasharray:4 3,color:#c2701a

    style ingest stroke-dasharray:6 4
    style core stroke-dasharray:6 4
```

## Other diagram types

Use the same standard header. The colors come from it, so only add structure.

**Sequence diagram**
- Start with `sequenceDiagram` and `autonumber`.
- Use `actor` for people and external systems, and `participant` for services. Give short aliases: `participant OMS as Order service`.
- Arrows: `->>` request, `-->>` response, `--)` asynchronous message.
- Use `activate` and `deactivate` for long-running work, `Note over X: text` for context, and `alt`/`else`/`opt`/`loop` for branches.
- At most about 6 participants and 15 messages. Split larger interactions.

**State diagram**
- Use `stateDiagram-v2` with `direction LR`.
- Name states in PascalCase. Label only transitions that need a condition.
- Highlight with classes: `classDef focus fill:#ffffff,stroke:#2f6fed,color:#2f6fed,stroke-width:2px` and `classDef bad fill:#ffffff,stroke:#c2701a,color:#c2701a,stroke-dasharray:4 3`, applied with `class StateName focus`.

**ER diagram**
- Entity names in `UPPER_SNAKE_CASE`, attributes in `snake_case`, with `PK` and `FK` markers.
- Label every relationship with a short verb phrase in quotes.

**Class diagram**
- Show only members that matter for the point being made. Use `..>` for dependencies and `-->` for associations.

**Other types (gantt, pie, mindmap, journey, C4):** use the same header, but they are not tuned for this style. Render and check carefully, and prefer a flowchart when it can express the same thing.

## Pitfalls

- The word `end` in lowercase breaks flowcharts. Write `End` or `"end"` in quotes.
- Labels containing parentheses, brackets, quotes or `#` must be wrapped in double quotes: `A["Order (new)"]`.
- Subgraph IDs have no spaces and must differ from node IDs. The visible title goes in the brackets.
- Do not add emojis or icons to labels.
- If a subgraph title still touches an arrow, or spills over the group's border, shorten the title first. If that is not enough, add a line for that group inside `themeCSS`, using its subgraph ID: `#my-svg-<id> .cluster-label div { transform: none; }` keeps the title centered (use this when two arrows enter on either side of it), or give it a different `translateX` value. `my-svg` is the default SVG id used by mermaid-cli.
- Do not change the font. If JetBrains Mono is not installed on the machine that renders, a fallback monospace font is used silently; this is acceptable.

## Rendering

Requires Node.js and mermaid-cli (`npm install -g @mermaid-js/mermaid-cli`). The output format follows the extension of the `-o` file.

| Format | Command | Notes |
|---|---|---|
| PNG (default) | `mmdc -i diagram.mmd -o diagram.png -b "#f4f7fb" -s 2` | Safest for Confluence, because it always displays the same way. `-s 2` doubles the resolution so text stays sharp when zoomed; use `-s 3` only if text looks soft. |
| SVG | `mmdc -i diagram.mmd -o diagram.svg -b "#f4f7fb"` | Scales without loss, good for browsers and web pages. Labels are HTML text embedded in the SVG, so some tools (Inkscape, PowerPoint, Illustrator, some Confluence versions) may show missing or misplaced text, and the monospace font must be installed on the viewing machine. Tell the user this, suggest checking the file where it will be used, and offer PNG as the fallback. |
| PDF | `mmdc -i diagram.mmd -o diagram.pdf -b "#f4f7fb" -f` | For printing or attaching to documents. `-f` fits the page to the diagram, and the fonts are embedded. |

- **Always set a solid background** with `-b "#f4f7fb"`, whatever the format. A transparent image becomes unreadable on dark backgrounds, such as Confluence dark mode.
- A syntax error makes `mmdc` fail with a parse message. Fix the `.mmd` and render again.

## Checklist before reporting

- [ ] File starts with the standard header, unchanged
- [ ] Only palette colors are used; blue appears on at most one or two nodes, amber only on failure elements
- [ ] Shapes and line types follow the vocabulary
- [ ] Flowcharts use `TB`, have at most about 18 nodes, and every subgraph has a numbered uppercase title and a dashed border
- [ ] Labels are short; no label contains an unquoted special character
- [ ] Rendered in the format the user chose (PNG if they did not say) with a solid background, and the result was inspected: no overlapping text, no title hidden behind an arrow, text legible at page width
