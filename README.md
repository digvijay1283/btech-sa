# SCOA Unit 1 Tracker: how it works

A study tracker for the SCOA (Soft Computing & Optimization Algorithms) Unit 1 question bank. It does the following:

- lists all 50 questions, grouped by topic
- lets you mark each question **Completed** (✓) or **Needs revision** (↻)
- shows answers that open when you tap a question (tables, lists, diagrams, example boxes)
- imports new answers as **JSON**, pasted or loaded from `.json` files
- backs up and restores your marks and answers

Everything is in **one file**, `scoa-unit1.html` (about 470 lines). There's no build step, framework or server. Open it in a browser and it works.

---

## 1. File layout

The file has three sections, in this order:

| Section | Approx. lines | What it contains |
|---|---|---|
| `<head>` | 1–100 | Fonts, two CDN scripts (markdown renderer), all CSS |
| `<body>` HTML | 100–175 | Header, stats, anchor example, toolbar, 2 modals, filter bar, list container |
| `<script>` | 175–465 | Question data, built-in answers, storage, rendering, import/backup logic |

Line numbers shift as you edit. Search for the names below instead (Ctrl+F in VS Code).

---

## 2. External dependencies

Two scripts load in `<head>` from cdnjs:

```html
<script src="https://cdnjs.cloudflare.com/ajax/libs/marked/12.0.2/marked.min.js"></script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/dompurify/3.1.6/purify.min.js"></script>
```

- **marked** converts the Markdown in imported answers into HTML.
- **DOMPurify** cleans that HTML so pasted content can't inject scripts.

Fonts come from Google Fonts (Inter and JetBrains Mono).

**Offline:** if the CDN can't load, `md()` falls back to showing the answer as plain preformatted text. The page still works, but without formatting.

---

## 3. Styling (CSS)

All colours are CSS variables in `:root`. Change them there to re-theme the whole page:

```css
:root{
  --bg:#f7f7f5;        /* page background */
  --card:#ffffff;      /* question cards, modals */
  --ink:#1c1c1e;       /* main text */
  --muted:#6b6b70;     /* secondary text */
  --line:#e7e7e3;      /* borders */
  --accent:#2f6b4f;    /* green: completed, primary buttons */
  --accent-soft:#e6f0ea; /* light green: example boxes */
  --rev:#b7791f;       /* amber: needs revision */
  --rev-soft:#fbf1df;
  --radius:14px;
}
```

Key classes:

| Class | Used for |
|---|---|
| `.q`, `.q.done`, `.q.rev`, `.q.open` | Question card and its states |
| `.qhead`, `.num`, `.qtext`, `.acts` | Card header row: number, text, buttons |
| `.btn.d` / `.btn.r` (`.on`) | ✓ and ↻ buttons |
| `.tag.has` / `.tag.mine` / `.tag.soon` | "answer" / "your answer" / "no answer yet" labels |
| `.ans` | The expandable answer area. Its child styles (`table`, `pre`, `blockquote`, `code`) control answer formatting. |
| `.ans blockquote` | **Green example box.** Any Markdown line starting with `>` renders as one. |
| `.overlay`, `.modal` | The Add answers and Backup popups |
| `.chip`, `.chip.on` | Filter buttons |
| `@media (max-width:560px)` | Mobile adjustments |

---

## 4. Data (top of `<script>`)

### `topics`: the sections and which questions belong to each
```js
const topics = [
 {name:"Introduction to Soft Computing", qs:[1,2,3,4,5,6,7]},
 {name:"Soft Computing vs Hard Computing", qs:[8,9,10,11,12]},
 ...
];
```

### `Q`: question text, keyed by question number
```js
const Q = {
  1:"Define soft computing, list a few soft computing techniques and its primary goal.",
  ...
  50:"Discuss in detail the various types of activation function..."
};
```

### `A`: built-in answers (HTML strings)
Only Q1–Q7 are built in. They're written in **HTML**, not Markdown, inside JS template literals (backticks):
```js
const A = {
  1:`<h4>Definition</h4><p>Soft computing is...</p>`,
  ...
};
```
All other answers come from imports, which are stored in `mine` (see the next section).

---

## 5. Storage (localStorage)

```js
const KEY  = "scoa-u1-progress";  // marks:   { "8": {d:true, r:false}, ... }
const AKEY = "scoa-u1-answers";   // answers: { "8": "markdown text", ... }
```

- `state[n].d` is true when question *n* is **completed**.
- `state[n].r` is true when question *n* **needs revision**.
- `mine[n]` holds the **imported Markdown answer** for question *n*.
- `save()` writes both objects. Every read and write is wrapped in `try/catch`, so the page still works if storage is blocked.

**Important:** localStorage belongs to **one browser on one device, at one address**.

- The claude.ai link and a local copy opened as `file:///.../scoa-unit1.html` have **separate storage**.
- To move your data between them, use **Backup / restore**.

---

## 6. How answers are chosen and rendered

```js
function getAns(n){ return mine[n] ? md(mine[n]) : (A[n] || ""); }
```

The page shows an imported answer (`mine`) first, then a built-in answer (`A`), and otherwise nothing.

`md(text)` runs `marked.parse()` and then `DOMPurify.sanitize()`.

