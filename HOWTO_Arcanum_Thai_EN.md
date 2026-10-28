# Game-specific guide: putting Thai into Arcanum: Of Steamworks and Magick Obscura

What was actually done for this game: unpacking, Thai fonts, encoding, translation, categorisation, polish, and
packaging. Written so a person can follow it and an AI can repeat it.

**Scope of this document:** methods and sample code only. Sample rows below are invented, only to show file formats.
Everything is generated from the reader's own copy of the game.

Target: Arcanum 1.0.7.4 + Unofficial Arcanum Patch (UAP), Windows, Python 3 (Pillow, pythainlp).

---

## 1. Game structure

### 1.1 Archives and override order
Data lives in DAT1 archives (zlib). Priority, lowest to highest (later wins):

```
Arcanum1.dat ... Arcanum5.dat, modules/Arcanum.dat, modules/Arcanum.PATCH0,
modules/Arcanum.PATCH1 (UAP text layer), modules/Arcanum.PATCH9
```

- Loose files in the game's `data/` folder beat all root archives.
- **Tested exception:** 17 module-level text files (stored in `modules/Arcanum.dat` / `PATCH1`) do NOT accept overrides
  from `data/`; they must be placed in `modules/Arcanum/mes/`. They are `game_rd_npc_{f2f,f2m,m2f,m2m,m2m_dumb}`,
  `game_rp_npc_{f2f,f2m,m2f,m2m}`, `gamearea`, `gamedesc`, `gameitemeffect`, `gamenote`, `gameplaque`, `gamequestlog`,
  `gamequestlogdumb`, `gamereplog` (all `.mes`).
- Translate the text **after UAP**. Reading only the base archives makes patched lines miss their keys.

### 1.2 DAT1 format
- Last 4 bytes = footer size. Seek back that far: entry count (uint32), then per entry:
  `name_len(uint32), name (NUL-terminated), unk, kind, size, packed, offset` (all uint32).
