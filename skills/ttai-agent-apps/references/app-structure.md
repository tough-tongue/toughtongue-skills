# App structure

How an Agent Desktop app package is laid out, what each file must hold, and how
its README briefs the live agent. How the app reads its data and updates on
screen: [runtime.md](runtime.md).

## Contents

- Package layout
- Files
- README.md
- Worked example: idea board

## Package layout

```text
<app_name>/
├── README.md        # the agent's manual; read with desktop_info
├── package.json     # optional: extra packages from the allowlist
├── src/
│   └── App.tsx      # fixed render code; default-exports the root component
└── data/
    └── board.json   # the file the live agent rewrites
```

There is no manifest: the backend lists the package's files. Tool calls use
package-relative paths (`src/App.tsx`). The app root is always `/<app_name>`;
the live agent's tools address `/<app_name>/data/…` and write only there.

## Files

| File           | Required         | Rule                                                                                                                                                                      |
| -------------- | ---------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `src/App.tsx`  | yes              | Default-exports a React 18 component that takes no props. Other `src/**` modules are free. The live agent never edits `src/`.                                             |
| `data/**`      | for agent writes | The only paths the live agent can write (`<root>/data/`). Read only through `import files from "./desktop-files"`. Keep each file a few KB.                               |
| `README.md`    | yes              | The agent's manual: data contract, controls, rules (see below). The live agent reads it with `desktop_info("<app_name>")`; it is not in the system prompt.                |
| `package.json` | no               | `dependencies` from this allowlist only: `yaml`, `marked`, `mermaid`, `dompurify`, `lucide-react`. React 18 is forced; any other package is dropped and its import fails. |

- No `spec.json`: it is not read, and a write that includes one gets a `warning`.
- Reserved paths: the runtime supplies `src/index.tsx` (mounts `App`),
  `src/desktop-files.ts` (the data API), `src/desktop-data.json`, and
  `index.html`, and rebuilds `package.json`. An app's own file at one of these
  paths is replaced.
- Every file is stored as text. No images or fonts as files: inline SVG, data
  URIs, or a CSS `@import`.

## README.md

The agent's manual, and the only place agent guidance lives. Sections, in order:

```markdown
# Title
One-line description.

## Data
The `data/` files and the shape of each.

## Controls
Which file to write to drive the running app, and how.

## Rules
What the agent must never do.
```

- The agent's prompt carries the app's title, root and description (from
  `create_filebase`). The README is not in the prompt: the agent reads it with
  `desktop_info("<app_name>")`, which returns the README and the file list.
- The agent reads the data files it is about to change, then writes whole files
  under `<root>/data/`. It never edits `src/`.
- There is no `spec.json` and no manifest: the backend lists the files, and the
  root is `/<app_name>`. A write that includes `spec.json` returns a `warning`
  ("spec.json is not read; put agent guidance in README.md"); move the guidance
  into the README and `delete: ["spec.json"]`.

<details>
<summary>Control and rule lines that work</summary>

- Name the file to edit and forbid everything else.
- Describe each data operation in one line: set the topic, add an item, move an
  item (keep its id).
- Say how ids are formed (short, stable) and that the whole file is rewritten
  each time.
- Say "write the file, then speak", so the screen changes as the agent says it.
- If anything is clickable, say the agent never sees clicks and must ask the
  learner what they picked.
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

- Call `idea_board_launch` once when brainstorming starts.
- Drive the board only by rewriting `/idea-board/data/board.json` with `idea_board_write`: send the whole file, then speak.
- Add a card to a column's `cards`; move a card by changing its column, keep its id.

## Rules

- Never touch `src/`.
- Keep ids short and stable (c1, c2). Write the whole file each time.
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
              <p key={card.id}>{card.text}</p>
            ))}
          </section>
        ))}
      </div>
    </div>
  );
}
```

</details>
