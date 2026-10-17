# Test the browser clipboard fix

This fork carries the complete shortcut-focus correction from upstream PR
[#19452](https://github.com/pingdotgg/t3code/pull/19452), based on official Nightly
`v0.1.1-nightly.20261016.3218`. Its PR targets a matching baseline branch rather
than the fork's older `main`. It does not update `main` or replace an installed app.

## What changes

The desktop checks the receiving page's focus and focused frame before enabling
native editing or forwarding an application shortcut. It no longer identifies the
page by Electron's global focused-webContents lookup, which can name a different
open tab. The focused-frame check keeps background agent input blocked.

Unit tests cover the older-tab failure and background-input protection. They do
not establish that physical macOS shortcuts work. Repeat the supervised test
below before using the patched version for normal work.

## Run the development app without replacing Nightly

This is the supported way to test the patch with a separate Electron profile.
Keep the official Nightly installed. Close any other **T3 development** app first:
development builds share `t3code-dev` (or the older `T3 Code (Dev)`) Electron data.
They do not use Nightly's `t3code-v2` Electron profile.

Use Node 24 satisfying the repository's `engines.node` requirement and the pinned
pnpm version in `package.json`. On this Mac, Node 24 is installed under
`/opt/homebrew/opt/node@24/bin`.

```sh
cd "$HOME/Developer/t3code/.claude/worktrees/browser-clipboard-focus"
export PATH="/opt/homebrew/opt/node@24/bin:$PATH"
pnpm install --frozen-lockfile
```

For a fresh checkout with no `.env`, copy `.env.example` to `.env`. It contains
public configuration. Do not link or copy a real installation's settings,
credentials, database or browser profiles into this worktree.

When ready to launch the test app at the Mac:

```sh
pnpm exec vp run dev:desktop --home-dir "$PWD/.t3/clipboard-preview"
```

The command builds and runs the development desktop and its development services.
Its server data lives under this worktree's ignored `.t3/clipboard-preview`, and the
development Electron profile is separate from the official packaged app. Normal
development mode disables automatic app updates. The window is a development
app, not an installation of the official Nightly.

The test state can be empty and will not contain Nightly's saved conversations or
site logins. Do not sign in or import cookies for this clipboard check. Keep this
Terminal open while testing. Stop it with Control-C in that Terminal, close the
development window, then open `/Applications/T3 Code (Nightly).app` normally.
Nothing needs to be copied back to Nightly.

## Packaged macOS build: preparation and limitations

A local unsigned Apple-silicon DMG can also be prepared. This command **builds**
an artifact; it does not install or launch it:

```sh
pnpm exec node scripts/update-release-package-versions.ts 0.1.1-preview.20261016.3218
env T3CODE_DESKTOP_SIGNED=false T3CODE_DESKTOP_MOCK_UPDATES=false \
  T3CODE_DESKTOP_SKIP_BUILD=false \
  pnpm exec node scripts/build-desktop-artifact.ts \
  --platform mac --target dmg --arch arm64 \
  --build-version 0.1.1-preview.20261016.3218 \
  --output-dir "$HOME/.codex/scratch/t3-browser-login-review-20261016/preview-artifacts"
```

The preview version omits the official update feed. Build signing and mock updates
must remain disabled; do not supply release credentials or publish the artifact.
The existing builder uses `--publish never`.
The version-alignment command changes four package manifests in the build
checkout. These are local build inputs, not additional changes to include in the
clipboard PR; restore only those generated version changes after building.

**A custom version, a renamed app, and `T3CODE_HOME` do not isolate a packaged
app's Electron profiles.** Packaged macOS builds use
`~/Library/Application Support/t3code-v2`; `T3CODE_HOME` only separates server data.
They can also acquire the same single-instance lock. Do not launch this packaged
preview in the daily account expecting an isolated test.

For an installed-app test without sharing the daily account's profiles, use a
separate macOS test account. After obtaining approval to install and run it:

1. Make the DMG available to that account, open it there and copy its app to that
   account's `~/Applications`. Keep the official Nightly in `/Applications`.
2. Open the copied app from that account. It is an unsigned local build; macOS
   may require approval in System Settings > Privacy & Security. Do not disable
   Gatekeeper globally or remove protections from the daily app.
3. Run only the harmless clipboard test. Use the test account's own empty T3
   state; do not copy daily data or sign in to sites.
4. Quit the test app, return to the daily macOS account and open the original
   Nightly. The test app and test-account data can remain unused or be removed
   later; do not delete the daily T3 data directory.

An installed preview that runs alongside Nightly in the **same account** needs
additional app-identity and Electron-data isolation work. That is outside this
clipboard-only patch. Use the development app above for the lighter test.

## Supervised physical-key check

This replaces clipboard contents with known harmless text. Have the person at
the Mac perform the keys; automation of a copy command is a different path.

1. Open two harmless pages with ordinary editable text fields, A then B. Keep
   both tabs open. Use `orchard-614` as source text and `river-927` as a baseline.
2. Right-click Copy of `orchard-614` in B. Click the destination field in B and
   press Command-V. Repeat in A after clicking its destination. Both should paste.
3. In A, right-click Copy of `river-927`. Select all of `orchard-614`, press
   Command-C, then right-click Paste into an empty destination. It should paste
   `orchard-614`. Repeat in B.
4. Repeat copying with ordinary selected page text, then switch through the chat
   composer and back to each page. Check that shortcuts follow the clicked field.
5. Check background agent-input protection with known harmless text and an empty
   composer. Background commands must not paste into the composer or trigger
   an application shortcut there. Also test a popup or frame before relying on it.

Do not read the pre-existing clipboard, use credentials as test text, sign in or
import cookies. Record the result and actual build version. A passing unit suite
or successful package build is not a passing physical-key test.
