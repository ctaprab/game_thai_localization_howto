# Game-specific guide: putting Thai into Vaporum (Unity 5.6.5 + old TextMeshPro)

What was actually done for this game: how Thai is displayed, how the text was categorised and translated, how the patch is built,
and how it is packaged. Written for people and for AI assistants.

**Scope:** methods and sample code only. Everything is generated from the reader's own copy of the game.

Target: Vaporum (Steam, FullRelease Build #419), Unity 5.6.5p4, Windows. No Python and no Unity Editor needed
(C# via the built-in `csc.exe`, and PowerShell 5.1).

---

## 1. Status (honest)

| Part | Status |
|---|---|
| Thai rendering in game | confirmed by the user with in-game screenshots (UI, tooltips, subtitles) |
| Translation | 1,754 of 1,894 entries (~24,700 words); the rest intentionally English (keyboard key names, V-Sync, Ambient Occlusion, console brand names...) |
| Checked in game | general UI, stats window, notes/subtitles (tall marks were clipped at the top, partly fixed, see 3.1) |
| Not checked in game | the protected-terms word-break guard (verified in preview only) |
| Packaging | a delta patch; tested against a simulated game folder, not on a clean machine |

---

## 2. Game structure

### 2.1 Text
The game has a user-extensible language system: `Localization/CustomLanguages/<Name>.txt`
- format: KEY line / VALUE line / blank line, repeated; UTF-8 **with BOM**, CRLF line endings;
- keep exactly: tags `<style=..>` `<color=..>` `<align=..>` `<b>`, placeholders `{0}` `{0:+0;-0}`, `\n`, numeric codes;
- because of this, **no game code needs to change**; the text is one file. What must be patched is the **font**.
- select it in game: Options > Game Options > Language, then restart.

### 2.2 Fonts
- TextMeshPro (old), static SDF atlases, no Thai. The main font is Latin and is used for all languages except Japanese/Chinese.
- Fonts live in `<Game>_Data/resources.assets` (release builds have no typetrees). Read the TMP_FontAsset object structure with the
  AssetStudio libraries (v0.16.x) and cache the typetree (`fonttree.cache`).
- SerializedFile v17: metadata little-endian, header big-endian, alignment 16 (header) / 8 (data). Objects are replaced or added by pathID;
  TypeId indexes the type table.
- Font object path IDs involved (re-check after game updates): 22744-22749 and 22767-22768.

---

## 3. Display strategy (why clusters are pre-rendered)

Old TextMeshPro has **no OpenType shaping (GPOS)**, so vowels and tone marks are not positioned. What works:

1. Create **one new TMP font asset** from an open Thai font (e.g. Sarabun) and add it to the `fallbackFontAssets` of every
   TMP font the game uses. The Latin fonts stay primary, so Latin text is untouched.
2. Pre-render every **consonant + vowel + tone cluster as ONE glyph** (GDI+ shapes it with the real Thai font) and map clusters to
   Private Use Area code points U+E000 and up (`clustermap.txt`). The compile step rewrites the Thai text with these PUA characters.
   - `clustermap.txt` is **append-only**; never delete or reorder or already-installed patches break.
   - Clusters exist only for text present in the language file, so after translating more you must compile + patch again.
3. **Word wrap:** Thai has no spaces; insert U+200B (zero-width space) at word boundaries using Windows ICU BreakIterator (`icu.dll`).
   TMP breaks lines at U+200B. `protected_terms.txt` stops ICU splitting game terms and names.
4. SDF atlas: Alpha8, 2048x2048, PointSize 32, Padding 5, GradientScale 6, pixel value = `0.5 + distance/(2*6)`; raw row 0 is the bottom
   while TMP glyph y counts from the top.
5. Always rebuild from the **original** `resources.assets` (backup it as `_ThaiBackup/resources.assets.ORIGINAL` and check its SHA-256).

### 3.1 Clipped top marks (important)
UI inside ScrollView / RectMask2D clips whatever exceeds the line box. A tone mark stacked on an upper vowel disappears at the top of the
first line (a word like "นี่" shows as "นี"), because Sarabun stacks exceed the Latin base font's line box (ascender 30.4 / descender -7.6 at 32 pt, times the font scale).
- Applied fix: render at em 30 and squash only the mark zone so ink top <= 29 and bottom >= -7.4 (env `TF_MAXTOP`, `TF_MINBOTTOM`; the `tf stats` command measures).
- The user's last screenshot suggests the real ceiling is lower (~25.6). If marks still vanish, try em 28-30 with `TF_MAXTOP=25 TF_MINBOTTOM=-6.2` (marks become flatter).

---

## 4. Pipeline

```
parts\th_part*.txt  (translations: key / value / blank; later files override earlier ones)
   -> merge (over the original EN.txt)
   -> tf compile  (ICU word breaks as U+200B + cluster -> PUA map + protected_terms)
   -> tf patch <game> <ORIGINAL> <out> <ttf> <em> clustermap atlas.png   (build atlas + edit resources.assets)
   -> tf export-payload ORIGINAL patched out.vtp                          (delta patch)
```
- `build_all.ps1 -Game <game folder> [-Install]` does everything (rebuilds are byte-identical); `make_release.ps1` builds the release from `release_template`.
- `tf.exe` commands: `compile`, `patch`, `preview`, `stats`, `glyphtest`, `show`, `export-payload`.

PowerShell 5.1 pitfalls:
- keep scripts ASCII only; Thai text lives in UTF-8 data files read with an explicit encoding (PS 5.1 mangles Thai in BOM-less .ps1);
- variables are case-insensitive ($L vs $l) and aliases (Cat...) collide with names;
- regex-replacing in an LF-only file can collapse lines; rewrite the file instead;
- call `powershell -ExecutionPolicy Bypass -File ...`.

---

## 5. Translation

### 5.1 Categorise first
All 1,894 entries were classified automatically (`classify.ps1`) into `CATEGORIES.md` (summary, per-category context, decisions to confirm)
and `text_index.tsv` (category / key / line / words / done / English / Thai; the user can edit the category column).
The user reviews categories and proposals **once**, then each category is translated in turn.

| Group | Example categories |
|---|---|
| tutorial / menus / settings | A1, A2 |
| level names / signs | B1, B2 |
| system messages / status effects | C1, C2 |
| stats window, circuits, modules, bonuses, rigs | D1-D4, R1 |
| item types, weapons, armour, gadgets, consumables, quest items | E1, I1-I5 |
| enemy names / scene containers | R2, R3 |
| notes and audio diaries (main story) | F1 |
| character lines (voiced/subtitles) | F2 |
| in-game enemy encyclopedia | G1 |

Suggested order: UI/stats -> items -> lines and statuses -> others -> notes -> encyclopedia (by volume and how often the player sees them).

### 5.2 Per-category steps
1. Read the category context + the game-context file (`GAME_CONTEXT.md`: summary, characters and voices, glossary, style rules).
2. Translate into `parts/th_part_<category>.txt`.
3. Scan for translationese (การ/ซึ่ง/ถูก/สามารถ ...) and rewrite.
4. For frequently seen or voiced categories show a 20-30 line sample table (English / Thai / alternatives) for the user to choose.
5. compile -> patch -> check -> back up -> install (game closed) -> user tests.
6. An Excel sheet (English / Thai / status / suggested fix) lets humans review; import the result as `th_part_review.txt`.

### 5.3 Style (the user wants Thai that people really say, not word-for-word)
- readable in one pass; translate meaning, restructure into Thai sentences;
- use words Thai players use (ตั้งค่า, เซฟ/โหลด, ดาเมจ, คริติคอล, เลเวลอัป, กระเป๋า);
- menus/buttons: 1-3 words, direct verbs;
- tooltips: short sentences, drop the subject, "คุณ" only when needed, friendly-polite tone;
- character lines should sound spoken, per the character table (endings and interjections); no archaic style since the setting is modern steampunk;
- note signatures "name, title": translate the title, transliterate the name;
- Arabic numerals; units as in the original; keep reference jokes' names;
- one term, one translation game-wide; add new terms to the glossary before using them;
- examples of decisions: Options = ตั้งค่า, Inventory = กระเป๋า, Apply = นำไปใช้, keyboard key names stay English,
  button hints "กด {0} เพื่อ..." and "<คลิกขวา> เพื่อ...";
- narrow spaces: use shorter words.

---

## 6. Install and package

Instead of the modified `resources.assets` (~140 MB), the package carries a **delta patch**:
- `payload.vtp` (gzip, ~0.25 MB): magic `VTP1`, SHA of the original and of the patched file, counts, replaced objects (pathId, length, bytes),
  added objects (pathId, typeId, length, bytes).
- The open-source patcher (`PatcherMain.cs`, `SF.cs`) **refuses** any input whose SHA is not the expected original (no override mode). `--list payload` shows which objects the patch touches.
- Installer (`Install.bat` -> `helper/Install.ps1`): locate the game via the Steam registry + `libraryfolders.vdf`, refuse if the game is running,
  back up the original, patch to a temp file, verify the SHA, move it in, copy `Thai.txt`; `Uninstall.bat` restores the backup.
- The installer compiles the patcher source locally with `csc.exe` (Framework64 v4.0.30319); the bundled exe is only a fallback.
- Use ASCII file names in the release (Thai names in zips from Compress-Archive can break); name the reader file `START_HERE_TH.txt`.
- Test installers on a simulated game folder (`-SkipRunningCheck`), never on the live folder.
- Release contents: the Thai font, README (TH/EN), AUDIT.txt.
- A game update changes `resources.assets`; rebuild the patch from the new original and re-check the font object paths.

---

## 7. Pitfalls actually met

- Masked UI (subtitles, notes) clips tall marks (section 3.1); test in screens that have a mask.
- A TMP font without the fallback shows blanks or boxes on that screen.
- You cannot build the patch from an already patched file: use Steam > Verify integrity or pass `-Original`.
- The credit line in the README must be filled by the project owner.
- The user may type Thai on an English keyboard layout by mistake; decode with the Kedmanee layout.
