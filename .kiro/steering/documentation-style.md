# Documentation Style Guide

Conventions for writing the tutorial and setup README files in this project.

## File conventions

- Write all content as GitHub-flavored markdown (`.md`).
- Use lowercase, hyphenated file names.
- Prefix each file name with a two-digit order number that reflects the sequence a
  reader should follow, e.g. `00-install-ops-manager.md`, `01-configure-backup.md`.
  Use gaps if helpful (e.g. `10-`, `20-`) so new guides can be inserted later.
- Each guide starts with a single top-level `#` heading that names the guide.
- Add a short intro sentence under the title stating what the reader will accomplish.

## Structure of a tutorial

Order sections so a reader can follow top to bottom:

1. **Overview** — what this guide covers and the end result.
2. **Prerequisites** — required software, versions, access, and resources.
3. **Steps** — numbered, sequential sections. One clear action per step.
4. **Verification** — how to confirm the step or the whole setup worked.
5. **Troubleshooting** (optional) — common errors and fixes.

## Writing rules

- Use numbered lists for sequential steps, bullet lists for non-ordered items.
- Keep sentences short and direct. Address the reader as "you".
- State the expected result after commands that produce important output.
- Call out prerequisites and version requirements explicitly.

## Commands and code

- Put every command or config snippet in a fenced code block with a language hint
  (```bash, ```yaml, ```json, ```javascript).
- Show one logical command per block unless commands are meant to run together.
- Use placeholders in angle brackets for values the reader must supply,
  e.g. `<ops-manager-url>`, `<project-id>`, and explain each placeholder.
- Do not include real credentials, API keys, or secrets. Use obvious placeholders.

## Self-contained guides

- Each guide must stand on its own. A reader should be able to follow it top to bottom
  without opening another guide in this project.
- Do not send the reader to another guide for a step, for example avoid "see
  `10-...md`, step 2, for how to do X". Instead, include the actual instructions inline.
- It is fine to mention that a guide builds on a previous one in the Overview or
  Prerequisites for context (e.g. "this assumes Ops Manager is already installed"), but
  the concrete steps needed to complete the current guide must be present in it.
- When this causes repetition across guides, accept the repetition. Standalone guides
  are worth more than avoiding duplicated instructions.

## Links and references

- Link to official MongoDB Ops Manager documentation for deep dives rather than duplicating it.
- Use descriptive link text: `[Ops Manager installation docs](url)`, not "click here".

## Index and browser preview

- `README.md` is the entry point: it lists every guide in order with a one-line summary.
- `index.html` renders the guides in a browser (sidebar navigation + markdown renderer).
  It fetches the `.md` files, so it must be served over HTTP (`python3 -m http.server`),
  not opened as a file.
- When you add, rename, remove, or reorder a guide, update **both** the `README.md`
  table and the `GUIDES` list in `index.html` so navigation stays in sync.
