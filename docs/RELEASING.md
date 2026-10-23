# Releasing OpenRSCAD

One tag ships everything: the desktop installers (`desktop/`, Tauri v2 — native
builds for macOS, Windows, and Linux that update themselves in place) and the
`openrscad-engine` npm package (the wasm build of `crates/openrscad-wasm`).

Releases are cut by **[changesets](https://github.com/changesets/changesets)**,
not by tagging by hand.

## How auto-update works

1. On launch (and from **OpenRSCAD → Check for Updates…**) the app fetches the
   release manifest:
   `https://github.com/matthova/openrscad/releases/latest/download/latest.json`
2. If the manifest version is newer than the running app, it downloads that
   platform's update artifact, verifies its **minisign signature** against the
   public key baked into `tauri.conf.json`, installs it, and relaunches.
3. `releases/latest` resolves only to the GitHub Release marked **Latest**,
   which is never a draft or prerelease, so neither is visible to the
   updater. A new release isn't marked Latest until all of its installers have
   uploaded (see below).

Update artifacts per platform:

| OS      | First install     | Self-updates?                        |
| ------- | ----------------- | ------------------------------------ |
| macOS   | `.dmg`            | ✅ (per-arch `.app.tar.gz`)           |
| Windows | NSIS `.exe`       | ✅ (re-runs the signed installer)     |
| Linux   | `.AppImage`       | ✅ AppImage only                      |
| Linux   | `.deb` / `.rpm`   | ❌ update via the package manager     |

## Cutting a release

1. **Add a changeset with each user-facing change.** In the PR, run
   `npx changeset`, pick the bump, write a one-line summary, and commit the
   `.changeset/*.md`. Pure `docs`/`ci`/`test`/`refactor` PRs need none. See
   `.changeset/README.md` and [CONTRIBUTING.md](../CONTRIBUTING.md).
2. **The `changesets` workflow keeps a "Version Packages" PR open**, updating it
   as changesets land on `main`. It runs `npm run version`
   (`changeset version` + `scripts/sync-versions.mjs`), which consumes the
   `.changeset/*.md` files, bumps every version-bearing file, and writes
   `CHANGELOG.md`. Review that diff on the PR.
   - CI does not start on that PR by itself — it is authored by `GITHUB_TOKEN`,
     so it lands in the approval-required state. Click **Approve workflows to
     run** if you want the checks.
3. **Merge the Version Packages PR** with any merge style. Unlike release-please
   there is no PR association to strip, so **Squash**, **Merge commit**, and
   **Rebase and merge** are all safe — the release keys on the committed version,
   not the PR.
4. Merging lands the bumped version on `main`. `changesets.yml` sees the tree's
   version has no matching `vX.Y.Z` tag, creates the tag + GitHub Release (notes
   from `CHANGELOG.md`), then dispatches:
   - **Release desktop app** — 4-OS installers + `latest.json`, uploaded onto
     that Release.
   - **Publish engine to npm** — `openrscad-engine`, via OIDC with provenance.

   The Release is created **not** marked Latest. Once every platform's
   installers are attached (about 20 minutes), the desktop workflow's `promote`
   job marks it Latest. Only then do the download links
   (`releases/latest/download/<name>`) and the updater's `latest.json` switch to
   it. Until then they keep serving the previous release, so they never 404. If
   any platform leg fails, the previous release stays Latest. Fix the problem
   and re-run the failed jobs, which re-runs `promote` too.

### Versions are owned by changesets

The root `package.json` version is the canonical source of truth.
`changeset version` bumps it and writes `CHANGELOG.md`; then
`scripts/sync-versions.mjs` propagates that version into `Cargo.toml`,
`Cargo.lock`, `desktop/src-tauri/{Cargo.toml,Cargo.lock,tauri.conf.json}`,
`desktop/{package.json,package-lock.json}`, and
`packages/npm/{package.json,package-lock.json}`.

**Never hand-edit those version fields.** CI no longer stamps versions — it
asserts them, so a tag whose commit carries a different version fails the build
rather than shipping a mismatched installer or a package whose `version()`
disagrees with its own manifest.

Deliberately *not* on the shared version: `web/package.json` (private
playground, never published), `editors/vscode/package.json` (ships to the VS
Code marketplace on its own cadence), and `fuzz/Cargo.toml` (`publish = false`).

### Version bumps

The repo shares one version; you pick the bump in each changeset. While we're
0.x:

| changeset | bump | example |
| --- | --- | --- |
| `patch` | patch | 0.2.0 → 0.2.1 |
| `minor` | minor | 0.2.0 → 0.3.0 |
| breaking change | `minor` (not `major`) | 0.2.0 → 0.3.0 |

Do not select `major` before we deliberately cut 1.0. To land a specific bump,
add (or edit) a changeset with that level; the highest pending bump wins.

### Escape hatches

- Re-run a build for an existing tag:
  `gh workflow run release.yml --ref vX.Y.Z -f tag=vX.Y.Z`
  (same for `publish-npm.yml`).
- Publishing a Release **by hand** still fires both workflows via
  `release: published`. Bot-authored releases are ignored on that trigger so the
  dispatch path can't double-fire. Untick **Set as the latest release** when
  you publish. `promote` marks it Latest after the build, and leaving the box
  ticked 404s every download link until then.

---

## ✅ Human-only checklist

These require secrets, paid accounts, or GitHub UI actions an agent can't do.

### One-time — required for the updater to work at all

- [ ] **Generate the updater signing keypair** (needs the Tauri CLI locally):
      ```sh
      cd desktop && npx tauri signer generate -w ~/.openrscad-updater.key
      ```
      This prints a **public key** and writes a password-protected **private
      key**. Keep the private key and password secret; they never enter the repo.
- [ ] **Paste the public key** into `desktop/src-tauri/tauri.conf.json` at
      `plugins.updater.pubkey`, replacing `REPLACE_WITH_UPDATER_PUBLIC_KEY`.
      Commit this. (The app will not build a release bundle until this is a real
      key.)
- [ ] **Add repository secrets** (GitHub → Settings → Secrets and variables →
      Actions):
  - [ ] `TAURI_SIGNING_PRIVATE_KEY` — contents of `~/.openrscad-updater.key`
  - [ ] `TAURI_SIGNING_PRIVATE_KEY_PASSWORD` — the password you chose
- [ ] **Confirm GitHub Actions can create releases**: Settings → Actions →
      General → Workflow permissions → **Read and write permissions**. (The
      workflow also requests `contents: write`.)
- [ ] **Allow Actions to open PRs**: Settings → Actions → General → **Allow
      GitHub Actions to create and approve pull requests**. Without it the
      changesets action cannot open its Version Packages PR.
- [ ] **Register the npm trusted publisher** (one-time; `openrscad-engine@0.0.0`
      already exists on the registry, so only the registration is left):
      npmjs.com → `openrscad-engine` → Settings → Trusted Publisher → GitHub
      Actions, with Organization or user `matthova`, Repository `openrscad`,
      Workflow filename `publish-npm.yml`, Environment blank. Fields are
      case-sensitive and exact, and npm does not validate them at save time.

### Per release

- [ ] **Merge the "Version Packages" PR** (any merge style). Everything
      downstream is automatic. Merge only when ready to ship: existing desktop
      users are offered the update once every installer has uploaded and the
      release is marked Latest.

### OS code-signing

What ships: the macOS `.app` is signed with the project's **Developer ID** and
**notarized** by Apple in CI, with the ticket stapled, so a downloaded copy
opens with no Gatekeeper warning. The release workflow fails a macOS leg whose
`.app` isn't Developer ID signed or has no stapled ticket. (The bundler only
*warns* when notarization credentials are missing, so that check is the only
thing that catches it.) Local builds fall back to the ad-hoc identity `"-"`
(`bundle.macOS.signingIdentity` in `desktop/src-tauri/tauri.conf.json`); CI's
`APPLE_SIGNING_IDENTITY` overrides it. Windows still shows a SmartScreen warning
on first install. Auto-update works either way.

