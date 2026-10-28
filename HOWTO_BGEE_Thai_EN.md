# Game-specific guide: putting Thai into Baldur's Gate: Enhanced Edition (v2.7.x)

What was found and done for this game: where the engine reads text and fonts, how to display Thai with correct vowels and tone marks,
line breaking, translation rules, and how to package it. Written for people and for AI assistants.

**Scope:** methods and sample code only. Everything is generated from the reader's own copy of the game.

Target: Baldur's Gate: Enhanced Edition v2.7.3.0 (Steam, Windows), Python 3.10+ with `pillow` and `fonttools`.

---

## 1. Status (honest)

This is a **prototype (Milestone 0)**, not a finished translation.
- Only ~55 strings and ~33 UI keys are translated out of ~34,000 strings and ~700 keys (the rest stays English and the game remains playable).
- **Confirmed in game:** Thai through the ko_KR language slot + a font with Thai glyphs; main-menu button labels; the first narration paragraph;
  the first conversation; correct consonant+vowel+tone cluster glyphs; reasonable font size; line-breaking behaviour.
- **Checked later in game by the user, no problems:** intro cutscene subtitles; character-creation screens; text under the main menu buttons; Thai digits;
  ฎ ฏ with below vowels.
- **Not done:** the remaining translation; text in videos/images/the .exe; keeping the original fantasy fonts; a packaged release.
- Tested on Windows only.

---

## 2. Engine facts (already discovered, do not re-research)

