# App structure

How an Agent Desktop app package is laid out, what each file holds, and how its
README briefs the live agent. How the app reads data, emits events and updates
on screen: [runtime.md](runtime.md).

## Contents

- Package layout
- Files
- README.md
- Worked example: idea board

## Package layout

```text
<app_name>/
├── README.md        # the agent's manual
├── package.json     # optional: extra packages from the allowlist
├── src/
│   └── App.tsx      # fixed render code; default-exports the root component
└── data/
    └── board.json   # the file the live agent rewrites
```

There is no manifest or `spec.json`. Paths are relative to the app root, with
no leading `/` (`src/App.tsx`, `data/board.json`), in MCP tools and in the
agent's `desktop_*` tools alike.

## Files

| File           | Required         | Rule                                                                                                                                                                      |
| -------------- | ---------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `src/App.tsx`  | yes              | Default-exports a React 18 component that takes no props. Other `src/**` modules are free. The live agent never edits `src/`.                                             |
| `data/**`      | for agent writes | The only paths the live agent can write. Read only through `import files from "./desktop-files"`. Keep each file a few KB: every agent write sends all data files.        |
| `README.md`    | yes              | The agent's manual (below).                                                                                                                                               |
| `package.json` | no               | `dependencies` from this allowlist only: `yaml`, `marked`, `mermaid`, `dompurify`, `lucide-react`. React 18 is forced; any other package is dropped and its import fails. |

- "Required": the write receipt's `missing` lists the file until it exists.
- Reserved: the runtime supplies `src/index.tsx` (mounts `App`),
  `src/desktop-files.ts`, `src/desktop-events.ts` and `index.html`. Never write
  them; your own copies are ignored.
- Every file outside `src/` reaches the app through `./desktop-files`: keep
  briefs and large unrelated files out of the package.
- Files are text. No images or fonts as files: inline SVG, data URIs, or a CSS
  `@import`.

## README.md

The agent's manual. Sections, in order:

```markdown
# Title

One-line description.

## Data

The `data/` files and the shape of each.

## Controls

How to drive the running app, and each event it emits.

## Rules

What the agent must never do.
```

The README is not in the prompt. The agent's five tools take `app` (the
FileBase `name`): `desktop_launch` shows the app, `desktop_info` returns this
README and the file list, `desktop_read` reads a file, and `desktop_write` /
`desktop_delete` change whole files under `data/`.

<details>
<summary>Control and rule lines that work</summary>

- Name the file to edit and forbid everything else.
- Describe each data operation in one line: set the topic, add an item, move an
  item (keep its id).
- Say how ids are formed (short, stable) and that the whole file is rewritten
  each time.
- Say "write the file, then speak", so the screen changes as the agent says it.
- For each emitted event give its name, data, and the agent's reaction; say
  whether it asks for a reply (`reply: true`) or is silent context.
- If something is clickable and not emitted, say the agent never sees it and
  must ask the learner what they picked.
- Keep each line imperative and under ~20 words.

</details>

## Worked example: idea board

<details>
<summary>README.md, data/board.json, and src/App.tsx</summary>

```markdown
# Idea Board

Columns of idea cards the agent fills while brainstorming.

## Data

`data/board.json`: `topic` (the framing question) and `columns[]` of
`{id, title, cards[]}`; a card is `{id, text}`.

## Controls

- Drive the board only by rewriting `data/board.json` with `desktop_write`: send the whole file, then speak.
- Add a card to a column's `cards`; move a card by changing its column, keep its id.
- `star {id}` (silent): the learner starred card `id`. Remember it; do not reply.

## Rules

- Keep ids short and stable (c1, c2).
```

```json
{
  "topic": "How might we make the first week unforgettable?",
  "columns": [
    {
      "id": "ideas",
      "title": "Ideas",
      "cards": [
        {
          "id": "c1",
          "text": "Interactive product tour led by the voice agent"
        }
      ]
    },
    { "id": "parked", "title": "Parked", "cards": [] }
  ]
}
```

```tsx
import files from "./desktop-files";
import { emit } from "./desktop-events";

type Card = { id: string; text: string };
type Board = {
  topic?: string;
  columns?: { id: string; title: string; cards: Card[] }[];
};

// Missing, half-written or wrongly shaped JSON reads as an empty board.
function readBoard(): Board {
  let raw: unknown;
  try {
    raw = JSON.parse(files["data/board.json"] ?? "{}");
  } catch {
    return {};
  }
  const board = raw as Board | null;
  if (!Array.isArray(board?.columns)) return {};
  const columns = board.columns.map((col) => ({
    ...col,
    cards: Array.isArray(col?.cards) ? col.cards : [],
  }));
  return { topic: board.topic, columns };
}

export default function App() {
  // Read on every render: an agent write re-renders this component in place.
  const board = readBoard();
  const page = { fontFamily: "system-ui", padding: 24 };
  if (!board.columns) return <p style={page}>Waiting for the agent…</p>;
  return (
    <div style={page}>
      <h1>{board.topic}</h1>
      <div style={{ display: "flex", flexWrap: "wrap", gap: 16 }}>
        {board.columns.map((col) => (
          <section key={col.id}>
            <h2>{col.title}</h2>
            {col.cards.map((card) => (
              <button
                key={card.id}
                style={{ display: "block", marginBottom: 8 }}
                onClick={(e) => {
                  // A learner action: tell the agent, then free Space for push-to-talk.
                  emit("star", { id: card.id });
                  e.currentTarget.blur();
                }}
              >
                {card.text}
              </button>
            ))}
          </section>
        ))}
      </div>
    </div>
  );
}
```

</details>
