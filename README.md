# DocBook

A personal library of long-form technical books. Each entry below is a self-contained book with its own table of contents and chapters. Click a title to enter the book.

---

## Books

### Systems & Software Engineering

- **[Systems Programming & Software Architecture Foundations with C++](books/systems-programming-cpp/)**
  An architecture-first, runtime-first guide for intermediate developers who want to understand what is *actually* happening when their code runs — and why software is built the way it is. C++ is the teaching language; the lessons transfer to every other language. *In progress.*

---

## How This Repo Is Organized

```
DocBook/
├── README.md                       <- you are here (book index)
└── books/
    └── <book-slug>/
        ├── README.md               <- book home: TOC, preface link, part navigation
        ├── preface.md
        └── part-XX-<slug>/
            ├── README.md           <- part index
            └── NN-<chapter-slug>.md
```

Each book lives in its own folder under `books/`. Each book's `README.md` is the entry point — open it to see the full table of contents and click into any chapter. Parts (where used) have their own index `README.md` for finer-grained navigation.

---

## Reading Conventions Used Across Books

- All books are written in plain Markdown so they render natively on GitHub and in any Markdown viewer.
- Cross-references between chapters use relative links so the books stay portable.
- "Experiments" sections contain runnable code or shell commands; "Exercises" sections are for you to do.
- "Tradeoffs" tables explicitly list what each design choice costs — there are no universal right answers in engineering.
- "Common Misconceptions" sections call out incorrect mental models the author has seen developers carry for years.

---

## License & Use

Personal study notes. Use freely; if a chapter helps you, the work was worth it.
