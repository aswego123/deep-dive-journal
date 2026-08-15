# How to Use This Vault

## Folder Structure

```
DeepDiveJournal/
├── Home.md                     ← start here (dashboard / index)
├── README.md                   ← this file
│
├── Clusters/                   ← "theme seasons": big buckets that group Topics
│   ├── AI and Compute.md
│   ├── Decision & Power.md
│   ├── Frontier Science.md
│   ├── Mind & Behavior.md
│   ├── Philosophy of Mind & Self.md
│   └── Thinking Tools.md
│
├── Topics/                     ← one note per idea/concept you're learning
│   ├── Parallelism.md
│   ├── Scaling Laws.md
│   ├── What is Compute.md
│   └── Neuroplasticity/        ← a Topic can grow into a folder
│       ├── Neuroplasticity.md              ← the main topic entry
│       ├── Brain Development and Learning.md
│       ├── Lara Boyd, TED x Vancouver- ... .md   ← source note
│       └── Frontiers in Neuroscience - ... .md   ← source note
│
├── Daily Notes/                ← what you learned / thought each day
│   ├── 2026-07-11.md
│   ├── 2026-07-12.md
│   └── 2026-07-13.md
│
├── Templates/                  ← pre-filled skeletons for new notes
│   ├── Daily Note Template.md
│   ├── Topic Entry Template.md
│   └── Weekly Review Template.md
│
└── Attachments/                ← images and other pasted files
```

### What goes where

| Folder | What it holds | When to add a note |
|---|---|---|
| `Clusters/` | Broad themes (a "season" of study) | Rarely — only when you start a new theme |
| `Topics/` | One idea per note (the core unit) | Every time you learn something worth keeping |
| `Daily Notes/` | Dated journal-style captures | Once a day, on days you learn/think something |
| `Templates/` | Skeletons used by the Templates plugin | Almost never — edit only to change the format |
| `Attachments/` | Images pasted into notes | Automatic (Obsidian drops files here) |

**Growth pattern for a Topic:** start as a single file in `Topics/` (e.g. `Topics/Parallelism.md`). Once it accumulates sub-notes and sources, promote it to a folder (like `Topics/Neuroplasticity/`) with the main entry inside (`Neuroplasticity.md`) plus its supporting notes.

## Import into Obsidian
1. Unzip the folder you downloaded — you'll get a folder called `DeepDiveJournal`.
2. Open Obsidian → click **"Open folder as vault"** (or the folder icon in the vault switcher, bottom-left) → select `DeepDiveJournal`.
3. That's it — it's now a live Obsidian vault.

## Set up Templates (so new entries auto-fill)
1. Go to **Settings → Core plugins** → turn on **Templates**.
2. Settings → Templates → set "Template folder location" to `Templates`.
3. Now, whenever you want a new entry: create a new note in `Topics/`, then `Ctrl/Cmd + P` → "Insert template" → pick "Topic Entry Template."
4. Same for daily entries in `Daily Notes/` using "Daily Note Template," and weekly reviews using "Weekly Review Template."

## Suggested workflow
1. Start in `Home.md` — pick your current theme season (a Cluster).
2. Each time you learn something, make a new note in `Topics/` from the Topic Entry Template.
3. Capture day-by-day thoughts in `Daily Notes/` using the Daily Note Template.
4. Every Sunday, make a Weekly Review note using that template.
5. Link notes to each other using `[[Note Name]]` — this is what makes Obsidian powerful. Over time you'll see your knowledge graph (View → "Open graph view").

## Optional upgrades later
- Install the **Dataview** community plugin to auto-generate lists (e.g. "show me all seedling-status notes").
- Install **Spaced Repetition** plugin if you want to quiz yourself on old entries.

[[Home]]