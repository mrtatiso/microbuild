# microbuild

microbuild is an editor for small 16×16 game assets. Everything you make in it is saved as one plain-text **`.micro`** file. That file holds every asset of a game: the pixel art, the animations, the scenes that place them, and the notes about them.

The `.micro` file is the real product. The editor is just a comfortable way to write one. A game, or the upcoming **microbuild engine**, reads the `.micro` file and asks for what it needs by name:

```
antorcha · apagada   → a still image
antorcha · loop      → an animation
antorcha · pasillo   → a scene built from those pieces
```

This document describes the format, so anyone can read or write `.micro` files: by hand, with a script, or from their own engine.

---

## Getting a `.micro`

Open `microbuild.html` in a browser. Nothing to install, nothing fetched from anywhere.

- Your work is saved in the browser as you go.
- **Ctrl+S** downloads `microbuild.micro`.
- **Ctrl+O**, or dropping a file on the window, opens one.

---

## The idea in one picture

```
[cluster]                 a group, e.g. one object or character: "antorcha"
 ├─ (string)              one 16×16 image: "apagada"
 ├─ <loop>                several 16×16 frames: an animation or a sheet: "loop"
 ├─ {scene}               a grid (up to 18×8) of the cluster's strings and loops: "pasillo"
 └─ /comment/             a note attached to any of the above
/comment/                 a free note, attached to nothing
```

Each kind of thing has its own pair of signs. In the file, every line starts with its sign, so you can always tell what a line is by its first character.

| sign | kind | what it is |
|---|---|---|
| `[ ]` | cluster | a named group. Everything else lives inside one. |
| `( )` | string | exactly one 16×16 image: a tile, a static sprite, an object, anything. |
| `< >` | loop | several 16×16 frames in order. With a speed it is an **animation**. Without one, it is a **sheet**. |
| `{ }` | scene | a grid of up to 18×8 cells, each holding a string or loop of the same cluster. |
| `/ /` | comment | a note. Engines ignore comments. |

A string never animates, and a loop is never a single image. A torch that can be off or lit is one cluster, `antorcha`, holding a string `apagada` and a loop `loop`. The engine asks for the cluster, then for the string or the loop.

---

## A complete file

```
@microbuild

[da:f9:9d:64:3e:08 antorcha #d29922 @0,0] /luz del pasillo/
(43:ea:a0:51:54:1a apagada: --~,1h;=o;--.:899)
<73:a3:6f:15:45:87 loop 6fps: --,;=;0b1;=m;--.:d92 | --,;=;0b10e2;=m;--.:d92d62> /encendida/

{b1:76:34:51:10:50 pasillo 6x3 @260,0}
43:ea:a0:51:54:1a*6
43:ea:a0:51:54:1a*2,73:a3:6f:15:45:87,43:ea:a0:51:54:1a*3
43:ea:a0:51:54:1a*6

/agregar agua en el nivel 2/ @-350,275
```

---

## General rules

- **Line 1** is `@microbuild`. Any other line that starts with `@` is reserved and ignored.
- Lines are trimmed. **Empty lines** only end a scene's rows; otherwise they mean nothing.
- **Order defines ownership.** After a `[cluster]` line, every `( )`, `< >` and `{ }` belongs to that cluster, until the next `[cluster]`.
- **`!` before a line** marks the entry as hidden, for example `!(… )` or `![… ]`. The editor dims it. An engine may treat it as a draft and skip it.
- **`@x,y`** is where the item sits in the editor's canvas. It is layout only, and engines can ignore it.
- Text encoding is UTF-8, and line endings may be `\n` or `\r\n`.

### IDs

Every cluster, string, loop and scene has a random **id shaped like a MAC address**: six lowercase hex pairs, for example `43:ea:a0:51:54:1a`. Comments have no id.

- An id is created once and **never changes**: not on rename, not when the item moves to another cluster.
- Ids are unique across the whole file.
- Scenes refer to their pieces by id. Engines can look things up by id (stable) or by name (readable).

### Names

