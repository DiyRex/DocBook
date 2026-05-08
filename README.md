# DocBook

A personal library of long-form technical books. Read them on the web (you're already on it), or install the **DocBook reader app** on your phone for a polished mobile experience.

---

## 📚 Books

### Systems & Software Engineering

- **[Systems Programming & Software Architecture Foundations with C++](books/systems-programming-cpp/)** — *in progress*
  An architecture-first, runtime-first guide for intermediate developers who want to understand what is *actually* happening when their code runs — and why software is built the way it is. C++ is the teaching language; the lessons transfer to every other language.

---

## 📱 Reader App

DocBook also ships as a native Android app that renders this repo with a clean library, modern typography, paginated chapter swiping, and offline-friendly in-memory caching.

- **App repo** → [DocBook-Mobile-App](https://github.com/DiyRex/DocBook-Mobile-App)
- **Install** → grab the latest APK from the [Releases page](https://github.com/DiyRex/DocBook-Mobile-App/releases).

### Quick start

1. Download the latest `docbook-vX.Y.Z.apk` from the app repo's Releases.
2. On Android, allow installs from your file manager / browser, then open the APK.
3. Launch DocBook → on the welcome screen, enter a repo identifier (e.g. `your-org/your-book` or `https://github.com/your-org/your-book`).
4. Tap **Connect**. The library loads from the repo's `index.json`.
5. Tap a book → tap a chapter → swipe horizontally between chapters, scroll vertically within. Use the bottom bar to jump or open the chapter index.

The app works with **any GitHub repo** that follows the structure documented in [AUTHORING](AUTHORING.md). Point it at this repo, your own, or a fork.

---

## ✍️ Adding Books

The full step-by-step guide is in **[AUTHORING.md](AUTHORING.md)**. In short:

1. Create `books/<your-slug>/` and write Markdown chapters in `part-XX-<slug>/NN-<chapter-slug>.md` files.
2. Register the book in the root **[`index.json`](index.json)** — that's how the reader app discovers titles, parts, chapters, and read-time estimates.
3. Add a one-line entry to the **Books** section above so people browsing on GitHub see it too.
4. Commit and push. Pull-to-refresh on the app's home screen and the new book appears.

---

> See **[ABOUT](ABOUT.md)** for repo layout, reading conventions, and license.
> See **[AUTHORING](AUTHORING.md)** for the book authoring + registration guide.
