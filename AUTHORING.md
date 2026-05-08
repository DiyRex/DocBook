# Authoring Guide — Adding & Registering Books

> **[← Back to home](README.md)** · **[About →](ABOUT.md)**

This is a complete walkthrough for adding a new book to this repo so the **DocBook reader app** picks it up automatically. There are two halves to "adding a book":

1. **Write the content** — drop Markdown files in a folder under `books/`.
2. **Register it** — add an entry to [`index.json`](index.json) at the repo root so the app's library, TOC, and pagination know about it.

If you skip step 2, the app will not see your book.

---

## TL;DR Checklist

- [ ] Pick a `book-slug` (kebab-case, lowercase): e.g. `database-internals`
- [ ] Create `books/<book-slug>/` with a `README.md`, an optional `preface.md`, and one or more `part-XX-<slug>/` folders containing `NN-<chapter-slug>.md` files
- [ ] Add a book object to `index.json` → `books[]` describing the title, parts, and chapters with their **paths and read-time estimates**
- [ ] Commit and push to `main`
- [ ] In the app, pull-to-refresh on the home screen — the new book appears

---

## 1. Filesystem Layout

The repo follows this structure. Every existing book follows it; new books should too.

```
DocBook/
├── README.md                      <- top-level books index (human-readable)
├── ABOUT.md                       <- repo conventions
├── AUTHORING.md                   <- this file
├── index.json                     <- machine-readable book index (app reads this)
└── books/
    └── <book-slug>/
        ├── README.md              <- book home (human-readable TOC)
        ├── preface.md             <- optional frontmatter
        └── part-XX-<slug>/
            ├── README.md          <- part index (human-readable)
            └── NN-<chapter-slug>.md
```

### Naming rules

- **`<book-slug>`**: kebab-case, lowercase, no spaces, descriptive. *Examples:* `systems-programming-cpp`, `database-internals`, `react-rendering-deep-dive`. Avoid version numbers in the slug — the slug is forever.
- **`part-XX-<slug>`**: `XX` is two-digit zero-padded part number; slug describes the part. *Example:* `part-01-what-programming-is`.
- **`NN-<chapter-slug>.md`**: `NN` is two-digit zero-padded chapter number, slug is the chapter title kebab-cased. *Example:* `03-how-os-executes-programs.md`.

The numeric prefixes serve a dual purpose: they make filesystem listings sort in reading order, and they are what humans see when browsing the repo on GitHub.

### Cross-document links inside a chapter

Use **relative links** so the docs work both on GitHub *and* in any Markdown viewer:

```markdown
See [Chapter 2](02-cpu-ram-stack-heap-registers.md) for details.
See the [part overview](README.md).
See the [preface](../preface.md).
```

The DocBook app's reader navigates linearly via the bottom prev/next buttons, so cross-chapter links inside the body text are mostly informational. The **app's chapter list comes from `index.json`** — not from these inline links.

---

## 2. Writing a Chapter

Every chapter file should follow the structure used by existing chapters. **The DocBook reader strips the redundant top-of-file `# Chapter N — Title` header and the bottom Prev/Up/Next nav block** when rendering, because the app provides its own chapter heading and navigation chrome. So leave them in for GitHub readability — the app cleans them up on its end.

### Chapter template

```markdown
# Chapter N — Chapter Title

## Learning Objectives

By the end of this chapter you will be able to:

1. ...
2. ...

This chapter is about ...

---

## N.1 First Section

...

## N.2 Second Section

...

### N.2.1 A Subsection

...

---

## N.X Tradeoffs

| Choice | Cost | Benefit |
|---|---|---|
| ... | ... | ... |

---

## N.Y Common Misconceptions

| Misconception | Reality |
|---|---|
| ... | ... |

---

## N.Z Exercises

1. ...

---

## What's Next

A short transition to the next chapter.

---

**[← Previous: Chapter N-1 — Title](NN-prev.md)** · **[Up: Part X](README.md)** · **[Next: Chapter N+1 — Title →](NN-next.md)**
```

### Style conventions

- **Markdown only**, GitHub-flavored. Tables, fenced code blocks, blockquotes, footnote-style links all work.
- **Code fences** can specify a language for highlighting on GitHub: ` ```cpp `, ` ```python `, ` ```bash `. The reader app renders them with a dedicated code-block style regardless of language.
- **Long-form is fine.** Aim for 4–8k words per chapter for technical depth; the reader's font scaling and pagination handle it. Use `## ` for major sections, `### ` for subsections.
- **Tables**: keep them narrow enough to render reasonably on a phone (3–4 columns max).
- **Diagrams**: ASCII inside fenced code blocks render fine. For images, stash them under `books/<slug>/assets/` and reference relatively (`![](assets/diagram.png)`); the app renders network-fetched images.