- `kind == 2` means zlib; otherwise stored raw. Names use `\`, compare case-insensitively.
- The reader needs `read_latest(name)` (search the order above from the end) and `list_latest(suffix)`.

### 1.3 Text-bearing files
| Kind | Location | Approx. count | Notes |
|---|---|---|---|
| Dialogue `.dlg` | `dlg/` | ~700 files | 31,381 unique lines, shared across files |
| Text `.mes` | `mes/` (root) and `modules/Arcanum/mes/` | ~80 files | menus, HUD, quest log, places, items |
| Rules `.mes` | `rules/` | a few | backgrounds etc. |
| Fonts `.ART` | `art/interface/*font.art` | 28 used | bitmap fonts |

Intentionally untranslated: in-game books and newspapers, multiplayer screens, developer-label lines in the spell list,
stat abbreviations (ST DX CN BE IN PE WP CH ...).

---

## 2. File formats (invented samples)

### 2.1 `.dlg`, one row per line
NPC row (4th field = iq, empty):
```
{12}{npc text}{npc text when the player is female}{}{...control fields...}
```
Player reply row:
```
{13}{player reply text}{gender}{iq}{condition}{next id}{effect}
```
- Translatable fields: NPC = field 1 and field 2 (if present); player = field 1.
- Fields starting with commands `E:` `G:` `T:` `R:` `B:` ... are not text: always skip.
- Negative iq (e.g. -4) = reply offered to a low-intelligence player.

### 2.2 `.mes`
```
{id}{text}
{id}{multi-line text
second line}  // comments allowed
```
Multi-line entries are translated line by line (each line is one unit).

### 2.3 Line categories in the work queue
| Tag | Meaning | Rule |
|---|---|---|
| N | NPC speaks to player | voice per character |
| F | NPC speaks to a female player (field 2) | identical to N except gender words |
| P | player reply | polite, gender-neutral |
| D | reply for a low-IQ player | deliberately broken, short, space between words |
| M | unit inside a `.mes` | per file kind |

---

## 3. Thai fonts

### 3.1 `.ART` font layout
```
0x84-byte header | 256x4-byte palette | 28-byte frame records x N | concatenated frame pixels
frame record = (width, height, data size, ox, oy, dx, dy) as seven int32
```
- Pixels are alpha levels 0..16. A frame is raw when `size == width*height`, otherwise RLE (byte with bit 7 set = n literal
  bytes follow, else repeat the next byte n times).
- Frame for byte code c is at index `c-32` (c < 0x80) or `c-31` (c >= 0x80).
- **`ox` is the pen advance.** Normal glyph: `ox = width`. Combining mark: `ox = 0`.
- Frame height is fixed per font (Flare14 = 16 rows); ink must stay inside it.
- After changing frames update the total data size at offset `0x44 + 4*i` (i = 0..7, all eight equal) and the frame count at `0x20`.
- Parse -> write back must reproduce the original bytes (test before editing).

### 3.2 Encoding and marks
- Text is stored as **CP874 single bytes** (Thai = 0xA1-0xFB, ASCII unchanged).
- All combining marks (ั ิ ี ึ ื ุ ู ็ ํ and ่ ้ ๊ ๋ ์) are `ox = 0` frames written **before** the consonant. A mark frame has left padding equal to the
  average consonant width (averaged over กขคงจชดตนบปมยรลวสหอ) so the ink lands on the following consonant.
- Reserved codes: 0x80-0x84 = tone marks/thanthakhat `่ ้ ๊ ๋ ์` **raised 2 px** (used when an upper vowel is present);
  0x85-0x89 = the same set **shifted left 2 px** (used over ป ฝ ฟ ฬ).
- Mark order on one consonant: below-vowel, upper-vowel, tone; then the consonant.
- `ำ` is written as `ํ` (nikhahit, zero advance) + `า`.

### 3.3 Choosing sizes (what was used with Tahoma during testing)
Automatic rule: try sizes from 40 down to 8 px, take the **largest** size where every Thai glyph (tone marks raised by 2 px)
fits the frame height once the baseline is placed.
- size >= 11 -> bold face, below -> regular;
- size <= 14 -> 1-bit hinted rendering, no anti-alias (thin distinctions ช/ซ, ึ/ิ must survive);
- size > 14 -> anti-aliased, intensity = alpha x 19 clipped to 16;
- very small fonts (Arial10, Garmond6/8/9) get a "squeezed" 7 px Thai with clipped tops.

Result (pixel size of the Tahoma render; if you use another Thai font, re-run the rule and inspect in game):

| Font file | px | Font file | px |
|---|---|---|---|
| arial10 | 7 | Euph30 | 25 |
| ArialB12 | 11 | Georgia30 | 27 |
| BookmanOldBold18 | 17 | LatinXCN30 | 28 |
| casablanca16 | 10 | morph15 | 11 |
| CasablancaAntique30 | 28 | Morph30 | 22 |
| ClarendonBLK18 | 17 | NewTimes16 | 17 |
| Cloister18 | 15 | Nick16 | 14 |
| Comic12 | 11 | Pepper20 | 15 |
| Courier10 | 8 | pork12 | 14 |
| Elga12 | 10 | roller | 16 |
| Flare12 | 12 (regular + faux bold) | Swiss921 | 29 |
| **Flare14** | **12 bold, 1-bit** | Zurich16 / Zurich20 | 14 / 19 |
| Garmond6 / 8 / 9 | 7 (squeezed) | | |

- **Flare14** is the font of both the dialogue window and the floating NPC speech (confirmed in the real game). It is hand-set and
  the most important to get right.
- Flare12 is not used by the dialogue window (untested). Symbol fonts (icons, bookimages, newsicons, mpicons) are skipped.
- Which screen uses which font is not known in advance: a screen showing garbage Latin letters instead of Thai needs a hand-tuned style.
- Already rejected: precomposed glyphs (do not scale), vertical squashing (blurs), 13 px or larger in a 16-row cell, boosted
  anti-aliasing (strokes merge).

### 3.4 Line breaking
The engine wraps only at spaces. At build time pythainlp (`newmm`) inserts an ordinary space at a word boundary once a Thai run
reaches **12** characters, never splitting short words from the one before them (ว่า ที่ ไว้ นี้ นั้น ได้ แล้ว ก็ ๆ จริง เลย อยู่ ไป มา ไหม นะ ครับ ค่ะ).
Spaces are added at build time only, never stored in the translation, except D lines which are deliberately spaced word by word.

---

## 4. Pipeline

```
extract -> translate (Thai JSON) -> normalize -> build -> check -> package
```
| Tool | Job |
|---|---|
| `dat.py` | read archives in override order |
| `extract.py` | list untranslated English lines of one file |
| `dq.py` | dialogue work queue (frozen list): show/apply/check/status |
| `merge_todo.py` | merge a batch of translations of a `.mes` file into its JSON |
| `normalize.py` | regex spelling normalisation across all JSON from the glossary (run after every edit) |
| `build.py` | read originals, replace text fields, space out, encode, write files + build fonts |
| `check.py` | structural comparison of built files vs originals; must say "0 problems" (781 files) |
| `audit.py` | automatic checks: placeholders, brackets, leftover English, length ratio, glossary names, N/F pairs, undamaged D lines, shifted pairs |
| `pq.py` | polish queue show/apply/done/status/find with `changes.log` |
| `make_release.py` | package the installer |

Storage rules:
- Key = exact English text, value = Thai (normal Unicode). Repeated lines are translated once; untranslated lines stay English.
- Dialogue uses one shared pool (`pool_work.dlg.json`); `.mes` files have one JSON each.
- Placeholders must match in kind and count: `@PCNAME@`, `@pcname@`, `%d`, `%s`, `{..}`, and the count of `[..]` (stage directions are in square brackets).

---

## 5. Order of work (milestones)

| Milestone | Scope | How to test |
|---|---|---|
| M0 | first conversation (one NPC) | talk through all branches, check marks and wrapping |
| M1 | the whole opening sequence | play the opening, all branches |
| M2 | menus, HUD, log, combat, all 28 fonts | open every menu; look for overflow/garbage |
| M3 | first town, trains, quest log, place names | walk, talk, buy, ride, open the log |
| M4 | character creation, stats, skills, spells, backgrounds | create a character through every screen |
| M5 | generic NPC reactions, player replies, rumours (.mes) | walk a town, listen to floating speech |
| M6 | item/object names and descriptions, notes, schematics | open inventory, examine objects |
| M7-M8 | all remaining story dialogue (queue via `dq.py`, ~280 lines per batch) | follow the main story |

After each milestone: build, check, the user installs and plays a short list, sends screenshots of **the game window only**, fix.

---

## 6. Translation principles for this game

### 6.1 Context
Steampunk fantasy: magick and technology cancel each other; many races (human, elf, dwarf, gnome, halfling, orc, ogre, dark elf...).
Speech differs by place and people: archaic and courtly (Cumbria), clipped and technical (Tarant), rough (gang quarters), gruff (dwarves),
aloof (elves). The player may be male or female, hence paired N/F lines.

### 6.2 Register (decided)
- **ข้า / ท่าน / เจ้า** is the baseline for the whole game (the user chose to keep it). Do not change pronouns game-wide.
- Very polite NPCs (e.g. a monk follower) use "ผม" and address the player as "ท่าน"; "sir/madam" in N/F pairs become ท่าน / ท่านหญิง.
- D lines (low-IQ player): เอ็ง/ข้า, broken, short, a space between every word; other names may be mangled; never turn into normal Thai.
- Stage directions stay in `[ ]` with the same bracket count.
- Polish rule: fix only what is unreadable or ungrammatical; do not rewrite the whole game for style.

### 6.3 Typical machine-translation faults and fixes
- English word order (passive "ถูก...โดย...", stacked "ซึ่ง", stacked "ของ") -> break into short sentences
- idioms translated literally -> translate the intent
- pronouns that do not match gender or role in F lines
- one name spelled several ways -> glossary + `normalize.py`
- a double negative flipping yes/no
- words with game-specific meanings (e.g. "Magick" = เวทมนตร์ versus "mage" = จอมเวท)

### 6.4 Glossary (principles)
English -> Thai table with notes. Categories: character names, places, organisations/religion, races, stats, skills, skill ranks,
technology disciplines, UI terms, forms of address (e.g. choose "ท่านผู้ดี", not "ท่านผู้เจริญ" which addresses monks).
Proper names are transliterated one way throughout; stat abbreviations stay English. `normalize.py` takes new regex rules whenever a decision is made.

---

## 7. Polish pass (v1.1)

1. Learn the story and characters from public walkthroughs and write your own "story bible" (timeline; each character's gender,
   race, manner; organisations; terms). Use it as context only; names that appear in the game win over guides.
2. Run `audit.py` first: placeholders, brackets, leftover English, Thai/English length ratio, glossary mismatches, identical N/F pairs,
   unbroken D lines, "shifted pairs" (a line's Thai fits its neighbour's English).
3. Mechanical fixes with `normalize.py`.
4. Read by eye in batches of ~100-170 units with `pq.py show A B`; write only the lines to change into `edits.txt` as `unit<TAB>new Thai`;
   `pq.py apply` checks placeholders and logs before/after in `changes.log`; then `pq.py done A B`.
5. Priority: tier 1 main story and key characters, tier 2 companions and common towns, tier 3 the rest, then the `.mes` files.
6. After each tier: normalize, build, check (0 problems), package a new version, user tests.

---

## 8. Installing and packaging

- The first release shipped as an installer (`Install.bat` + `install.ps1`: finds the game, backs up replaced files, writes a manifest)
  and as a copy-paste tree (`data/` + `modules/Arcanum/mes/` + `filelist.txt`).
- The newer approach is a patcher that reads the user's own `.dat` files, verifies CRC32, and builds the files from a
  position-keyed translation table (file name + row id + field index) and Thai glyph frames rendered from an open Thai font
  (e.g. Sarabun or Noto Sans Thai). General details are in the generic guide, section 9.
- Patcher output must equal the reference build byte for byte (test with its `--out` mode).
- README: requires Arcanum 1.0.7.4 + Unofficial Patch; what is not translated; uninstall (delete the files listed in `filelist.txt`, never other files in `data/`).
- Keep the Unofficial Patch installer out of the package; tell users where to get it.

---

## 9. Pitfalls actually met

- Translating from the base archives instead of the UAP text -> lines miss their keys.
- Putting module-level files in `data/` -> no effect.
- A font without Thai -> Latin garbage on that screen.
- File names are unreliable for knowing which NPC a dialogue belongs to -> confirm in game.
- A built file identical to its original need not be shipped.
- Run tools with `PYTHONUTF8=1`; Git-Bash heredocs mangle backslashes, so write code or data containing `\` with a file-writing tool.
- The game runs elevated, so it cannot be driven or screenshotted by tools: rely on user screenshots of the game window only, never the whole desktop.
