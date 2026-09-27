# Notenoid — Feature Reference

**Companion to** `Notenoid-v11-Code-Analysis.md`
**Source:** `notenoid@11.0.0`
**Scope:** every user-facing feature, with the code that implements it.

This document is organised by *feature* rather than by *file*. Each section
answers: what it does, how it works, where it lives, and what it touches.

---

## Contents

1. [Notes — the core object](#1-notes--the-core-object)
2. [The Markdown editor](#2-the-markdown-editor)
3. [The Markdown toolbar](#3-the-markdown-toolbar)
4. [Images](#4-images)
5. [The attachments grid](#5-the-attachments-grid)
6. [Image grids in the note body](#6-image-grids-in-the-note-body)
7. [Image sizing](#7-image-sizing)
8. [Checklists](#8-checklists)
9. [Math / formulas](#9-math--formulas)
10. [Find in note](#10-find-in-note)
11. [Organisation — folders, labels, pin, archive, trash](#11-organisation)
12. [The Vault](#12-the-vault)
13. [Views & navigation](#13-views--navigation)
14. [Grid & list, drag to reorder](#14-grid--list-drag-to-reorder)
15. [Search](#15-search)
16. [Themes](#16-themes)
17. [Appearance & personalisation](#17-appearance--personalisation)
18. [Import / Export](#18-import--export)
19. [Platform behaviour](#19-platform-behaviour)
20. [Settings reference](#20-settings-reference)

---

## 1. Notes — the core object

A note is a **real folder on disk** containing a real `node.md`. There is no
database, no server, and no JSON note store. `node.md` is the single source of
truth for both content *and* metadata.

```
<Root>/
├── Archive_Work_errands_Shopping List/
│   ├── node.md              ← YAML front-matter + Markdown body
│   └── attachment/          ← created only when attachments exist
│       ├── receipt.png
│       └── loop.gif
├── Notes/
│   └── node.md
└── .notes/
    └── config.json          ← folder emoji/tints + label names ONLY
```

### Every piece of note state and where it lives

| State | Stored in | Front-matter key |
|---|---|---|
| Title | `node.md` | `title` (and in the folder name) |
| Body | `node.md` | *(the markdown below the fence)* |
| Stable ID | `node.md` | `id` (UUID) |
| Type (text / list) | `node.md` | `type` |
| Colour | `node.md` | `color` |
| Pinned | `node.md` | `pinned` |
| Archived | `node.md` | `archived` |
| Trashed | `node.md` | `trashed` |
| Vaulted | `node.md` | `vaulted` |
| Folder (genre) | `node.md` | `folder` |
| Labels | `node.md` | `labels: ["a", "b"]` |
| Manual position | `node.md` | `order` |
| Created / updated | `node.md` | `created`, `updated` |
| Images | `<note>/attachment/*` | — |
| Checklist items | the markdown body as `- [ ]` / `- [x]` | — |
| Folder emoji + tint | `.notes/config.json` | — |
| The list of label names | `.notes/config.json` | — |
| App settings | localStorage `notes-app-settings-v1` | — |

### The folder name encodes metadata

```
[Archive_]FolderName(_LabelName…)_NoteTitle
```

Recomputed on **every save**. Change the title, archive it, move it to a
folder, or add a label → the folder is renamed to match. The `title` inside
`node.md` is never modified, so the human title stays exactly as typed.

Forbidden characters are replaced with **full-width look-alikes** so titles
stay readable: `/ → ／`, `\ → ＼`, `: → ：`, `* → ＊`, `? → ？`, `" → ＂`,
`< → ＜`, `> → ＞`, `| → ｜`. Collisions become `Name (2)`, `Name (3)`…

> **On the web and OPFS, a folder rename is a full copy + delete**, because
> the File System Access API can only rename *files*. On Android it is a cheap
> native `renameTo`. The app is identical from the user's side either way.

### New notes don't touch the disk until they matter

A brand-new note lives **in memory only** and materialises to disk on a
**500 ms debounce** once it has real content. An abandoned empty note never
creates a folder. Closing the editor or backgrounding the app materialises it
immediately.

### Saving

| Trigger | What happens |
|---|---|
| you type | textarea updates instantly; the store is updated at most every 140 ms |
| 600 ms of quiet | the note is written to disk |
| editor closed / app hidden / page unloading / app destroyed | everything pending is flushed immediately |
| a write fails | retried up to 6 times with growing delays, then an error toast — **your text is never dropped** |

---

## 2. The Markdown editor

> **Design principle: the document is always plain markdown.**
> Edit mode is a transparent textarea. Preview mode is a renderer. Toggling
> between them never transforms the text.

### Two modes

| | Edit | Preview |
|---|---|---|
| What you see | raw markdown in a textarea | rendered HTML |
| Default | — | ✅ **preview** is the default |
| Interactive | text only | checkboxes, links, code-copy, image sizing, lightbox |

**The mode persists across note switches and remounts.** A module-level
holder stores it outside React specifically so that a note materialising on
disk can't flip you back to the other mode mid-sentence.

### Layout of the editor screen

```
┌──────────────────────────────────────────┐
│ ← ←    📌 🔍                     ✕        │  header
├──────────────────────────────────────────┤
│ [ 👁  B I U S `  # ## ### ¶  …  ↶ ↷ ]     │  toolbar (sticky)
├──────────────────────────────────────────┤
│ ┌──────────────────────────────────────┐ │
│ │ 🖼 🖼 🖼              [⊞ view all]   │ │  ◀── ATTACHMENTS GRID (only if
│ └──────────────────────────────────────┘ │      the note has images)
│ Title…                                   │
│ ──────────────────────────────────────── │
│                                          │
│  rendered markdown / textarea            │
│                                          │
├──────────────────────────────────────────┤
│ 🎨 🖼 ☑ 🏷 📁 📦 ⋯                  Done  │  footer
└──────────────────────────────────────────┘
```

### Switching modes never loses your place

This is the most carefully engineered part of the editor. Edit mode and
preview mode have completely different heights, so a naive scroll restore
would land you somewhere unrelated.

The system gives every rendered block a **`data-src-line`** attribute — the
markdown source line it came from. That yields an **exact** mapping between
textarea lines and preview blocks:

- a probe point is placed at **35%** down the viewport
- before switching, the exact content at that probe is recorded
- after switching, the matching block is pinned back to that same probe
- a settle loop keeps holding the position while images/fonts/code finish
  laying out, and **hands control back within 6 px** the moment you scroll

Result: the same paragraph stays at the same screen position, every time, both
directions, on web and on Android.

### Text scaling and fonts

| | |
|---|---|
| **Note text size** | 70–170 %, scales the editor, the preview and note cards together |
| **Note font** | 14 choices incl. monospace |
| **App text size** | 70–150 %, scales everything *except* the note body |

Heading sizes in the preview are `em`-based, so they scale proportionally with
the note text size.

---

## 3. The Markdown toolbar

23 buttons + 2 popovers, in 6 groups, horizontally scrollable.

### Group 1 — view

| Button | Action |
|---|---|
| 👁 / ✎ | toggle **preview ↔ edit** |

### Group 2 — inline formatting

| Button | Wraps the selection in |
|---|---|
| **B** | `**…**` |
| *I* | `*…*` |
| <u>U</u> | `<u>…</u>` |
| ~~S~~ | `~~…~~` |
| `` ` `` | `` `…` `` |

Caret behaviour: with **no selection**, the caret lands *between* the marks so
you can just start typing. With a **selection**, it lands *after* the whole
formatted text. The selection is always collapsed afterwards so your next
keystroke doesn't replace what you just formatted.

### Group 3 — blocks

| Button | Effect |
|---|---|
| H1 / H2 / H3 | prefixes the line with `# ` / `## ` / `### ` |
| ¶ Body text | strips the heading / list / quote marker |
| — | *divider* | |

Headings behave as a **radio group**: pressing H2 on an already-heading line
changes it rather than stacking `## ##`.

### Group 4 — lists & blocks

| Button | Effect |
|---|---|
| • Bullet | `- ` |
| 1. Numbered | `1. ` — **auto-increments** as you press Enter |
| ☑ Checklist | `- [ ] ` |
| ❝ Quote | `> ` |
| { } Code block | ` ``` ` fence |

**Enter behaviour:**
- on a list item → continues the list (numbering increments)
- on a task item → adds another unchecked task
- on an **empty** list item → **exits the list** instead of making noise

### Group 5 — insert

| Button | Action |
|---|---|
| 🔗 Add link | popover: optional link text + URL (Enter applies) |
| 🔗✖ Remove link | keeps the text, drops the link |
| 🖼 **Add image** | opens the file picker → **saves into attachments** |
| 🖼🖼 **Insert at cursor** | picker: new files or existing attachments → places them in the body (1 = single, many = grid) |
| — Divider | inserts `---` |
| ▦ Table | inserts a 3×3 table skeleton |
| ⌫ Clear formatting | strips markdown syntax, **keeps the words** |
| ⧉ Multiple formats | checkbox grid — apply bold + italic + H2 in one go |

### Group 6 — history

| Button | Shortcut |
|---|---|
| ↶ Undo | `Ctrl/Cmd + Z` |
| ↷ Redo | `Ctrl/Cmd + Shift + Z`, `Ctrl/Cmd + Y` |

Undo is a custom snapshot stack (300 entries, typing coalesced into ~650 ms
chunks, every toolbar action its own step) because the browser's native undo
never sees toolbar or programmatic changes and is broken by a controlled
textarea.

### Clear formatting — what it keeps

| Removed | **Kept** |
|---|---|
| `**bold**`, `*italic*`, `~~strike~~` | code blocks (verbatim) |
| `<u>` tags | code span *text* (backticks removed, content kept) |
| `#` headings, `>` quotes | images |
| `- `, `1. `, `- [ ] ` markers | attachment links |
| `---` rules | tables |
| `[text](url)` → `text` | |
| leftover `*` `~` markers | |

With a selection, only the selection is cleared. Without one, the whole note.
A mid-line selection never loses the list marker at the start of its line, and
`snake_case_names` are never mangled.

### Paste

Pasting images (a clipboard with **only** images) is intercepted and saved into
attachments. Mixed text+image pastes behave normally.

### Keyboard shortcuts

| Keys | Action |
|---|---|
| `Ctrl/Cmd + Z` | undo |
| `Ctrl/Cmd + Shift + Z` / `Ctrl/Cmd + Y` | redo |
| `Enter` | continue / exit a list |
| `Shift+Enter` | plain newline |
| `Esc` | close the editor (a popover closes first) |
| `↑` / `↓` / `Enter` / `Shift+Enter` | move between find-in-note matches |

### What's supported

| | |
|---|---|
| ✅ | Headings H1–H6, bold, italic, underline, strikethrough, inline code, code blocks, bullet/numbered/task lists, tables, blockquotes, horizontal rules, links, images, **math**, raw HTML |
| ❌ | Syntax highlighting, Mermaid, emoji shortcodes, definition lists |

> Code blocks get a **Copy** button but no colouring.

---

## 4. Images

### The one rule that governs everything

> **An upload never auto-inserts into the note body.**
> Images always land in the attachments area first. Placing one in the body is
> a separate, explicit second action.

There are five ways to add an image, and **all five go to attachments only**:

| # | Where |
|---|---|
| 1 | Toolbar **🖼 Add image** |
| 2 | Footer **🖼** popover (Upload tab) |
| 3 | Footer **🖼** popover (**By URL** tab) |
| 4 | **Pasting** an image into the editor |
| 5 | The composer "new note with image" shortcut |

Only **two** things put an image into the body:

- the **⊕ button on a thumbnail** in the attachments grid
- the **🖼🖼 Insert image at cursor** picker in the toolbar

> **Why:** earlier versions had an "insert" mode flag that could survive a
> *cancelled* picker, silently duplicating a phantom image into the next
> unrelated note. The mode flag was removed entirely — auto-insertion is now
> structurally impossible.

### Compression on the way in

| Rule | Behaviour |
|---|---|
| **GIF** | passed through untouched (animation preserved), hard cap **8 MB** |
| **≤ 1600 px** | stored as-is — no re-encode, no quality loss |
| **> 1600 px** | scaled down to fit 1600×1600 |
| **PNG** | re-encoded as PNG (transparency kept) |
| **everything else** | re-encoded as JPEG at quality 0.82 |
| **any error** | the original file is stored |

### Viewing, editing, removing

Clicking any image — in the attachments grid, the gallery, or inline in the
preview — opens the **lightbox**:

| Button | Action |
|---|---|
| ✎ Edit image | crop / draw / add text, then save **in place** (the filename never changes, so no markdown reference breaks) |
| ⬇ Download | saves the file to your device |
| ⊘ Remove from note | strips it from the body, **keeps the file** |
| 🗑 Delete image | strips it from the body **and** deletes the file |

Removing one image from a multi-image grid collapses the grid so you never
get an empty bordered box.

---

## 5. The attachments grid

A **separate grid that sits at the top of the editor**, above the title and
above the body.

It appears **only when the note has at least one image** — text-only notes
never see it.

### Layout

| Property | Value |
|---|---|
| Position | top of the editor, above the title input |
| Shown when | `attachments.length > 0` |
| Max height | **128 px** — it is capped and scrolls internally, so a note with 200 images still opens instantly |
| Scroll | **vertical only**; horizontal is hidden, so the grid reflows into fewer columns instead |
| Columns | `repeat(auto-fill, minmax(5rem, 1fr))` — **fully responsive**, min tile 80 px, the last row stretches |
| Gap | 10 px |
| Thumbnail | 80×80 px, rounded, **square crop** |
| Top-right | a **⊞ view-all** button opening a 2/3-column gallery |

### Each thumbnail has four controls

| Control | Where | Action |
|---|---|---|
| the image | whole tile | open the **lightbox** |
| ✎ | top-left | **edit** the image |
| ✕ | top-right | **delete** the attachment |
| ⊕ | bottom-left | **insert into the note body** |
| `GIF` badge | bottom-right | shown for animated GIFs |

While an image is still loading, its tile shows a soft pulsing skeleton.

### Adding more below

The grid grows when you use any of the five upload routes above. The most
common is the toolbar's **🖼 Add image** button — it is literally labelled
*"Add image (saved to attachments)"*, which tells you where it goes.

The footer **🖼** popover offers two tabs:

- **Upload** — "Choose images or GIFs" · multiple allowed ·
  *"Saved into the note's attachment folder"*
- **By URL** — paste a link and the image is fetched and stored locally

---

## 6. Image grids in the note body

When you place images into the body, the app decides between a single image
and a grid based on **how many you selected**.

### The rule

| How many | What you get | Markdown written |
|---|---|---|
| **1** | a **plain markdown image** | `![alt](attachment/photo.png)` |
| **2, 3 or 4** | a **2-column grid** | `<div class="img-grid" data-cols="2">…</div>` |
| **5 or more** | a **3-column grid** | `<div class="img-grid" data-cols="3">…</div>` |

**Why the split at 4?** Up to four images read comfortably as a 2×2 block. A
2-column grid of 8 images would become an absurdly tall column, so five and up
switch to 3 columns.

### Inserting them

Use the toolbar's **🖼🖼 Insert image at cursor** button. It offers two
sources:

1. **Choose new images** — pick one or many; they're stored as attachments and
   inserted immediately
2. **From this note** — tick existing attachments in a 4-column picker, then
   press *Insert N*

Either way the images are placed **at your cursor**, with sensible blank lines
around them, and your caret lands right after them.

### How the grid looks

```css
.img-grid {
  display: grid;
  grid-template-columns: repeat(2, minmax(0, 1fr));   /* 3 with data-cols="3" */
  gap: 8px;
  width: 100%;
  max-width: min(100%, var(--note-media-grid-max));   /* the "Image grid width" setting */
  margin: 0.6rem 0;
}
.img-grid img {
  width: 100%;
  aspect-ratio: 1 / 1;      /* every tile is square */
  object-fit: cover;         /* square crop, never distorted */
  border-radius: 10px;
}
```

- every tile is a **square crop** whatever the source aspect ratio — a portrait
  and a landscape in the same grid render identically sized
- tiles never overlap, never get clipped
- a gentle 1.5 % scale on hover
- on phones under 480 px the gap tightens to 6 px and the corners soften

**Global width control:** Settings → Appearance → **Image grid width**
(40–100 %, default **75 %**) caps how wide any grid can be. Individual grids
can be narrower still via the per-image sizing (next section).

### Why plain HTML instead of a markdown extension?

Because it **round-trips perfectly**. The `<img src>` attributes are rewritten
back to `attachment/…` paths on save, so `node.md` stays ordinary, portable
markdown that Obsidian and every other tool can still open.

---

## 7. Image sizing

Every image and every grid can be sized **individually** — the size is stored
in the note itself and travels with the file.

### Opening the size control

- **long press** the image for ~half a second, **or**
- **right-click** / long-press it on desktop

The old always-visible button was removed on purpose so that a large image
never puts a big control over your text while you're reading.

### The menu

| Preset | Width |
|---|---|
| **S** | 35 % |
| **M** | 55 % |
| **L** | 75 % |
| **Full** | 100 % |
| **Auto** | natural width |

Plus a **slider from 20 % to 100 %** in 5 % steps for anything in between.

### How it's stored

| Object | Stored as | Example |
|---|---|---|
| a single image | `size=NN` in the image's markdown title | `![photo](attachment/a.png "size=75")` |
| a grid | `data-size="NN"` on that grid only | `<div class="img-grid" data-cols="2" data-size="50">` |

Sizing **one** image never affects any other. Auto simply means "no size
recorded" — the image renders at its natural width, capped at 60 % of the
viewport height so it can't dominate the note.

The menu is viewport-anchored and clamped, so resizing an image never makes
the menu jump or slide off-screen — and your scroll position is held exactly.

---

## 8. Checklists

A note is either **text** or a **list**. The footer button switches between
them, converting the body in the same step.

### Storage

Checklists are **plain markdown task lines** — `- [ ] item`, `- [x] item` —
so any other markdown tool can read them. There is no proprietary format and
no separate checklist database.

### In the preview

Checkboxes are **live**. Ticking one writes straight back to the note
immediately, and the viewport stays exactly where it was.

### Behaviour

| | |
|---|---|
| **Enter** on a task | adds the next unchecked task |
| **Enter** on an empty task | exits the checklist |
| **Progress** | the note card shows a `done / total` badge |
| **Switch to text** | markers are removed, the words stay |
| **Switch to list** | each line becomes an unchecked task |

> New notes start as text. Tap the ☑ button in the footer to turn a note into a
> checklist.

---

## 9. Math / formulas

Formulas render beautifully with **KaTeX**, entirely offline. Toggle:
Settings → Appearance → **Math formulas** (on by default).

### Everything that works

| You type | You get |
|---|---|
| `$x^2$` | inline formula |
| `$$x^2$$` | display formula |
| `\(x^2\)` | inline formula (auto-converted) |
| `\[x^2\]` | display formula (auto-converted) |
| `E = mc^2` *(on its own line)* | display formula |
| `a^2 - b^2 = (a-b)(a+b)` | display formula |
| `x = \frac{-b \pm \sqrt{b^2-4ac}}{2a}` | display formula |

### What it will never touch

The detector is deliberately conservative. It leaves alone:

- **prose** — a line containing ordinary 3+ letter words is never mathified
- **code** — fenced blocks and inline code spans are skipped
- **structure** — headings, list items, quotes, tables, raw HTML
- **front-matter**
- anything already containing `$`

### Quality guarantees

- **line count is never changed** — which is what lets maths coexist with the
  exact scroll-anchoring system
- malformed LaTeX renders a **red error string** instead of breaking the note
- if a bare line sits inside an existing `$$ … $$` block it is not re-wrapped
- fonts are bundled, so it works with no network

---

## 10. Find in note

A search field in the editor header.

| | |
|---|---|
| **Open** | 🔍 in the editor header |
| **Count** | live `3/12` as you type |
| **Next / previous** | ↑ / ↓ buttons, or `Enter` / `Shift+Enter` |
| **In edit mode** | jumps the viewport to the match — without selecting it, so your typing isn't interrupted |
| **In preview mode** | every match is **highlighted** in yellow and the active one highlighted in orange with an outline |
| **Close** | ✕, or `Esc` |
| **On switch** | highlights are cleared automatically |

Matching is Unicode-normalised and case-insensitive.

---

## 11. Organisation

### Colours — 30 options

The first swatch is always **"Theme default"**, which makes the note follow
whatever theme you're using. Picking it **removes** any custom colour.

After it: 29 named colours — Cerulean, Brown, Gray, Teal, Green, Pink, Purple,
Coral, Sand, Orange, Sky, Cyan, Mint, Sage, Forest, Lime, Olive, Honey,
Apricot, Rust, Cherry, Rose, Magenta, Violet, Indigo, Steel, Sandstone, Clay,
Charcoal.

**Custom colours are permanent.** They belong to the note and are never
rewritten when you switch themes — only the light/dark variant swaps so the
text stays readable. Note text is **black on light themes and white on dark
themes, always**, for every colour.

### Folders (genres)

- 20 preset emoji + **any custom emoji** (including complex sequences) via the
  pencil tile
- 6 colour tints, each with a light and a dark variant
- live preview of the folder icon
- deleting a folder moves its notes out rather than deleting them

### Labels

- create, rename, delete (with a live count of affected notes)
- assign to any note from the editor or the card
- labels show as a section in the sidebar
- deleting a label removes it from notes but **keeps the notes**

### Pin

Pin a note to keep it at the top of its list. The pin also shows on the card.

### Archive

Archived notes leave the main list and gain an `Archive_` prefix on their
folder. A dedicated sidebar view lists them, and search can optionally include
them (Settings → Search & Notes → **Include Archive in search**).

### Trash

| Action | Result |
|---|---|
| **Delete note** | moved to trash, with a 6-second **Undo** toast; the note is also un-pinned |
| **Restore** | returns to the main list, and also **un-archives** if it was archived |
| **Delete forever** | removes the folder and all its attachments, permanently |
| **Empty trash** | removes everything in the trash |

---

## 12. The Vault

A hidden section that keeps sensitive notes out of sight.

> ⚠️ **Important and easy to misunderstand: the vault is a screen lock, not
> encryption.** Vaulted notes are ordinary plaintext `node.md` files on disk
> with a `vaulted: true` flag. The password protects the *view* in the app, not
> the bytes on your device. Anything that can read your notes folder can read a
> vaulted note.

### What it does

| | |
|---|---|
| **Purpose** | keep private notes out of the main list until you explicitly open the vault |
| **Password** | stored only as a **PBKDF2-SHA256 hash, 150,000 iterations**, with a random per-record salt |
| **Security questions** | 2 of 15 options; answers hashed the same way |
| **Change password** | requires the current password |
| **Forgot password** | **answering the two security questions is enough** — no current password required |
| **Session** | unlocking lasts until the app is closed |
| **Vaulted notes** | are excluded from Notes, Archive, Folder, Label and Trash views |

### Setup

Settings → Vault → **Set up protection**: choose a password, then two security
questions and their answers. The two questions must be different.

### Practical advice

If you need genuine at-rest protection, keep sensitive notes in an encrypted
container outside the app's root folder and import them when needed. The vault
is a privacy convenience, not a security boundary.

---

## 13. Views & navigation

Six sidebar destinations, plus detail views:

| View | Contents |
|---|---|
| **Notes** | active, non-archived notes |
| **Folders** | the folder manager with preview cards |
| **Archive** | archived notes |
| **Vault** | vaulted notes (password-gated) |
| **Trash** | deleted notes |

Below those: your **folders** and **labels** as drill-down views. The sidebar
is always visible, full height, on phone and desktop.

The top bar has: the app name, **search**, a **grid/list** toggle, the
**moon / sun / system** theme menu, and a **new note** button that always
wears the same colour as uncoloured note cards.

### Back navigation

The **system/browser back button** works the way you'd expect — it closes the
topmost thing first:

```
lightbox → image editor → image gallery → note editor → dialogs → drawer
→ previous view → (only then) leave the app
```

Closing a layer through its own button also consumes its history entry, so
back never "unwinds" something you already dismissed.

---

## 14. Grid & list, drag to reorder

| | |
|---|---|
| **Layouts** | grid (masonry) or list (single column) |
| **Gesture** | long-press to lift, then drag |
| **Feel** | the grid reflows in real time around the card you're moving, with a ghost slot showing exactly where it will land |
| **Extras** | haptic tick on lift, edge auto-scroll while dragging, clicks right after a drag are swallowed so you never open a note by accident |
| **Memory** | the arrangement is remembered and stored in the note's `order` field |
| **Sorting** | manually ordered notes keep their position; anything without a manual position falls back to most-recently-updated |

---

## 15. Search

The top-bar search covers, in every view:

- **titles**
- **note bodies** (raw *and* markdown-stripped, so `**bold**` matches `bold`)
- **checklist item text**
- **label names**
- **folder names**
- **image file names**

Matching is Unicode-normalised and case-insensitive. Archived notes can be
included via Settings → Search & Notes.

Separately, **Find in note** searches inside the note you're editing
([§10](#10-find-in-note)).

---

## 16. Themes

**37 hand-built themes** plus **"Follow system"** = 38 buttons.

| Group | Themes |
|---|---|
| **Classic** | Keep Light, Keep Dark |
| **Light & colourful** | Sky Blue, Fresh Mint, Lavender, Rose Quartz, Warm Peach, Sandy Beige, Cool Slate |
| **Dark UI** | Midnight Blue, Espresso, Forest Night, Deep Plum, Charcoal, Twitter Dim |
| **Comfy palettes** | GitHub Light/Dark, Catppuccin Latte/Mocha, Solarized Light/Dark, One Dark, T3 Chat, T3 Ghost, Nord, Dracula, Gruvbox, Rosé Pine, Tokyo Night, Everforest, Kanagawa |
| **High contrast** | High Contrast Light, High Contrast Dark, High Contrast Navy |
| **Special** | Frutiger Aero, Vaporwave, Aurora |

Each has a **live preview swatch**, and the active one is checked.

Themes reach every corner of the app — cards, dialogs, **scrollbars**,
**text selection**, the **note colour defaults**, and on Android the
**native status and navigation bars**. A theme switch is instant; nothing is
rewritten on disk.

The **moon / sun** button in the top bar offers Light / Dark / System. "System"
follows the OS, including live changes on Android.

---

## 17. Appearance & personalisation

| Setting | Range |
|---|---|
| **App name** | up to 24 characters — replaces the brand everywhere, including the browser tab title |
| **Custom wallpaper** | any image, auto-compressed to 1600 px JPEG |
| **Background blur** | 0–40 px |
| **Background dim** | 0–80 % |
| **Fixed wallpaper** | keep it still while scrolling |
| **Note spacing** | Compact / Cozy / Spacious |
| **App text size** | 70–150 % |
| **Note text size** | 70–170 % — affects the note body only |
| **App font** | 14 choices |
| **Note font** | 14 choices incl. monospace |
| **Image grid width** | 40–100 % |
| **Animations** | on/off — disables card entrances, hover lifts and micro-motion |

**Reset appearance settings** restores the defaults.

> Two text scales exist because they're genuinely different jobs: shrinking the
> app UI to fit more cards on screen shouldn't shrink the text you're reading
> in a note.

---

## 18. Import / Export

### Export — a real backup ZIP

Settings → General → **Backup & restore → Export**.

| Option | Effect |
|---|---|
| **Scope** | all notes / one folder / one label |
| **Include images** | embed the attachment files |
| **Include archive** | archived notes are part of the backup |
| **Include vault** | asks for the vault password first |

The ZIP contains `manifest.json`, every `node.md` in the same on-disk layout,
the attachments, `config.json` with folders and labels, and a `README.txt`.
App settings are **never** included — a backup restores your *notes*, not your
preferences.

### Import — one click

Settings → General → **Backup & restore → Import**, drop the ZIP.

Restores every note with its title, body, colour, pin, archive and vault
flags, labels, folder assignment, timestamps and attachments — plus folder
emoji/tints and the label list, merged without duplicates.

**Both the current format and legacy V5 exports are accepted.**

---

## 19. Platform behaviour

| | Web | Android |
|---|---|---|
| **Where notes live** | a real folder you pick, or the browser's private storage | real phone storage |
| **Root selection** | the browser's folder picker | an in-app picker that browses the real filesystem |
| **Permission** | the browser re-asks on each new session | All-files access (granted once) |
| **Back button** | browser back | system back, same behaviour |
| **Theme following the OS** | native | backed by the device's actual UI mode, live |
| **Status / nav bars** | `<meta theme-color>` | real system bar colours, live |
| **Fonts & maths** | bundled | bundled — fully offline |
| **Opening a `.txt`/`.md`** | — | "Open with Notenoid" from any file manager |

### Sharing a folder with another app

Pick your root folder somewhere both apps can reach (e.g. a cloud-synced
folder) and the notes stay in sync through that folder — because the app stores
nothing anywhere else. This is also the intended way to move notes to a new
device.

---

## 20. Settings reference

Every setting, where it lives, and its default.

### General

| Setting | Type | Default | Effect |
|---|---|---|---|
| App name | text | `Notenoid` | brand name + document title |
| Confirm before deleting | switch | **off** | ask before trashing |
| Backup & restore | buttons | — | Export / Import |
| Storage | info + Change / Rescan | — | where notes live |
| Reset appearance settings | button | — | restore defaults |

### Appearance

| Setting | Type | Default | Range |
|---|---|---|---|
| Theme | grid | `system` | 38 options |
| Custom wallpaper | file | none | — |
| Background blur | slider | 8 px | 0–40 |
| Background dim | slider | 35 % | 0–80 |
| Fixed wallpaper | switch | on | — |
| Note spacing | select | `cozy` | compact / cozy / spacious |
| App text size | slider | 100 % | 70–150 |
| **Note text size (markdown)** | slider | 100 % | 70–170 |
| App font | select | default | 14 |
| **Note font** | select | default | 14 |
| **Image grid width** | slider | **75 %** | 40–100 |
| **Math formulas** | switch | **on** | — |
| Animations | switch | on | — |

### Search & Notes

| Setting | Type | Default | Effect |
|---|---|---|---|
| Include Archive in search | switch | on | merge archived notes into results |
| Search covers | info | — | titles, bodies, checklists, labels, folders, image names |

### Vault

| Setting | Effect |
|---|---|
| Set up protection | first-time password + 2 security questions |
| Lock now / Unlock | end or resume the vault session |
| Change password | requires the current password |
| Security questions | replace them with the current password + one correct answer |
| Export vault | opens Export with the vault included |

---

*End of feature reference. Implementation details and file/line references for
everything here are in `Notenoid-v11-Code-Analysis.md`.*
