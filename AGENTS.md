# AGENTS.md

## Repo shape

- Everything lives in one file: `sharedSecret.html` (~5.7k lines). No package manager, build, linter, test suite, or CI. Verification = open it in a browser.
- Internal layout (line numbers drift; locate by these markers):
  - `<style>` block (~lines 8–649).
  - Markup: Generate/Recover tabs, QR display dialog, camera scan dialog, footer with a hand-maintained version string (`version 2.0.2 · 2026-11-12`, search `app-footer`).
  - Vendored libraries, each preceded by a `<!-- Source: ... -->` comment: `secrets.js` (grempe, Shamir sharing), `lz-string`, three StackOverflow snippets, `qrcode-generator`, minified `jsQR`.
  - App code: the final `<script>` marked `<!-- By: Fabrício Silva -->`. All app functions live there and are wired through inline `onclick` attributes in the markup — renaming a function requires updating the HTML too.
- **Rule: never modify any `<script>` block except the last one** (the app code above). All fixes and features go in that final block; every other script block stays byte-for-byte untouched — no edits, reformatting, or reordering.
- `String.prototype.hexEncode`/`hexDecode` (one of the StackOverflow snippets) are load-bearing: the share pipeline depends on them.

## Running / verifying

- Generate/recover work from `file://` — just open `sharedSecret.html`. There are no network calls; all processing is local.
- Camera QR scanning needs `getUserMedia`, i.e. a secure context. Serve over localhost to test it (`python3 -m http.server`), matching the app's own failure message.
- No automated tests. Manual smoke test: Generate (e.g. 10 tokens / 2 minimum) → copy 2 tokens → Recover tab → paste → result must equal the original text.

## Token protocol (compatibility-sensitive)

Format produced by `displayTokens`:

```
[description;YYYY-MM-DD_HH:MM:SS;minShares;share]
```

- Pipeline: text → `LZString.compressToUTF16` → `hexEncode` → `secrets.share(...)` → each share `LZString.compressToBase64` → wrapped in the token. Recovery reverses exactly this. Changing any step breaks previously issued tokens.
- `description` cannot contain spaces or `; [ ]` — these are the parse delimiters (validated in `generate()`).
- `minShares` is capped at 9 because only **one digit** is effectively read back: `tokenMinShares()` takes the single character before the final `;`. A multi-digit minimum would silently corrupt recovery.
- `numShares` max 255 (`secrets.getConfig().maxShares`).
- Recovery requires all tokens to share the same `description`+timestamp prefix (enforced in `recoverText`); tokens from different generations are rejected.

## Conventions & quirks

- Generation is intentionally staged with `setTimeout(..., 500)` between `generate` → `generateShares` → `validateCombinations` → `displayTokens` so the UI can show status. After generating, the app auto-validates up to 50 token combinations (`MAX_VALIDATIONS`): `minShares` tokens must recover the text, `minShares − 1` must not. Keep this passing when touching the pipeline.
- App state is held in script-scope globals (`original`, `minShares`, `sharesBase64`, `lastShareLines`, …) shared by the generate flow; recovery re-reads everything from the DOM instead.
- **Rule: all variable names and user-facing messages must be in English**, including error/status text set via `setStatus(...)`. Some existing source comments are Portuguese (`Fonte:`) — don't propagate that to new code. Commit messages are short and informal (e.g. `dark background on qr code dialog`); default branch is `main`.
