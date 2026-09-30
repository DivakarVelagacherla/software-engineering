# CLAUDE.md

This repo is Divakar's personal interview-prep notes collection. Folders are created **one at a
time, per book** — a new topic folder gets added when work on it starts, holding that topic's
source PDFs (interview Q&A dumps, often with duplicate or near-duplicate files exported from the
same source), and is not pre-provisioned ahead of time. The deliverable for each folder is one
consolidated, book-style markdown file at the repo root — flowing narrative prose organized by
topic, not a Q&A crib sheet — modeled on `internals-of-core-java.md` (built from `coreJava/`).

## Repo layout

- `coreJava/` — source PDFs for Core Java → produced `internals-of-core-java.md`
- Other topic folders are added as new books are started; expect the folder list here to change
  over time rather than treating any past list of folder names as current
- Book outputs live at the repo root, named `internals-of-<topic>.md`

## Book-synthesis workflow — read this before starting a new book

`internals-of-core-java.md` cost close to 100k tokens to produce, almost entirely from reading
near-duplicate PDFs in full. Follow this sequence for the next book so it costs a fraction of
that:

1. **Hash every PDF before reading any of them**: `md5 *.pdf` (or `shasum`). Files with
   identical hashes are byte-identical — read exactly one copy, skip the rest outright. Do this
   for the whole folder up front, not file-by-file.
2. **For files that share a base filename but aren't identical by hash** (e.g. `Foo.pdf`,
   `Foo-1.pdf`, `Foo-2.pdf`) — this pattern has meant "same prose content, one extra page" (an
   added table of contents) every time it's shown up so far. Before reading a variant in full,
   read only its first 1–2 pages and diff that against the base file's already-extracted notes.
   If it matches, skip the rest of that file. Do not re-read all N pages of a variant just to
   confirm what page 1 already tells you.
3. **Extract straight to scratchpad notes per source file, in condensed form** — short bullets
   capturing the substance, not verbatim transcription. The scratchpad notes are what actually
   get used to write the book; the raw PDF reads sitting earlier in the conversation should not
   need to be revisited. Write one scratchpad file per source PDF as you finish it, rather than
   holding everything in context until the end.
4. **Write the book in a single pass** from the scratchpad notes once every source is extracted
   — don't re-read the original PDFs or scroll back through conversation history to write it.
   Organize by topic, not by source document or original question order. Merge the same question
   asked at multiple difficulty levels into one deepest treatment instead of repeating it per
   level.
5. Default to flowing narrative prose over Q&A formatting in the output — this was an explicit,
   deliberate choice for `internals-of-core-java.md` (structure/tone closer to a technical book
   like Alex Xu's *System Design Interview* than an interview crib sheet), and it carries forward
   to future books unless told otherwise.

## Known preferences (apply without re-asking)

- Consolidated notes should read like a book, not a Q&A list.
- If a new book's source material doesn't obviously fit the same "dedupe + narrative synthesis"
  approach, ask once via a clarifying question rather than assuming `internals-of-core-java.md`'s
  exact approach transfers unchanged.
