# How to make a game show Thai text (a guide for people and for AI assistants)

Scope: getting Thai to **display** correctly in a game that has no Thai support: unpacking, finding text, adding
Thai glyphs to the game's fonts, encoding, line breaking, building, installing, testing, packaging.
Translation quality is a separate topic (see the last section for a few rules for AI translators).

This file contains methods and sample code only. Everything the reader needs to produce is generated from the
**reader's own copy of the game**.

> For AI assistants: follow the sections in order, change one thing at a time, and always prove each stage
> on a single string/screen before scaling. Never ask the user to upload game files; work on their local copy.

---

## 0. Rules before starting

1. Work only on a copy of the game you own. Never modify the original archives, the .exe, or save files.
   Ship overrides (loose files) plus an uninstaller.
2. Prefer a package that builds the edited files on the user's machine from their own copy (see section 9),
   instead of carrying edited game files around.
3. Use an open Thai font for the glyphs: Sarabun, Noto Sans Thai, Prompt, Kanit, IBM Plex Sans Thai, etc.
4. Treat the first success as a milestone: one dialogue line in Thai in the real game. Only then do the rest.
5. If you cannot run the game yourself (AI), ask the user for a screenshot of the **game window only**.

Tools: Python 3 with Pillow (glyph rendering) and pythainlp (word breaking). Optional: a C# compiler
(`csc.exe` ships with Windows .NET Framework) for the final patcher.

---

## 1. Recon: what does the engine read, and from where?

List the game folder. Typical layouts:
- loose folders (`data/`, `text/`, `Localization/`) - easiest, edit and drop in;
- archives (`.dat .pak .bsa .big .wad .vpk .arc ...`) - need an unpacker;
- an engine with its own asset bundles (Unity `.assets/.bundle`, Unreal `.pak/.utoc`) - use an existing
  tool for that engine instead of writing one.

Find the **override order**. Most engines let loose files beat archives, and later archives beat earlier ones.
Some engines have a second layer (module/mod folders) that does *not* take overrides from the first layer.
Test it, do not assume: change one visible string, put the file in the suspected folder, start the game.
Write down which files work in which folder.

Check whether the game has community patches that change the text. If so, translate the **patched** text.
Anything you key by the original English will silently miss lines that the patch changed.

---

## 2. Unpacking an archive

Unknown format? Steps that usually work:
1. Look at the last and first 64 bytes. Many archives have a footer pointing to an index.
2. Look for a table of file names (ASCII/latin-1), followed by fixed-size records (offset, size, packed size).
3. Try zlib on a data block (`zlib.decompress`). If it fails, try raw deflate (`wbits=-15`), LZ4, LZO.
4. Search for a known tool or documented format (game-modding wikis, QuickBMS scripts). Prefer a known tool.

Skeleton for a footer-indexed, zlib-compressed archive (adapt field order to your format):

```python
import struct, zlib

def read_index(path):
    f = open(path, 'rb')
    f.seek(-4, 2); footer = struct.unpack('<I', f.read(4))[0]
    f.seek(-footer, 2); n = struct.unpack('<I', f.read(4))[0]
    idx = {}
    for _ in range(n):
        ln, = struct.unpack('<I', f.read(4))
        name = f.read(ln).rstrip(b'\0').decode('latin1')
        _unk, kind, size, packed, offset = struct.unpack('<5I', f.read(20))   # example layout
        idx[name.lower()] = (name, kind, size, packed, offset)
    return f, idx

def read_file(path, name):
    f, idx = read_index(path)
    with f:
        _, kind, size, packed, offset = idx[name.lower()]
        f.seek(offset); raw = f.read(packed)
    return zlib.decompress(raw) if packed != size else raw
```

Implement `read_latest(name)` that searches archives in the engine's override order, and `list_latest(suffix)`.
Cache the index per archive.

---

## 3. Find the text

1. Take a sentence you can see on screen, search all extracted files for it (as latin-1 and UTF-16).
2. Identify the container format. Examples of simple ones: `key=value`, `{id}{text}` rows, CSV, JSON, XML.
3. Separate **text fields** from **non-text fields** (ids, conditions, script commands, flags). Only text
   fields are touched.
4. Write a parser that can *rebuild the file byte-for-byte* when nothing is changed. Test that first.
5. Replace only text fields; keep the number of rows/entries, ids, line endings, comments.
6. Keep placeholders exactly (`%d`, `%s`, `{0}`, `@NAME@`, `<tag>`), including their count and order.
7. Make a structural checker that compares every built file with its original: same rows, same ids, same
   placeholders; only the text may differ. Run it after every build and require "0 problems".