---

## 3. Registering the Book in `index.json`

This is the step most easily forgotten. The reader app loads `index.json` at startup and uses it for the library list, book covers, table of contents, and pagination. **Markdown files alone are not enough** — they must be referenced from the index.

### Top-level shape

```json
{
  "schema": 1,
  "title": "DocBook",
  "tagline": "A personal library of long-form technical books.",
  "books": [
    { /* book object */ },
    { /* another book */ }
  ]
}
```

### A book object

```json
{
  "slug": "database-internals",
  "title": "Database Internals: How Storage Engines Really Work",
  "shortTitle": "Database Internals",
  "subtitle": "From B-Trees to LSM-Trees and beyond",
  "description": "An end-to-end walk through the design of modern OLTP and OLAP storage engines.",
  "tags": ["Databases", "Storage", "Systems"],
  "accentColor": "#7C3AED",
  "icon": "book",
  "status": "in-progress",
  "path": "books/database-internals",
  "frontmatter": [
    { "title": "Preface", "path": "preface.md", "minutes": 5 }
  ],
  "parts": [
    {
      "slug": "part-01-storage-foundations",
      "title": "Part 1 — Storage Foundations",
      "summary": "How disks, pages, and buffer pools interact.",
      "chapters": [
        {
          "n": 1,
          "title": "How Disks See Your Data",
          "subtitle": "Sectors, blocks, and the block I/O contract",
          "path": "part-01-storage-foundations/01-how-disks-see-your-data.md",
          "minutes": 28
        }
      ]
    }
  ]
}
```

### Field reference

| Field | Required | Notes |
|---|---|---|
| `slug` | yes | Kebab-case identifier, must match the folder name under `books/`. |
| `title` | yes | Full title, shown on the book detail page and home card. |
| `shortTitle` | no | Used in tight headers (book detail app bar, reader). Defaults to `title` if omitted. |
| `subtitle` | no | One-line tag below the title. Empty string is fine. |
| `description` | no | 1–3 sentences shown on the book detail page. |
| `tags` | no | Array of short strings. Up to 4 are shown on the home card. |
| `accentColor` | no | Hex string `#RRGGBB`. Drives the book cover gradient, chip color, code accent in the reader. Defaults to `#1E3A5F`. |
| `icon` | no | Reserved for future per-book icons; currently always renders the open-book glyph. |
| `status` | no | `"in-progress"` (renders a WIP badge), `"complete"`, or `"draft"`. Defaults to `"in-progress"`. |
| `path` | yes | Repo-relative folder of the book, e.g. `books/database-internals`. |
| `frontmatter` | no | Array of `{ title, path, minutes }` for non-numbered intro pages (preface, foreword). Optional. |
| `parts` | yes | Array of part objects (see below). A book with one part is fine — just one entry. |

### A part object

| Field | Required | Notes |
|---|---|---|
| `slug` | yes | Kebab-case folder name under the book. |
| `title` | yes | Shown above the part's chapter list and inside the reader as the running part name. |
| `summary` | no | 1–2 sentences shown under the part heading. |
| `chapters` | yes | Array of chapter objects (see below). |

### A chapter object

| Field | Required | Notes |
|---|---|---|
| `n` | yes | Integer chapter number. The reader displays it prominently. |
| `title` | yes | Shown as the chapter title in the TOC and reader. |
| `subtitle` | no | One-line summary; shown under the chapter title in the reader header and TOC. |
| `path` | yes | **Book-relative** path to the markdown file, e.g. `part-01-storage-foundations/01-how-disks-see-your-data.md`. The full URL the app fetches is `<repo-raw-base>/<book.path>/<chapter.path>`. |
| `minutes` | no | Estimated read time in minutes. Used for the "X min read" pill and the book's total time. Defaults to 0 if omitted. A reasonable estimate: word count ÷ 250. |

### Chapter ordering

The reader navigates chapters **in the order they appear in `parts[].chapters[]`**. The `n` field is purely cosmetic — the array order is what determines prev/next.

---

## 4. Putting It All Together — Worked Example

Adding a new 3-chapter book called `kafka-from-scratch`:

### Step 1: create the folders and files

```
books/kafka-from-scratch/
├── README.md
├── preface.md
└── part-01-the-log/
    ├── README.md
    ├── 01-what-a-log-is.md
    ├── 02-segments-and-indexes.md
    └── 03-the-broker-loop.md
```

### Step 2: write the chapters using the template above

### Step 3: register in `index.json`

Add a new entry inside the top-level `books` array:

```json
{
  "slug": "kafka-from-scratch",
  "title": "Kafka From Scratch: Building a Log-Structured Broker",
  "shortTitle": "Kafka From Scratch",
  "subtitle": "What's actually in the log, the index, and the broker loop",
  "description": "Re-derives Kafka's data model and broker architecture from first principles by building a small clone in Go.",
  "tags": ["Kafka", "Distributed Systems", "Go"],
  "accentColor": "#0F766E",
  "status": "in-progress",
  "path": "books/kafka-from-scratch",
  "frontmatter": [
    { "title": "Preface", "path": "preface.md", "minutes": 4 }
  ],
  "parts": [
    {
      "slug": "part-01-the-log",
      "title": "Part 1 — The Log",
      "summary": "The append-only log is the heart of Kafka. Three chapters to build one.",
      "chapters": [
        { "n": 1, "title": "What A Log Is",         "path": "part-01-the-log/01-what-a-log-is.md",        "minutes": 18 },
        { "n": 2, "title": "Segments and Indexes",  "path": "part-01-the-log/02-segments-and-indexes.md", "minutes": 22 },
        { "n": 3, "title": "The Broker Loop",       "path": "part-01-the-log/03-the-broker-loop.md",      "minutes": 26 }
      ]
    }
  ]
}
```

### Step 4: also link it from the human README

Add a line to the **Books** section in [`README.md`](README.md) so people browsing on GitHub see it. Example:

```markdown
- **[Kafka From Scratch: Building a Log-Structured Broker](books/kafka-from-scratch/)** — *in progress*
  Re-derives Kafka's data model and broker architecture from first principles by building a small clone in Go.
```

### Step 5: commit and push

```bash
git add books/kafka-from-scratch index.json README.md
git commit -m "Add Kafka From Scratch (Part 1, Ch 1-3)"
git push
```

### Step 6: pick it up in the app

Open the DocBook app → pull-to-refresh on the home screen → the new book appears. Tap it to see the cover, parts, and chapters; tap any chapter to read.

---

## 5. Updating an Existing Book

- **Adding chapters to an existing part**: append to `parts[i].chapters[]` and add the file. Existing chapter numbers and the rest of the array keep their position.
- **Adding a new part**: append to `parts[]` with its own folder.
- **Renaming a chapter**: change `title` (and optionally `subtitle`) in `index.json`. If you also rename the file, update `path`. Be aware: people who bookmark via direct GitHub links will hit dead links.
- **Marking a book complete**: change `status` to `"complete"` to remove the WIP badge.
- **Read-time tuning**: estimate by `word_count / 250`. Easier to update later as you write.

The app caches `index.json` and chapter bodies for 10 minutes per session (RAM only). To force-refresh, pull down on the home or chapter screen, or clear the cache from Settings.

---

## 6. Validation

A quick sanity check before pushing:

```bash
# Make sure index.json parses
python3 -c "import json; json.load(open('index.json'))"

# Make sure every chapter path actually exists on disk
python3 - <<'PY'
import json, os, sys
idx = json.load(open('index.json'))
ok = True
for b in idx['books']:
    base = b['path']
    for f in b.get('frontmatter', []):
        p = os.path.join(base, f['path'])
        if not os.path.exists(p):
            print(f"missing: {p}"); ok = False
    for part in b['parts']:
        for c in part['chapters']:
            p = os.path.join(base, c['path'])
            if not os.path.exists(p):
                print(f"missing: {p}"); ok = False
sys.exit(0 if ok else 1)
PY
```

If all paths exist and the JSON parses, the app will load it cleanly.

---

## 7. FAQ

**Q. Do I have to register a book in `index.json` for it to render?**
Yes. The app does not crawl folders; it reads `index.json` and follows the explicit paths.

**Q. Can I have books with no parts?**
The schema requires at least one part. If your book is a single linear narrative with no logical division, just give it one part called `"Main"` and put all chapters there.

**Q. Can a chapter live outside its part folder?**
Yes — `chapter.path` is just a book-relative path, so `chapter.path = "extras/glossary.md"` works even if the chapter conceptually belongs under a part. Folder structure is *convention* for humans; the app trusts the JSON.

**Q. Do chapter `n` numbers have to be sequential?**
No, they're cosmetic. But sequential numbering matches reader expectations and matches the file numeric prefix.

**Q. How do I add an image / diagram?**
Stash under `books/<slug>/assets/` and reference relatively. The reader fetches images over the network when displayed.

**Q. Can I use math (LaTeX) or Mermaid diagrams?**
The reader uses the `flutter_markdown` baseline — no LaTeX or Mermaid. Use ASCII diagrams in fenced code blocks (they render with the code-block styling) or pre-rendered images.

**Q. How do I delete a book?**
Remove its entry from `index.json` and (optionally) the `books/<slug>/` folder.

---

> Questions or improvements to this guide? Open an issue or PR.
