# About DocBook

> **[← Back to home](README.md)**

DocBook is a personal library of long-form technical books written in plain Markdown so they render natively on GitHub and in any Markdown viewer.

---

## Repo Layout

```
DocBook/
├── README.md                       <- book index (home)
├── ABOUT.md                        <- this file
└── books/
    └── <book-slug>/
        ├── README.md               <- book home: TOC, preface link, part navigation
        ├── preface.md
        └── part-XX-<slug>/
            ├── README.md           <- part index
            └── NN-<chapter-slug>.md
```

Each book lives in its own folder under `books/`. Each book's `README.md` is the entry point — open it to see the full table of contents and click into any chapter. Parts (where used) have their own index `README.md` for finer-grained navigation.

### Adding a new book

1. Create `books/<book-slug>/` with at minimum a `README.md` (the book's home page with TOC).
2. Add chapter files inside the book's folder, optionally grouped under `part-XX-<slug>/` subfolders.
3. Add a one-line entry to the **Books** list in the top-level `README.md`.

The layout is flat by design — no global build, no toolchain, no dependencies. A book is a folder of Markdown.

---

## Reading Conventions

- **Cross-references** between chapters use relative links so the books stay portable.
- **Experiments** sections contain runnable code or shell commands.
- **Exercises** sections are for the reader to do.
- **Tradeoffs** tables explicitly list what each design choice costs — there are no universal right answers in engineering.
- **Common Misconceptions** sections call out incorrect mental models authors have seen developers carry for years.
- **Navigation footers** at the bottom of each chapter link to the previous chapter, the part index, and the next chapter.

---

## License & Use

Personal study notes. Use freely; if a chapter helps you, the work was worth it.
