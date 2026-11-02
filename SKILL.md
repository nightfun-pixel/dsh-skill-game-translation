---
name: game-translation
description: Use when asked to translate/localize a game build (APK, archive, or source tree) into another language, including when a prior pass (another model, another tool) claims it is done. Defines the gate that makes "done" mean zero source-script text left anywhere, not "the main pass finished": a mandatory residual-text scan across every layer a game can hide text in, the LLM-batch translation discipline, and container-specific fix recipes for text the obvious scan misses.
license: MIT
---

# Game translation

A checklist, not a framework. It exists because "translated" and "the translation pass completed
without errors" are different claims — a pass can finish clean and still leave text behind, because
most game formats hide text in more than one place at once: a string table, a compiled script, a
serialized data blob, and a texture with the text drawn into its pixels can all exist in the same
build. A translator that only looks at the first of those reports success truthfully and is still
wrong.

Follow the steps in order. Do not skip the scan in step 4 because step 3 "went fine" — the two are
independent claims.

## 0. Check for an existing pipeline first

If the project already has an automated translate pipeline (detect engine → extract → translate →
apply → integrity gate → repack), run that first and read what it reports, rather than rebuilding any
of it by hand. Fall through to the manual steps below only for what the pipeline's own report says it
did not handle: an unsupported engine, a fallback to a generic/best-effort adapter, or any nonzero
count from the residual scan in step 4. The manual steps are also the right starting point when there
is no pipeline at all.

## 1. Find every place text can hide

Do not stop at the first container format you recognize. For the kinds of builds this applies to
(Ren'Py, Godot, Unity/IL2CPP, RPG Maker, and similar), check each of these independently — a build
missing from one does not mean it is missing from the others:

| Layer | Where it hides | Why a naive scan misses it |
| --- | --- | --- |
| Structured string tables | `.rpy`/translation files, resource string tables, JSON/CSV string files | Usually found; the easy case |
| Compiled/bytecode scripts | `.rpyc`, `.gdc`, IL2CPP method bodies | Often compressed or tokenized — a raw byte search over the archive finds nothing even though the string is right there, because the bytes on disk are not the UTF-8 text |
| Serialized data blobs | `MonoBehaviour`/`TextAsset`-style objects, nested dict/array payloads, dialogue-command tables | Easy to find by key name (`text`, `label`) and easy to miss everything that *isn't* under that key — walk the structure recursively and test every string leaf, not a fixed list of field names |
| Code-level literals | Metadata/string-literal tables compiled into the binary | Invisible to anything that only reads game assets, because the string lives in the managed/native runtime's own data, not in a game file |
| Baked-into-image text | UI chrome, buttons, tooltips, loading-screen frames rendered as pixels | No text scanner of any kind finds this. The only way to find it is to look: dump every UI-sized texture, make contact sheets, and read them |

For the source-script scan itself, match the Unicode ranges of the source language rather than
grepping for specific words — for Japanese: Hiragana `぀-ゟ`, Katakana `゠-ヿ`,
CJK Unified Ideographs `一-鿿`, fullwidth forms `＀-￯`. Apply the same range check to
every layer in the table above, including decoded/decompiled text and object-tree string leaves, not
only raw file bytes.

## 2. Translate through the configured LLM, not your own wording

Extract strings, build context and a glossary, call the batch translator, validate the answers. Do
not substitute your own prose for game text — that is a second, unreviewed translation living next to
the first, and it drifts from the glossary and from terms the rest of the game already uses.

- **Batch through JSON mode.** Send arrays of `{id, source, context, max_len}`, request a JSON object
  keyed by id back, chunk at a size the model's `max_tokens` can actually return (a batch that can
  overflow the output budget comes back truncated or empty, and an empty batch is silent data loss —
  check every id you sent for a key in what came back).
- **Carry a glossary of recurring terms** (character/place/mechanic names, currency, recurring UI
  words) into every batch so the same source string gets the same translation everywhere.
- **Give every UI string a hard character limit** derived from the box or button it fills, and reject
  an answer that exceeds it.
- **Validate before applying:** the answer contains at least one character in the target script
  (unless the source was punctuation/placeholder-only), any `{0}`/`%s`/`<tag>`-style placeholders from
  the source are present unchanged, and the length limit holds. On failure, retry once with more
  explicit instructions before falling back to anything else.
