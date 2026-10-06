# CLAUDE.md

This repo is Divakar's personal software-engineering notes and books collection — currently
system design, Java internals, and Spring Boot, with more topics added over time. It used to be
split across separate per-topic repos; those were consolidated here so there's one source of
truth going forward.

## How this repo relates to the portfolio

This repo is the content source for the "Learning" section of the portfolio site
(`DivakarVelagacherla/portfolio`). The intent is that committing new notes here should be enough
to have them show up there automatically, without hand-editing the portfolio repo — closer to an
API/CMS relationship than a one-off copy-paste. That's the main reason structure and naming
conventions in this file matter more than they would in a repo nobody else reads: whatever
ingests this content needs the shape to stay predictable across topics.

## Repo layout

- One top-level folder per topic/book — `internals-of-core-java/`,
  `everything-about-spring-and-spring-boot/`, `system-design/`, and so on. New topics get a new
  top-level folder when work on them starts.
- **Folder name = that book/topic's actual title, kebab-cased** (e.g. the book titled "Internals
  of Core Java" lives in `internals-of-core-java/`, not `java/`). The portfolio's Learning page
  derives its display name and routing directly from the folder name, so name it right the first
  time rather than a shorthand you'd want to rename later.
- **Folders are stable once created — don't rename or restructure existing ones.** The whole
  point of this repo feeding the portfolio automatically is that adding a new topic is just a new
  folder; reorganizing existing folders is exactly the kind of churn that setup is meant to avoid.
  If something about an existing folder's structure turns out to be wrong, ask before changing it
  rather than assuming a cleanup pass is welcome.
- Each topic folder has its own `README.md` acting as that topic's table of contents / index.
- The root `README.md` is a short repo-wide index pointing at each topic folder — it should stay
  high-level and not accumulate topic-specific detail (that belongs in the topic's own README).

### Two content shapes, pick based on the source material

This repo currently has two different internal shapes, both intentional:

1. **Book style** (`internals-of-core-java/`, `everything-about-spring-and-spring-boot/`) — the
   source material was a single flowing narrative (Part → Chapter → Section). Split it into
   **one file per major Part/Section**, not one file per chapter or subtopic. Keep the split
   coarse — the goal is "one giant topic per page," not maximum granularity. Each part file keeps
   its original heading levels. The folder's `README.md` holds the book's intro/how-to-read-this
   plus a table of contents linking to each part file in order.
2. **Topic-notes style** (`system-design/basics/`, `system-design/advanced/`) — the source
   material was always a set of discrete, mostly-independent topics. Keep it as many small flat
   files grouped into subfolders by theme, with the folder's `README.md` as a roadmap/checklist
   linking to each file.
3. **Design case-study style** (`system-design/designs/`) — one markdown file per real-world
   system design actually practiced for interviews, flat (no subfolder per design). Any diagrams
   (draw.io exports) live alongside in the same folder, filename matching the design's markdown
   file (e.g. `url-shortener.md` + `url-shortener.png`), referenced from the markdown as a normal
   image link.

When starting a new topic, look at the shape of the source material and pick whichever of these
fits — don't force one style onto content that doesn't match it. If it's genuinely unclear, ask
once rather than guessing.

Note on `system-design/` specifically: it's not a book, it's an interview-prep guide, split by
skill level and purpose — `basics/` (what things are and why, foundational topics), `advanced/`
(same, but for more advanced topics), and `designs/` (basics + advanced applied to real designs,
plus how to talk through them in an interview).

## Book-synthesis workflow — read this before starting a new book from source PDFs

The book in `internals-of-core-java/` cost close to 100k tokens to produce, almost entirely from
reading near-duplicate PDFs in full. Follow this sequence for the next book sourced
from PDFs so it costs a fraction of that:

1. **Hash every PDF before reading any of them**: `md5 *.pdf` (or `shasum`). Files with
   identical hashes are byte-identical — read exactly one copy, skip the rest outright. Do this
   for the whole folder up front, not file-by-file.
2. **`brew install poppler` if it isn't already installed, then run every PDF through
   `pdftotext -layout` into scratchpad `.txt` files, then `md5` *those* text files.** This is
   the real fix for the near-duplicate problem (files that share a base filename pattern like
   `Foo.pdf`/`Foo-1.pdf` but aren't byte-identical because of embedded metadata or one extra
   page) — comparing extracted *text* content costs nothing in model tokens (it's a shell
   command) and reliably groups files by actual prose content instead of guessing from a
   filename pattern or spending a read on a sample page. On the Spring/Spring Boot book, this
   collapsed 18 source PDFs into 7 truly unique documents before a single one was read into
   context. Only read (via the `Read` tool) the one `.txt` file per content-hash group that you
   actually need — never the original PDF once its text has been extracted this way, and never
   more than one file per matching hash group.
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

## "update repo" — standing command

When Divakar says **"update repo"**, treat it as a request to bring whatever was just added into
line with the conventions in this file, without re-explaining what's wrong each time. Do this:

1. Run `git status` to find new/untracked and modified files since the last commit.
2. For each new file, check it against the relevant convention above — topic folder naming
   (kebab-case, matches actual title), the correct content shape for that folder (book/topic-notes
   /design case-study), filename casing and spelling, and image colocation/naming for design
   case-studies (`<design>.md` + `<design>.png`, flat, no `images/` subfolder).
3. Fix naming/placement issues directly (rename, move, fix stray headers that duplicate the
   title, etc.) rather than just flagging them.
4. Update the relevant index files so the new content is actually linked, not just present on
   disk: the topic's own `README.md` (and, for `system-design/designs/`, the top-level
   `system-design/README.md` too, since it also lists designs).
5. Only ask before acting if something is genuinely ambiguous (e.g. the new content doesn't
   obviously fit any of the existing shapes) — don't ask for confirmation on mechanical
   renames/relinks that directly follow the rules already written down here.

## Known preferences (apply without re-asking)

- Consolidated notes should read like a book, not a Q&A list.
- Book-style content gets split coarsely (by Part/Section) if split at all — never one file per
  chapter or subtopic.
- If a new book's source material doesn't obviously fit one of the two shapes above, ask once via
  a clarifying question rather than assuming an existing approach transfers unchanged.
