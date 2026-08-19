# Agent Note: Browser translators rewrite React-owned DOM and kill slot entries

Status: implemented

English | [中文](2026-08-19-translator-dom-rewrites-kill-slot-entries.zh.md)

## Problem

A Chrome profile with an active DOM translator — the built-in translate or an extension such as immersive-translate — corrupted the conversation composer two ways. The composer paints draft glyphs in a backdrop `<div>` of raw text nodes beside a transparent textarea and a hidden mirror ([one shared scrollport](2026-07-31-composer-text-layers-share-one-scrollport.md)), so a translator that rewrites text nodes rewrites the backdrop's children underneath React.

While the backdrop stayed mounted, React kept updating detached text nodes: keystrokes were captured and submitted but painted nowhere. On submit the draft cleared, React's deletion commit tried to remove a child the translator had already replaced, and the commit threw `NotFoundError: Failed to execute 'removeChild' on 'Node'`. The per-entry boundary caught the throw and logged `slot entry crashed in 'conversation.composer.bar'`; the single-kind entry abdicated permanently and rendered its unstyled empty `<div data-slot-error>` in place of the composer bar until refresh.

The page declared `<html lang="zh-CN">` with no translation refusal — exactly the state a translator acts on for a reader whose UI language disagrees with the document language.

## Decision

`apps/web/index.html` declares `translate="no"` on `<html>` and carries `<meta name="google" content="notranslate">`. The app ships its own zh/en locale switcher, so page-level translation has no legitimate job here; refusing it removes the whole class of translator rewrites instead of chasing the affected slots. `InputBar`'s text-layer stack (the `.grow` wrapper around backdrop, textarea, and mirror) additionally carries its own `translate="no"`, so the draft stays untouched even where a host document never adopted the page-level refusal.

The slot crash face is now visible. `packages/client/web/src/base.css` styles `[data-slot-error]` with a dashed error outline, a minimum height, and a warning glyph painted by a `::before` pseudo-element — pseudo-element content is not a text node, so the marker itself survives a translator. A crashed entry can no longer masquerade as a vanished composer; the failure names its slot until refresh.

## Verification

Component suites cover the guarded paths end to end: the composer (`input-bar`, 68 tests), the goal editor including the IME composition Enter (`goalbar`), and — riding the same change — the popup select, queue dock, workspace rename, and directory browser composition guards (239 tests green across the six suites; `tsc -b tsconfig.client.json` and the repository oxlint configuration clean). The slot crash face is verified by inspection of the built stylesheet served by `dsh web`.

The assembled app was exercised against the browser profile that reproduced both symptoms, translator active: a multi-line draft stayed visible keystroke by keystroke, and the composer survived submit. Before the change, the same profile produced the invisible draft and the vanished composer on nearly every submit.

## Alternatives considered

**Per-slot `translate="no"` only, no page-level refusal.** Rejected as the sole measure: the built-in translator and extensions differ in which hints they honor, and the composer is not the only React-owned text. The page-level refusal is the broad defense; the composer's own attribute is the local guarantee.

**Recovery or retry for abdicated single-kind entries.** Rejected for this change: abdication exists so a crashing entry cannot loop. Fail-visible is the minimal honest face for an unexpected crash; a recovery policy deserves its own decision if a real crash loop ever appears.

**Intercepting React's DOM mutations.** Rejected: the framework owns its commit path, and defending foreign DOM mutation at the framework layer is disproportionate to refusing the mutation at its source.

## Consequences

Translators no longer see translatable content at the document level; a reader who wants the UI in another language uses the built-in locale switcher. The crash-face styling applies to every slot in the shell, so any future entry crash is visible rather than silent. Draft text inside the composer is never a translation subject — it is user-authored input. The same change completes the IME composition-guard sweep (`isComposing`, the legacy `keyCode 229`, and Safari's compositionend-then-keydown ordering, via the deferred ref clear) across `GoalBar`, `PopupSelectView`, `QueueDock`, the `WorkspaceBrowser` rename inputs, and `DirectoryBrowser`'s path and folder inputs — the guard `InputBar` and `QuestionComposer` already carried.