- **Reasoning on the retry pass only.** A "thinking" pass is slower and costlier and can spend its
  entire token budget on reasoning and return nothing for the translation itself. Run the first pass
  with reasoning off and enable it only for the strings that failed.
- **A flat 403 on every request, regardless of model**, is a network/region block, not the provider
  refusing content — check whether the project has an HTTP proxy configured before concluding the
  endpoint rejected the request.

## 3. Apply, preferring index rewrites over in-place byte patches

Prefer changing *what a string's slot points to* over overwriting bytes at a fixed offset. A
translated string is essentially never the same byte length as the source, and almost every compact
or compiled format records lengths or offsets somewhere — rewrite that table along with the string
rather than trying to keep the replacement the same length.

- If a container has a side index that duplicates each entry's size and/or a hash/checksum (an asset
  pack directory, a bundle catalog, a zip central directory) separate from the entry's own bytes, the
  runtime trusts *that index*, not the entry. Changing an entry's content without updating its index
  row produces a build that looks correct in every tool that reads the entry directly and fails only
  in the game's own loader — regenerate every row whose size or hash changed, not just the literal
  table.
- If a string lives in a managed/native runtime's string-literal pool, append the new literal rather
  than mutating one in place, and update whatever derived id (hash, GUID, version string) the runtime
  uses to decide whether its on-device cache of that data is stale — otherwise a device that already
  ran the build keeps using its cached copy and the patch has no visible effect there.
- For text baked into a texture: erase the source pixels (fill with the surrounding background, not a
  flat color, so the result keeps the art's texture/grain) and draw the translation using a font
  asset already shipped in the build if one covers the target script, matching the original's size
  and alignment; shrink or enable auto-sizing for short labels whose box was sized for the shorter
  source-language text.
- **Don't touch a string used anywhere as a comparison or lookup key** (`if x == "..."`, a dictionary
  key, a node/object name another part of the game references by that exact text) — translating it
  breaks the comparison. Collect every such usage across the whole project first, and exclude those
  exact strings from translation everywhere, not just at the site where you noticed the comparison.
- **Don't apply a single-word denylist that rejects every one-word string** on the theory that it must
  be an identifier — that also rejects real one-word UI labels (`Settings`, `Quit`, `Continue`).
  Accept a one-word string when it is Title Case or ALL CAPS; keep rejecting camelCase/snake_case,
  which read as identifiers.
- **Don't drop a string because it contains a newline** — a naive "printable characters only" filter
  in some languages treats a string with an embedded newline as non-printable and silently discards
  it, which loses exactly the long descriptive text (not the short UI labels) without raising an
  error.

## 4. The gate: scan again, on the built output, before calling it done

Re-run the step-1 scan against the actual files you shipped (the rebuilt archive/package), not the
source assets you translated from. Report an exact count per layer. A nonzero count is not a note for
later — it means go back to step 1 for that layer, translate what step 1 missed, and rebuild. The
scan has to include the image sweep: render or dump the UI textures again and look, because nothing
earlier in the pipeline can verify that step on its own.

Keep a short running log of what was checked, by file/bundle name, and its count, so a later session
(or a retry after this one gets interrupted) does not have to re-derive what was already covered.

## 5. Verify by running the build, within limits

Install and walk the build far enough to see every screen the translation touched: title, settings,
menus, HUD, any unlock/gallery/bonus screen you were asked to open. Stop there. Do not drive the game
further into story content on your own initiative — if the next screen is content you should not be
generating or describing (explicit or otherwise sensitive material), stop and tell the user what's
next and let them continue from there themselves, rather than playing through it to "finish checking."
Checking that a cheat/unlock *works on data you can read* (the value in the file, or in an emulated
run of the exact instructions you wrote) is a legitimate substitute for playing combat or late-game
content to confirm it live.

## 6. Report honestly

State, in one pass:

- What's translated, and by which scan in step 4 confirmed it (counts, not "looks done").
- What's deliberately left alone, and why — a logo baked into promotional art, an author's branding,
  anything you judged was not meant to be touched.
- What's unverified, and why (a feature behind content you stopped before reaching, a scan that
  can't run without a real device).
- Exact output file paths.

A claim with no scan behind it is a guess. Say which parts are which.
