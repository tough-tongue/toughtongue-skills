# App structure

How an Agent Desktop app package is laid out and described.

## Contents

- Package layout
- `spec.json`
- Code vs data
- Worked example: idea board
- Limits

## Package layout

```text
<app_name>/
├── spec.json        # manifest the host and the agent read
├── src/
│   └── App.tsx      # fixed render code; default-exports the root component
└── data/
    └── board.json   # the control panel the agent edits
```

Tool calls use package-relative paths (`src/App.tsx`). Only the `root`,
`entryFile`, and globs inside `spec.json` carry the `/<app_name>` prefix the
host mounts.

## `spec.json`

| Field                       | Meaning                                                 |
| --------------------------- | ------------------------------------------------------- |
| `appName`                   | Same as the catalog `app_name` (the catalog name wins)  |
| `version`                   | `1`                                                     |
| `title`                     | Display title                                           |
| `root`                      | `/<app_name>`                                           |
| `entryFile`                 | `/<app_name>/src/App.tsx`                               |
| `files`                     | **Required.** Every other package-relative path to load |
| `plannedExtensions`         | `[]`                                                    |
| `description`               | Markdown the agent reads: the shape of the data file    |
| `filesystem.glob`           | `/<app_name>/**/*`                                      |
| `filesystem.editableGlobs`  | The exact data file, e.g. `/<app_name>/data/board.json` |
| `filesystem.preferredPaths` | The one file to rewrite most often                      |
| `filesystem.coreFileGlobs`  | Files the agent must not touch (`src/**`)               |
| `instructions`              | Short imperative rules for the voice agent              |

<details>
<summary>Instruction lines that work</summary>

- Name the file to edit and forbid everything else.
- Describe each data operation in one line: set the topic, add an item, move an
  item (keep its id).
- Say how ids are formed (short, stable) and that the whole file is rewritten
  each time.
- Keep each line imperative and under ~20 words.

</details>

## Code vs data

- **Code** (`src/**`): fixed render logic in a sandboxed React 18 runtime.
  `src/App.tsx` default-exports the root component. Only `react` and `react-dom`
  are available; the runtime supplies its own `package.json`. Code changes take
  effect in new sessions.
- **Data** (one file under `data/`): one small, flat JSON file that `App.tsx`
  imports (`import board from "../data/board.json"`). The agent rewrites the
  whole file, so keep the shape trivial: a title plus a list of items with `id`
  and text. Avoid nesting beyond two levels.

Render code reads the data file and draws it; it never writes it.

## Worked example: idea board

<details>
<summary>spec.json, data/board.json, and src/App.tsx</summary>

```json
{
  "appName": "idea-board",
  "version": 1,
  "title": "Idea Board",
  "root": "/idea-board",
  "entryFile": "/idea-board/src/App.tsx",
  "files": ["src/App.tsx", "data/board.json"],
  "plannedExtensions": [],
  "description": "# Idea Board\n\nColumns of idea cards. Edit only data/board.json.",
  "filesystem": {
    "glob": "/idea-board/**/*",
    "editableGlobs": ["/idea-board/data/board.json"],
    "preferredPaths": ["/idea-board/data/board.json"],
    "coreFileGlobs": ["/idea-board/src/**/*"]
  },
  "instructions": [
    "Edit ONLY /idea-board/data/board.json; never touch src/.",
    "To change the framing question, set the top-level topic.",
    "To add an idea, append a card {id, text} to a column.",
    "To move an idea, remove it from one column and add it to another, keeping its id.",
    "Keep ids short and stable (c1, c2). Write the whole file each time."
  ]
}
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
import board from "../data/board.json";

export default function App() {
  return (
    <div style={{ fontFamily: "system-ui", padding: 24 }}>
      <h1>{board.topic}</h1>
      <div style={{ display: "flex", gap: 16 }}>
        {board.columns.map((col) => (
          <section key={col.id}>
            <h2>{col.title}</h2>
            {col.cards.map((card) => <p key={card.id}>{card.text}</p>)}
          </section>
        ))}
      </div>
    </div>
  );
}
```

</details>

## Limits

- One write: 1–200 files, each up to 1 MB. One read: up to 64 paths.
- During a session the agent can write files up to 128 KB; keep data files to a
  few KB.
- Paths use letters, digits, `.`, `_`, `-`, and `/`; any other character becomes
  `_`.
- App names: lowercase letters, digits, and hyphens; 2–64 characters; starting
  with a letter or digit.
- Plain-text storage: no secrets or personal data.