Store translations keyed by the **original string** (a flat JSON `{"original": "ไทย"}`), or by position/id.
Original-keyed storage means repeated lines are translated once and partially translated files still work.

Some text will never be reachable (hard-coded in the .exe, baked into images, videos, online-only UI). List
these for the user instead of silently skipping them.

---

## 4. Fonts (the hard part)

Find out how the game draws text:

| Engine uses | What to do |
|---|---|
| TrueType/OpenType font file | Swap in a Thai-capable open font; check the glyph range and line height. |
| Bitmap font (sprite sheet per glyph, `.fnt`, `.art`, `.png` atlas) | Add Thai glyph bitmaps (this section). |
| Signed-distance-field / TextMeshPro atlas | Use the engine's own tool to build a Thai atlas from an open font. |

### 4.1 Bitmap fonts, step by step

1. **Parse and repack the font file**; repack must reproduce the original bytes before you change anything.
2. Pick a **single-byte encoding for Thai: Windows-874 (CP874)**. Thai occupies 0xA1-0xFB. Keep ASCII as is.
   (If the engine is UTF-8/UTF-16 aware, use Unicode and skip the byte tricks below.)
3. Know the **cell geometry**: frame height is usually fixed; the baseline row; the pen advance per glyph
   (often a field like `ox`/`advance`). Thai needs about 11 rows above the baseline for stacked marks.
4. **Render glyphs with Pillow** from the open Thai font:
   - choose the largest size that fits the font cell (all Thai glyphs, including tone marks raised over upper
     vowels, must stay inside the cell);
   - for small sizes render 1-bit (hinted, `fontmode='1'`) - anti-aliasing merges strokes; for large sizes
     anti-aliased is fine; map coverage to the font's alpha scale;
   - do not squash vertically and do not use precomposed glyph tricks - they blur or do not scale.
5. **Combining marks** (upper/lower vowels: ั ิ ี ึ ื ุ ู ็ ํ ; tone marks and thanthakhat: ่ ้ ๊ ๋ ์):
   store them as **zero-advance glyphs** (pen advance 0). In the byte stream write each mark **before** its
   consonant, so the pen has not moved yet and the mark is drawn over that consonant.
   Order for several marks on one consonant: below-vowel, upper-vowel, then tone mark.
6. **Variants** are needed for good typography. Reserve unused byte values (0x80-0x89 are free in CP874):
   - raised tone marks (used when an upper vowel is already there);
   - left-shifted tone marks (used over tall consonants ป ฝ ฟ ฬ).
   The encoder chooses the variant from the cluster.
7. **Sara am** (ำ): write nikhahit (ํ, zero-advance mark) + sara aa (า).
8. Every font file the UI uses needs Thai glyphs; different screens use different fonts. Generate all of them
   automatically (largest style that fits each cell) and hand-tune only the ones you see broken.
   A screen showing garbage Latin letters instead of Thai = that font id has no Thai yet.
9. Update any header fields that store totals (e.g. total pixel size) when frames change.

### 4.2 Encoding Thai text (Unicode -> engine bytes)

```python
MARKS = set('ัิีึืฺุู็่้๊๋์ํ')
def clusters(s):                      # base character + its combining marks
    out = []
    for c in s:
        if c in MARKS and out and out[-1][0] not in MARKS:
            out[-1][1].append(c)
        else:
            out.append([c, []])
    return out

def encode(s):
    s = s.replace('ำ', 'ํา')          # sara am -> nikhahit + sara aa
    out = bytearray()
    for base, marks in clusters(s):
        for m in marks:                               # marks first (zero advance), then the consonant
            out.append(variant_byte(base, marks, m))  # plain / raised / left-shifted
        out += base.encode('cp874', 'replace') if ord(base) > 127 else base.encode('latin1')
    return bytes(out)
```

`variant_byte` returns the raised code if an upper vowel is present, the left-shifted code if the base is
ป ฝ ฟ ฬ and the mark is a tone, otherwise the normal CP874 byte of the mark.

---

## 5. Line breaking (the engine does not know Thai)

Most engines wrap only at ASCII spaces. Thai has none between words, so long sentences overflow windows.
Insert ordinary spaces at word boundaries **at build time** (never in the stored translation):