`render()` rebuilds the whole list on every change:
1. It loops over `topics` and filters each question by the active filter (`all` / `pending` / `done` / `rev`) and the search text.
2. It writes the topic heading with a `done/total` count.
3. For each question, it builds the card with its tag, the ✓/↻ buttons and the answer.
4. It attaches the click handlers: tap the text to open or close, ✓ and ↻ to toggle, and "Remove this added answer" (which needs two clicks).
5. It calls `stats()` to update the counters and the progress bar.

`toggle(n, k)` flips `state[n][k]`, saves, re-renders, and **re-opens any cards that were open**, so ticking doesn't collapse what you're reading.

---

## 7. JSON import format

`parseAnswers(raw)` accepts any of these shapes:

```json
{ "answers": [ { "q": 8, "answer": "markdown..." }, { "q": 9, "answer": "..." } ] }
```
```json
[ { "q": 8, "answer": "..." } ]
```
```json
{ "q": 8, "answer": "..." }
```

Rules:
- `q` (or `qno` / `question_no`) must be a question number that exists in `Q` (1–50).
- `answer` must be a non-empty string of **Markdown**.
- Extra fields such as `unit` and `topic` are ignored.
- Wrapping ```` ```json ```` fences are stripped automatically.
- If one item is invalid, nothing is imported and an error is shown.

There are two ways to import:
- **Paste:** `doImport.onclick` reads the textarea.
- **File(s):** `importFile.onchange` reads one or more `.json` files.

### Markdown features that render well
- Headings: `### Section`
- **Bold**, *italic*, `inline code`
- Numbered and bulleted lists, including nested ones
- Tables: `| a | b |` + `|---|---|`
- ASCII diagrams inside ```` ``` ```` code blocks
- `> 🚗 Example...` for the **green example box**

---

## 8. Backup format

**Copy backup** puts this on your clipboard:
```json
{ "scoa_unit1_backup": 1, "progress": { ...state... }, "answers": { ...mine... } }
```
- If the browser blocks clipboard access, the backup is shown in the textarea for you to copy manually.
- **Restore** checks for `scoa_unit1_backup`, then **replaces** all your marks and answers.

**Reset progress** (at the bottom of the legend) clears only the **marks**. Imported answers stay.

---

## 9. Common changes

### Add or edit a question
Edit its text in `Q`. If it's a new number, also add it to the right `qs` array in `topics`.

### Make a tracker for another unit or subject
1. Copy the file, e.g. `scoa-unit2.html`.
2. Change `<title>`, the `<h1>`, and the subtitle `<p>`.
3. Replace `topics` and `Q` with the new question bank.
4. Empty the built-in answers: `const A = {};`
5. **Change both storage keys** so the units don't overwrite each other:
   ```js
   const KEY = "scoa-u2-progress", AKEY = "scoa-u2-answers";
   ```
6. Change `scoa_unit1_backup` in `copyBackup.onclick` and `doRestore.onclick`, e.g. to `scoa_unit2_backup`.
7. Update the anchor-example `<details>` block.

### Change the example-box colour
Edit `.ans blockquote` (it uses `--accent-soft`).

### Add a new filter (e.g. "has no answer")
1. Add a button: `<button class="chip" data-f="noans">No answer</button>`
2. In `render()`'s filter function, add: `if(filter==="noans" && getAns(n)) return false;`

### Move all imported answers into the file permanently
Restore isn't needed if the answers are built in. Convert each Markdown answer to HTML and put it in `A`, or keep them as Markdown and change `getAns` to `md(A[n])` for those entries.

---

## 10. Asking Claude for more answers

Use this prompt so the output matches what the importer expects:

> Give me answers for SCOA Unit 1 Q__–Q__ in tracker JSON format:
> `{"answers":[{"q":N,"answer":"<markdown>"}]}`. Use headings, lists, tables, ASCII diagrams in code blocks, and end each answer with a `> 🚗` example line using the self-driving car example.

For big batches, ask for a `.json` **file** and load it with **📄 Choose .json file(s)**.

---

## 11. Publishing changes

- **Local use:** open `scoa-unit1.html` directly in a browser. Remember that local storage is separate from the online version (section 5).
- **Updating the claude.ai link:** give the edited file to Claude and ask it to republish to the existing **SCOA Unit 1 Tracker** artifact.
- **Before replacing the version you use:** click **Copy backup** first, just in case.

---

## 12. Quick function index

| Name | Purpose |
|---|---|
| `topics`, `Q`, `A` | Data: sections, question text, built-in answers |
| `state`, `mine`, `save()` | Marks, imported answers, persistence |
| `md(text)` | Markdown → safe HTML |
| `getAns(n)` | Answer to display for question *n* |
| `parseAnswers(raw)` | Validates and parses import JSON |
| `render()` | Draws the whole list |
| `toggle(n, k)` | Flips completed (`d`) or revision (`r`) |
| `stats()` | Counters and progress bar |
| `show(id)` / `hide(id)` / `msg(id, text, ok)` | Modal and message helpers |
| `doImport`, `importFile` | Paste and file import handlers |
| `copyBackup`, `doRestore` | Backup and restore handlers |