- [x] **macOS**: Apple Developer Program membership; a "Developer ID
      Application" certificate (created by the Account Holder in Xcode →
      Settings → Accounts → Manage Certificates, exported from Keychain Access as
      a `.p12`); repository secrets:
  - `APPLE_CERTIFICATE`: `base64 -i cert.p12`
  - `APPLE_CERTIFICATE_PASSWORD`: the `.p12` export password
  - `APPLE_SIGNING_IDENTITY`: `Developer ID Application: <Name> (<TEAMID>)`,
    as printed by `security find-identity -v -p codesigning` (no quotes)
  - `APPLE_ID`: the Apple ID email used for notarization
  - `APPLE_PASSWORD`: an **app-specific password** for that Apple ID
    (account.apple.com → Sign-In and Security → App-Specific Passwords)
  - `APPLE_TEAM_ID`: the 10-character team ID

  Keep these working:
  - Changing or resetting that Apple ID's password **revokes every
    app-specific password**. Generate a new one and update `APPLE_PASSWORD`, or
    the next release's macOS legs fail at the notarization check.
  - The certificate expires. Apple shows the date under Certificates, IDs &
    Profiles. Before then, create a new one and replace `APPLE_CERTIFICATE`,
    `APPLE_CERTIFICATE_PASSWORD`, and `APPLE_SIGNING_IDENTITY`. Builds
    already shipped stay valid.
  - Back up the `.p12` and its password somewhere safe. Apple can't
    re-download the private key.
- [ ] **Windows**: obtain a code-signing certificate (Azure Trusted Signing is
      the cheapest modern option; an OV/EV cert also works) and wire it into the
      Windows bundle config / workflow.
- [ ] **Linux**: no signing required.

### Optional / nice-to-have

- [x] Decide whether Linux users should be steered to the **AppImage** (the only
      self-updating Linux format) vs. `.deb`/`.rpm` in the README/download page.
      The README download table links the AppImage; `.deb`/`.rpm` stay reachable
      via the "Browse all downloads" release-page link.
- [x] Add a download/landing page linking to the latest release assets — the
      README's **Download** table links each platform via stable, version-less
      asset aliases (`OpenRSCAD-<platform>...`) uploaded by the release workflow, so
      `releases/latest/download/<name>` always resolves to the newest build.
- [ ] Add an in-app download/callout in the **web playground** (`web/src/App.tsx`,
      gated to the non-Tauri build) linking to the desktop downloads.