```python
from pythainlp.tokenize import word_tokenize
NOBREAK = {'ว่า','ที่','ไว้','นี้','นั้น','ได้','แล้ว','ก็','ๆ','จริง','เลย','อยู่','ไป','มา','ไหม','นะ','ครับ','ค่ะ'}
def space_out(s, limit=12):
    out, run = [], 0
    for t in word_tokenize(s, engine='newmm', keep_whitespace=True):
        if t.isspace() or not any('฀' <= c <= '๿' for c in t):
            out.append(t); run = 0 if (t.isspace() or t[-1] in '.,!?]') else run + len(t); continue
        if run >= limit and t not in NOBREAK:
            out.append(' '); run = 0
        out.append(t); run += len(t)
    return ''.join(out)
```

Then test the widest windows (dialogue box, tooltips, buttons, quest log). Where the box is fixed-width and
the Thai is longer than the original, shorten the Thai.

---

## 6. Build pipeline

```
translations (JSON, Unicode Thai)
   -> normalize terms (one spelling per name)
   -> build: read originals from archives, replace text fields, space_out, encode, write loose files + fonts
   -> check: structural comparison with the originals (must report 0 problems)
   -> package (section 9)
```

Run it the same way every time. Keep a glossary file (names, titles, game terms) and a regex normalizer so a
name never has two spellings.

---

## 7. Testing loop

1. Milestone 0: one conversation/screen in Thai in the real game. Check: marks over letters, no boxes/garbage,
   spaces, wrapping, no crash.
2. Then scale up by area (menus/HUD, first town, main quest, ...). After each: build, check, user plays a short
   checklist, sends window screenshots of anything wrong, fix, repeat.
3. Typical defects: clipped tops of marks, marks colliding, a screen using a font without Thai, overflowing
   boxes, wrong folder (override ignored), patched text not matched, stale files from an older build.

---

## 8. Install / uninstall for end users

- Never overwrite silently: back up every file you replace and write a manifest of what you installed.
- Uninstall = restore backups, delete the rest of the manifest. Do not delete files other mods put in the same folder.
- Auto-detect the game folder (current folder, parent, Steam libraries, GOG paths); allow `--game "path"`.
- Record the game/patch version the text was built for; the patcher must refuse files that differ (see below).
- README: requirements, install, uninstall, what is not translated, troubleshooting (still English -> wrong
  folder or missing community patch; garbled -> another mod overwrote the font).

---

## 9. Packaging as a patcher

Instead of carrying edited game files, package a **patcher** that builds them on the user's machine:

1. Translation table keyed by **position** (file name + row id + field index), not by the original text, so the
   table contains only Thai and numbers. Include for each target file the CRC32 and size of the original.
2. Font data contains only the **new Thai glyph frames** (made from an open Thai font), plus where to put
   them. The patcher opens the user's own font file and inserts the frames.
3. The patcher (a small C# or Python program) does: find game -> read originals from the user's archives in
   override order -> verify CRC -> apply table -> write loose files -> write manifest and backups.
4. Test mode `--out DIR` writes to a scratch folder; compare it byte-for-byte with your reference build
   before every release.
5. Package contents: patcher exe + its source, the two data files, README (TH/EN).
6. Unsigned executables may trigger antivirus warnings. Publish the source, and optionally a hash list.
7. Keep other people's patches and installers out of the package; tell users where to get them.

C# compile without an SDK: `%WINDIR%\Microsoft.NET\Framework64\v4.0.30319\csc.exe -out:Patcher.exe Patcher.cs`
(C# 5 syntax; `System.IO.Compression.DeflateStream` can read zlib data after skipping the 2-byte header).

---

## 10. Notes for AI translators (so the text reads like real Thai)

Machine translation tends to be literal. Rules that help:
- Translate meaning, not word order. Break long English sentences; Thai prefers short clauses.
- Avoid English-style passives ("ถูก...โดย..."), chains of "ซึ่ง", and stacked possessives ("ของ...ของ...").
- Pick one register per character and keep it (royal/archaic vs. plain vs. rough speech); pronouns and
  particles carry the character.
- Idioms: render the intent, not the words. If a joke cannot work, write a Thai joke of the same effect.
- One spelling per proper name; keep a glossary and normalize after every batch.
- Read it aloud: if a Thai speaker would not say it, rewrite it.
- Keep placeholders, brackets and counts identical; never translate command fields.
- Review in small batches with the original beside the Thai; fix only what is unclear or ungrammatical.
