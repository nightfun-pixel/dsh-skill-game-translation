[English](README.md) · [Русский](README.ru.md)

# game-translation

A [Claude Code skill](https://docs.claude.com/en/docs/claude-code/skills) for translating/localizing a
game build. It does not replace a project's own translation pipeline; it is the checklist that stops
an agent from reporting a build "translated" after only the main pass finished, when the source
language is actually still present in a layer the main pass does not look at.

## Why this exists

A game build hides text in more than one place at once: a string table, a compiled script, a
serialized data blob, and a texture with the text drawn into its pixels can all carry text in the same
build. A pipeline that handles the first of those cleanly and finishes without errors reports success
truthfully — and the build can still ship with text untranslated in the other three, because nothing
checked them. This skill is the scan that checks all four before "done" is said out loud, plus the
translation discipline and the per-layer fix recipes the scan's findings usually call for.

It was written after a session that translated a Unity/IL2CPP build by hand — extracting strings from
serialized bundles, patching compiled-in string literals, redrawing text baked into UI textures, and
flipping data-driven unlock flags — because an earlier automated pass had left roughly 40% of the
game's text in the source language, in layers the first pass's scan never reached. The skill
generalizes that session's method and its specific failures into a container-agnostic checklist.

## What it covers

| Step | Content |
| --- | --- |
| 0 | Use an existing automated pipeline first, when the project has one; fall back to the manual steps only for what it reports it did not handle. |
| 1 | Where text hides, by layer: string tables, compiled/bytecode scripts, serialized data blobs, code-level string literals, text baked into images — with why a naive scan misses each one. |
| 2 | Translating through an LLM in batches: JSON-mode calls, a glossary, per-string length limits, answer validation, reasoning on the retry pass only, and telling a region block apart from a content refusal. |
| 3 | Applying translations: index/offset rewrites over in-place byte patches, keeping any side index (pack directory, bundle catalog, string-literal table) in sync with what it indexes, and three concrete mistakes that silently drop or corrupt strings. |
| 4 | The gate: re-run the step-1 scan on the actual built output, not the source assets, and treat any nonzero count as unfinished work, not a note. |
| 5 | Verifying the build by running it, within limits — covering every UI screen the translation touched without the agent playing itself into content it should not generate or describe. |
| 6 | Reporting what's translated (with its scan count), what's deliberately untouched, and what's unverified. |

## Install

Drop the folder where the harness reads user skills and restart, or let it pick the skill up on its
own if it watches the directory:

```
cp -r game-translation <skills-root>/game-translation
```

`<skills-root>` is `~/.claude/skills` for Claude Code, and the user-skill root the skill shop in a DSH
(`@deepseek-ai/dsh`) profile reports for its `user-dsh` target otherwise.

## Scope

This is a checklist for an agent to follow, not a script to run — it names the checks and the recipes
in prose because the containers it applies to (engine string tables, compiled scripts, asset bundles,
textures) differ enough between projects that a single fixed script cannot cover all of them. A
project with its own automated pipeline (detect → extract → translate → apply → integrity gate →
repack) should keep using it; this skill is what fills the gap between that pipeline's own report and
an actually complete translation.

## Why it is built this way

The step-4 gate is the one rule everything else in the skill exists to make actionable: scanning again
on the built output, not the source assets, is what catches a translation that was correct in the
intermediate file but never reached the package, or a rebuild step that silently reverted a change.
Everything upstream of it (where text hides, how to translate it, how to apply it without corrupting a
side index) exists to give that final scan as few real misses as possible to report.