- A **cluster name** is unique in the file.
- A **string, loop or scene name** is unique inside its cluster.
- Names may contain spaces. They never contain the signs that delimit their own line: `[ ]` for clusters, `{ }` for scenes, and `( ) < > :` for strings and loops.

---

## Line by line

### `[cluster]`

```
[id name #rrggbb @x,y] /comment/
```

| part | meaning |
|---|---|
| `id` | MAC-like id |
| `name` | what the engine calls it |
| `#rrggbb` | the cluster's color in the editor. Layout only. |
| `@x,y` | position in the editor |
| `/comment/` | optional note |

### `(string)`

```
(id name: SEED) /comment/
```

A single 16×16 image. `SEED` is the encoded image (see [Seeds](#seeds)). `?` means it has not been drawn yet.

### `<loop>`

```
<id name 6fps: SEED | SEED | SEED> /comment/
<id name: SEED | SEED>
```

Frames are listed in order and separated by `|` with spaces around it. Seeds never contain spaces, so splitting on whitespace and dropping the `|` tokens is enough.

- **`Nfps`** present (1–60): it is an **animation** that plays at N frames per second.
- **`Nfps`** absent: it is a **sheet**, a set of frames that is not played automatically. Pick one by index.
- A frame can be `?` if it is empty.

All frames of a loop were drawn with one shared palette in the editor, but each seed still carries its own colors. Every frame can be decoded on its own.

### `{scene}`

```
{id name WxH @x,y} /comment/
ROW
ROW
…
```

A grid **W** wide and **H** tall (W ≤ 18, H ≤ 8). The rows follow the header line directly, one line per row, until an empty line or the next line that starts with a sign.

Each row is a comma-separated list of cells:

| token | meaning |
|---|---|
| `id` | this cell shows the string or loop with that id |
| `.` | empty cell |
| `id*n` / `.*n` | the same cell repeated n times across |
| ` =n` at the end of the line | this row appears n times in total, going down |

```
43:ea:a0:51:54:1a*6                                        → 6 × apagada
43:ea:a0:51:54:1a*2,73:a3:6f:15:45:87,43:ea:a0:51:54:1a*3  → 2 × apagada, loop, 3 × apagada
.*6 =2                                                     → 2 empty rows
```

A cell that holds a loop plays that loop's animation.

### `/comment/`

```
(… ) /a note about this string/     ← attached: at the end of its owner's line
/a free note/ @x,y                  ← free: on its own line, with a position
```

Inside a comment, `\n` is a line break and `\\` is a backslash. The comment runs from the first `/` to the last `/` on the line, so it may contain `/` itself.

---

## Seeds

A seed is one 16×16 image in a single word, with no spaces:

```
--[~],BODY--.:PALETTE
```

```
--~,1h;=o;--.:899
│ │ └──┬──┘    └┬┘
│ │  body      palette: one color, #889999
│ └ mirrored: only the left 8 columns are stored
└ start
```

### Palette

- Each color is **3 hex digits**, `rgb`, and stands for `#rrggbb` with each digit doubled: `d92` → `#dd9922`.
- The palette lists **up to 9 colors**. In the body, color **1** is the first one, **2** the second, and so on.
- Color **0** is always **transparent**.

### Mirror flag `~`

If the seed starts with `--~,`, the image is symmetric left to right. Only the **left 8 columns** are stored, and column `15 − x` copies column `x`.

The stored width **W** is therefore 8 with `~` and 16 without it.

### Runs

Pixels are run-length encoded. A **digit** is a color index (0–9). It may be followed by a **letter** that says how many times it repeats:

```
a=1  b=2  c=3  d=4  e=5  f=6  g=7  h=8  i=9  j=10  k=11  l=12  m=13  n=14  o=15  p=16
```

`1h` means color 1, 8 times. `0b1` means transparent, transparent, color 1. A digit with no letter is a single pixel.

### Body

The body is in one of two modes. The writer picks whichever is shorter.

**Row mode** is the default. Rows are separated by `;` and the body ends with `;`.
- A row is a sequence of runs. Pixels missing at the end of a row are transparent.
- An empty row (nothing between two `;`) is fully transparent.
- `=` repeats the previous row once. `=` plus a letter repeats it that many times: `=o` is 15 more copies.
- Rows missing at the end of the image are transparent.

**Linear mode** is used when the body starts with `|`. It is one long run sequence that fills the image left to right, top to bottom, W pixels per row. Whatever is not covered is transparent.

### Worked example

```
--~,1h;=o;--.:899
```

1. `~`, so W = 8.
2. The palette is `899`, so color 1 = `#889999`.
3. Row 0 is `1h`: 8 pixels of color 1.
4. `=o` repeats it 15 times, giving rows 1–15.
5. Mirroring fills columns 8–15.

The result is a solid 16×16 grey square.

### Reference decoder (JavaScript)

```js
const RUN = 'abcdefghijklmnop';

function decodeRuns(s) {
  const out = [];
  for (let i = 0; i < s.length; i++) {
    const v = s.charCodeAt(i) - 48;
    if (v < 0 || v > 9) continue;
    const n = RUN.indexOf(s[i + 1]);
    if (n >= 0) { for (let k = 0; k <= n; k++) out.push(v); i++; }
    else out.push(v);
  }
  return out;
}

// returns 256 entries (row-major), each '#rrggbb' or null for transparent
function decodeSeed(seed) {
  if (seed === '?') return Array(256).fill(null);
  const m = seed.match(/^--[a-z]?(~)?,(.*?)--\.(?::([0-9a-fA-F]*))?$/);
  if (!m) throw new Error('bad seed');
  const [, mirror, body, pal = ''] = m;
  const W = mirror ? 8 : 16;
  const colors = [null];
  for (let i = 0; i + 3 <= pal.length; i += 3) {
    const [r, g, b] = pal.slice(i, i + 3);
    colors.push('#' + r + r + g + g + b + b);
  }
  const grid = Array.from({ length: 16 }, () => Array(16).fill(0));
  if (body.startsWith('|')) {
    decodeRuns(body.slice(1)).forEach((v, i) => {
      if (i < W * 16) grid[Math.floor(i / W)][i % W] = v;
    });
  } else {
    const rows = [];
    for (const tok of body.split(';').slice(0, -1)) {
      if (tok[0] === '=') {
        const n = tok[1] ? RUN.indexOf(tok[1]) + 1 : 1;
        for (let k = 0; k < n; k++) rows.push((rows[rows.length - 1] || []).slice());
      } else rows.push(decodeRuns(tok));
    }
    rows.slice(0, 16).forEach((r, y) => r.slice(0, W).forEach((v, x) => (grid[y][x] = v)));
  }
  if (mirror) for (const row of grid) for (let x = 0; x < 8; x++) row[15 - x] = row[x];
  return grid.flat().map((v) => colors[v] || null);
}
```

Older files may have a letter right after `--` (for example `--t,`). Ignore it; the decoder above already does.

---

## What an engine does with it

1. **Parse** the file line by line using the first character: `[` `(` `<` `{` `/`, or a scene row.
2. **Index** clusters by name. Inside each cluster, index strings, loops and scenes by name, and everything by id.
3. **Decode** seeds into pixels once, at load time.
4. **Play** a loop that has `fps` with `frame = floor(time × fps) mod frameCount`. A sheet has no fps: the game picks the frame.
5. **Draw** a scene as a W×H grid of 16×16 cells, resolving each id to its string or loop.
6. **Ignore** comments, `#color` and `@x,y`. Skipping `!` entries is up to the engine.

The microbuild engine is on its way. Until then, the rules above are everything needed to read a `.micro` file.

---

## License

microbuild is **source available**: free to use and to modify, but not to sell. It is released under the [PolyForm Noncommercial License 1.0.0](LICENSE). Keep the `LICENSE` file and its `Required Notice` line with any copy.

**What you make with microbuild is yours.** `.micro` files and the art inside them belong to whoever made them, and this license does not cover them.
