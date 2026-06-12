# agent-skills

A personal collection of reusable **AI agent skills** — small, self-contained packages
that teach an AI agent how to carry out a specific real-world workflow reliably.

Each skill lives in its own folder under [`skills/`](skills/) and ships with:

- a `SKILL.md` — the instructions the agent follows, plus the metadata that tells it
  *when* to use the skill;
- any helper `scripts/` the workflow relies on;
- a packaged `.skill` file you can install directly;
- a short `README.md` describing what it does and how to use it.

## What is a "skill"?

A skill is a portable bundle of instructions (and optional scripts/assets) that an
agent loads on demand. The agent reads the skill's description to decide whether a task
matches, and if so, follows the workflow inside. Think of it as codifying "the right way
to do X" once, so it can be repeated consistently instead of improvised each time.

## Skills in this repo

| Skill | What it does |
|-------|--------------|
| [receipt-expense-extractor](skills/receipt-expense-extractor/) | Turns PDFs that each contain many receipts into a clean expense spreadsheet (date, description, total, source file), with OCR and a human-in-the-loop review step. |

## Installing a skill

Each skill folder contains a packaged `*.skill` file (a zip archive of the skill
directory). Download it and install it in a compatible agent environment — for example,
in Claude's Cowork mode, open the `.skill` file and choose **Save skill**. Once
installed, the skill triggers automatically when your request matches its description.

You can also just read the `SKILL.md` to see exactly how the workflow operates — the
instructions are plain Markdown.

## Repository layout

```
agent-skills/
└── skills/
    └── <skill-name>/
        ├── SKILL.md           # agent instructions + triggering metadata
        ├── README.md          # human-facing overview
        ├── scripts/           # helper scripts the skill uses
        └── <skill-name>.skill # packaged, installable bundle
```

Adding a new skill is just a matter of dropping another folder under `skills/` following
the same shape.