| # | Fact | Status |
|---|---|---|
| 1 | Narration/dialogue/names are in `lang/<lang>/dialog.tlk` (TLK V1, **UTF-8**, ~34,000 strings) referenced by index (strref) | read from files + confirmed in game |
| 2 | Button/menu labels are NOT in the TLK but in the Lua table `L_<LOC>.lua` (resource type 0x409 inside `PATCH20.BIF`), format `KEY = "value",` | confirmed in game |
| 3 | The language list is embedded in `Baldur.exe`; there is no th_TH and you cannot add one, so **borrow an unused slot** | read from files |
| 4 | Borrow **ko_KR**: put the Thai `lang/ko_KR/dialog.tlk` + `override/L_KO_KR.lua` and set `Language Text = ko_KR` in `Baldur.lua` | confirmed in game |
| 5 | v2.x draws text from **TTF fonts** stored in BIF (resource type 0x40A; 20 of them, e.g. NANUM, NORMAL, STONEBIG, TOOLTIP, DIALOG, FLOATTXT ...) | read from files |
| 6 | A file `NAME.TTF` in `<game>\override\` replaces the font of that name | confirmed in game |
| 7 | In ko_KR mode the whole UI uses the single NANUM font (so it looks different from the English mode with several fantasy fonts) | confirmed in game |
| 8 | Old `.FNT` + `.bmp` fonts in `lang/*/fonts` predate patch 2.0; v2.7 does **not** use them (a Thai font made that way showed empty boxes) | confirmed in game |
| 9 | No text shaping: pre-combine consonant+vowel+tone into one glyph | confirmed in game |
| 10 | A glyph taller than the font `ascent` is clipped at the top; fix by raising ascent to 1180 | confirmed in game |
| 11 | The game wraps at spaces, and breaks inside a run at any character when too long; U+200B has no effect | confirmed in game |

Text outside TLK/Lua that is untouched: text inside videos, images with text, hard-coded .exe strings, text in `UI.lua`/`BGEE.lua` that does not
go through `L_*`, and voice acting. Report these to the user; never skip silently.

---

## 3. File formats

All numbers little-endian (see the community IESDP documents).

- **chitin.key (KEY V1):** 24-byte header (`'KEY '`, `'V1  '`, BIF count, resource count, BIF table offset, resource table offset); BIF entries 12 bytes;
  resource entries 14 bytes (`name[8]`, `type u16`, `locator u32`; `locator >> 20` = BIF index, `locator & 0x3FFF` = file index in the BIF).
- **BIF:** 20-byte header (`'BIFF'`, `'V1  '`, file count, tileset count, file table offset); 16-byte file entries (locator, offset, size, type, unk).
- Relevant resource types: `0x40A` TTF font, `0x409` Lua (`L_*`, `BGEE`, `UTIL`, `INCLUDES`), `0x408` Lua (`UI`), `0x400` old FNT, `0x001` BMP.
- **Priority:** files in `<game>\override\` beat the BIF.
- **dialog.tlk (TLK V1):** 18-byte header (`'TLK '`, `'V1  '`, lang u16, count u32, string data offset u32); 26-byte entries (flags u16, sound[8], volume, pitch, offset, length);
  text is UTF-8 without NUL. strref = entry index from 0. Write back by changing only the text, keep count and order; do not touch `sound` of voiced entries.
- **Tokens in text to keep:** `<CHARNAME>`, `<PRO_HESHE>`, `<PRO_HIMHER>`, `<PRO_HISHER>`, `<GABBER>`, `<PRO_SIRMAAM>`, `<LADYLORD>` etc. and `\n`.
- **L_<LOC>.lua:** `uiStrings = { KEY = "value", ... }` with the same keys in every language. Build `L_KO_KR.lua` from `L_EN_US.lua` and replace only the keys you translated
  (untranslated ones then show English, not Korean).
- **Baldur.lua** (`Documents\Baldur's Gate - Enhanced Edition\`): `Language Text`, `[Fonts] ko_KR = NANUM`, `FontZoom`. The game rewrites it when it closes, so edit only while the game is closed.

---

## 4. Thai display: cluster glyphs in the PUA

1. Split text into units: a plain character, or a cluster (consonant, upper vowel, tone, lower vowel).
2. Clusters with marks map to private-use code points U+E000, E001, ... (recorded in `translations/clusters.json`, rebuilt on every text build).
3. `build_ttf.py` makes a glyph for every code by composing the consonant outline + the positioned marks into one shape.
4. `ำ` is split into nikhahit (inside the cluster glyph) + `า`.
5. A mark without a leading consonant (typo) is placed on a no-break space (U+00A0), 0.35 em wide.
6. Thai text without marks uses the plain Unicode Thai glyphs U+0E01-0E5B added to the font.

Downside: the built `dialog.tlk` contains PUA characters, so it is no longer standard Thai; keep the source translations as plain Unicode Thai and never feed the built file to other tools.

### 4.1 Font and glyph insertion
- Thai source font: **Sarabun** (open Thai font). Download from `raw.githubusercontent.com/google/fonts/main/ofl/sarabun/` (ask the user first, every time).
- Open the game's `NANUM.ttf` (TrueType, 1000 units/em) with fontTools, draw outlines from Sarabun through `TransformPen`, store them as new glyphs, and add:
  `glyf`, `glyphOrder`, `hmtx`, `cmap` (all format 4/12 subtables), `maxp.numGlyphs`.
- Delete tables tied to the old glyph count: `hdmx`, `VDMX`, `LTSH`, `DSIG`, `gasp`; set `post.formatType = 3.0`.
- Source composite glyphs must be decomposed first (`DecomposingRecordingPen`), otherwise `TTGlyphPen` fails.
- Decided values:

| Value | Used | Reason |
|---|---|---|
| ASCENT (`hhea.ascent`, `OS/2.sTypoAscender`, `usWinAscent`) | 1180, `lineGap = 0` | tallest glyph is about 1172; lower values clip the top |
| THAI_SCALE | 1.06 (relative to NANUM's em) | at 0.92 the user found it too small |
| MARK_SCALE | 0.85 (scaled about the anchor point) | keeps stacks from getting too tall |
| MARK_DROP | 40 units (upper marks) | lowers them |
| GAP | 35 units | space of tone mark above the vowel top |
| advance of space (U+0020) | 160 | Thai text hardly uses spaces; wide spaces look wrong |

- Use the variant glyphs that Sarabun provides: `uniXXXX.small` (tone over an upper vowel; lower vowels under ญ ฐ), `.narrow` (shifted left for ป ฝ ฟ ฬ).
- Not yet checked: ฤๅ ฦๅ and rare marks (ฎ ฏ + lower vowels and Thai digits were checked in game by the user without problems).
- Rendering differs per font size because of engine hinting; check every size the game uses.

### 4.2 Line breaking (five methods were tested; the simplest won)
- **No automatic spacing.** The game breaks inside long runs itself.
- Put one space after `. , ; : ! ? ...` when followed by Thai (the tool does it).
- Translators add natural spaces (between sentences or phrases).
- Did not work / abandoned: pythainlp word segmentation (wrongly split proper names), a space after every word or every 12 characters, U+200B, moving Thai into CJK/Hangul ranges.

---

## 5. Pipeline

```
user's game (chitin.key + data/*.BIF + lang/en_US/dialog.tlk)
   extract_game_files.py -> work/ttf/NANUM.ttf , work/lua/L_EN_US.lua        (game files: never ship)
translations/tlk_by_strref.json + ui_lua.json   (Thai translations, key = strref / Lua variable name)
   build_text.py -> out/lang/ko_KR/dialog.tlk , out/override/L_KO_KR.lua , translations/clusters.json
fonts_thai/Sarabun-Regular.ttf  (fetch_fonts.py downloads from the original source)
   build_ttf.py NANUM -> out/override/NANUM.TTF
   install.py install -> <game>/lang/ko_KR/dialog.tlk , <game>/override/{L_KO_KR.lua,NANUM.TTF} , Baldur.lua: Language=ko_KR
```
- `build_text.py` must run **before** `build_ttf.py` because cluster glyphs are created only for clusters that occur in the text.
- Other tools: `find_strings.py "regex"` (find a strref from on-screen English), `export_todo.py --size 150` (split into jobs),
  `check_translations.py f.json [--merge]` (check tokens/newlines/characters then merge), `preview_ttf.py` (render a sample image without starting the game),
  `install.py install|uninstall` (backs up into `backup/`).
- Set `PYTHONIOENCODING=utf-8`, call with `py -3 -I` (isolated). Thai/PUA output cannot be read in the console; verify with rendered images.

---

## 6. Translation

### 6.1 Priority
1. Button/menu labels (`ui_lua.json`, ~700 keys): the first thing players see
2. character creation, race/class/ability descriptions
3. main dialogue (by area/character), cutscene narration
4. items, spells, NPC/place names, quests/journal
5. system messages (combat, errors)

### 6.2 Batches
- `export_todo.py --size 150` writes `work/todo/part_NNNN.jsonl` (one `{"id": strref, "en": "..."}` per line). These jobs contain game text; do not share them.
- The translator (human or AI) returns one JSON object `{"24630": "Thai", ...}`.
- `check_translations.py result.json` checks token counts, newlines, characters; then `--merge` into `tlk_by_strref.json`.
- Estimate: ~33,000 strings / 150 = ~220 batches; plan weeks, not hours; have a Thai reader spot-check 5-10% of each batch.

### 6.3 Rules
1. Keep every token and `\n` exactly, same counts. Thai pronouns have no gender like English; choose sentences that stay correct for gender tokens.
2. Do not translate codes/commands/file names/empty text.
3. Proper names come from `glossary.json`; one name, one spelling game-wide; transliterate new names per the Royal Institute rules and add them to the glossary at once.
4. System terms (attack bonus, armour, saving throw, hit points...) are fixed once in the glossary.
5. Dialogue uses fantasy-fiction Thai (ข้า/เจ้า/ท่าน by the speaker's rank); UI/system uses short plain Thai.
6. Button labels as short as the English; shorten the translation instead of changing the font.
7. Keep formulas and numbers (`1d8+3`, `+2`); the same text under several strrefs gets the same translation.
8. Voice acting stays English; subtitles must carry the same meaning.
9. Translations that will be shared are keyed by **strref**, never by the English text.

---

## 7. Install, test, package

- The user must close the game before installing (the game rewrites `Baldur.lua` on exit). The installer backs up first and has `uninstall`.
  The language to pick in game is Korean (the visible text is Thai).
- The kit deliberately has **no double-click `.bat`/`.exe` installer**, to prevent accidental installs; the install command still asks for YES.
  If a `.bat` is ever needed it must be pure ASCII (Thai text with `chcp 65001` made cmd misread lines).
- Testing: the user plays and sends screenshots of **the game window only** (never the whole desktop). Always check on real screens, because length and wrapping only show in play.
  Symptoms: boxes/missing characters = cluster glyphs not built (forgot `build_text.py` then `build_ttf.py`); tops of tone marks cut = ascent too low.
- Caveats: Steam may restore `lang/ko_KR/` on verify/update, so reinstall (assumption, unconfirmed); a game update may change strrefs, so extract again and recheck.
- **Package contents:** tools (`tools/`), documents, translations keyed by strref.
  **Keep out of the package (local working files):** anything extracted from the game (`work/`), build outputs (`out/`), backups (`backup/`).
  For Sarabun, link to its source or include the font file.

## 8. Dead ends (do not retry)

| Topic | Result |
|---|---|
| `.FNT` + `.bmp` fonts | wrong path: v2.7 uses TTF |
| Tahoma | looks good, but replaced by Sarabun (open font) |
| automatic segmentation + a space per word / every 12 characters | misplaced spaces, split proper names -> dropped |
| U+200B, CJK/Hangul code ranges | no effect |
| raising marks by hand according to mark height | stacks too tall, clipped by ascent -> use Sarabun's variant glyphs + scale/drop marks |
| editing the .exe / injecting a DLL | not done |

## 9. An option not tried yet
Play in en_US and add Thai to all ~11 TTF fonts of the English UI, to keep the original look. This needs a Thai `dialog.tlk` for en_US,
i.e. editing the English language file directly. Not tried; decide with the user first.
