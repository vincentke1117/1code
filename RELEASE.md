# Desktop App Release Guide

## Prerequisites

1. **Apple Developer Credentials** (for code signing and notarization):
   - Developer ID certificate installed in Keychain
   - Notarization keychain profile: `21st-notarize`
   - To create profile: `xcrun notarytool store-credentials "21st-notarize" --apple-id YOUR_APPLE_ID --team-id YOUR_TEAM_ID`

2. **Cloudflare R2 Access**:
   - `wrangler` CLI authenticated (`npx wrangler login`)
   - Access to `components-code` bucket

3. **GitHub CLI**:
   - `gh` CLI authenticated (`gh auth login`)
   - Push access to `21st-dev/21st` repo

## Quick Release (Recommended)

### 1. Bump Version

```bash
cd apps/desktop
npm version patch --no-git-tag-version  # 0.0.5 → 0.0.6
```

### 2. Commit Version Bump

```bash
git add apps/desktop/package.json
git commit -m "chore(desktop): bump version to 0.0.X"
git push
```

### 3. Run Release Script

```bash
cd apps/desktop
bun run release
```

This script:
1. Cleans `release/` folder
2. Downloads Claude Code binary from Anthropic CDN
3. Builds the app (main, preload, renderer)
4. Packages for macOS (ARM64 + x64)
5. Generates update manifests (`latest-mac.yml`, `latest-mac-x64.yml`)
6. Submits DMGs for Apple notarization (async, doesn't wait)
7. Uploads all files to R2 CDN
8. Creates GitHub release with assets

### 4. Wait for Notarization & Staple

Check notarization status:
```bash
xcrun notarytool history --keychain-profile "21st-notarize"
```

Once both DMGs show `status: Accepted`, staple and re-upload:

```bash
cd apps/desktop/release

# Staple notarization tickets
xcrun stapler staple 1Code-0.0.X-arm64.dmg
xcrun stapler staple 1Code-0.0.X.dmg

# Re-upload stapled DMGs to R2
npx wrangler r2 object put "components-code/releases/desktop/1Code-0.0.X-arm64.dmg" \
  --file="1Code-0.0.X-arm64.dmg" --content-type="application/x-apple-diskimage"
npx wrangler r2 object put "components-code/releases/desktop/1Code-0.0.X.dmg" \
  --file="1Code-0.0.X.dmg" --content-type="application/x-apple-diskimage"

# Update GitHub release assets
gh release delete-asset v0.0.X 1Code-0.0.X-arm64.dmg --yes
gh release delete-asset v0.0.X 1Code-0.0.X.dmg --yes
gh release upload v0.0.X 1Code-0.0.X-arm64.dmg 1Code-0.0.X.dmg --clobber
```

### 5. Publish Release & Update Changelog

**Important:** The release is created as a draft. You must publish it:

```bash
# Publish from draft and set as latest + add changelog
gh release edit v0.0.X --draft=false --latest --notes "$(cat <<'EOF'
## What's New in v0.0.X

### Features

- Feature 1
- Feature 2

### Improvements & Fixes

- Fix 1
- Fix 2
EOF
)"
```

Or separately:
```bash
# Just publish from draft
gh release edit v0.0.X --draft=false --latest

# Then update changelog
gh release edit v0.0.X --notes "..."
```

### 6. Sync to Public Repository

**Important:** After releasing, sync to the open-source repo:

```bash
cd apps/desktop
./scripts/sync-to-public.sh
```

This script:
1. Syncs code from private repo to public `21st-dev/1code` repo
2. Creates a matching GitHub release in the public repo
3. Uses release notes from the private repo

### 7. Update Download Links

Update version in `apps/web/lib/desktop.ts`:

```typescript
export const DESKTOP_VERSION = "0.0.X"
```

### 8. Commit & Tag

```bash
git add -A
git commit -m "feat(desktop): release v0.0.X with <summary>"
git push

# Fix tag to point to release commit
git fetch --tags
git tag -d v0.0.X
git tag v0.0.X
git push origin :refs/tags/v0.0.X
git push origin v0.0.X
```

## File Locations

### Build Outputs (`apps/desktop/release/`)

| File | Purpose |
|------|---------|
| `1Code-X.X.X-arm64.dmg` | ARM64 installer (Apple Silicon) |
| `1Code-X.X.X.dmg` | x64 installer (Intel) |
| `1Code-X.X.X-arm64-mac.zip` | ARM64 auto-update package |
| `1Code-X.X.X-mac.zip` | x64 auto-update package |
| `latest-mac.yml` | ARM64 update manifest |
| `latest-mac-x64.yml` | x64 update manifest |
| `*.blockmap` | Delta update blockmaps |

### CDN URLs

- Manifests: `https://cdn.21st.dev/releases/desktop/latest-mac.yml`
- DMGs: `https://cdn.21st.dev/releases/desktop/1Code-X.X.X-arm64.dmg`
- ZIPs: `https://cdn.21st.dev/releases/desktop/1Code-X.X.X-arm64-mac.zip`

### GitHub Releases

- Private: `https://github.com/21st-dev/21st/releases/tag/vX.X.X`
- Public: `https://github.com/21st-dev/1code/releases/tag/vX.X.X`

## How Auto-Updates Work

1. App checks `https://cdn.21st.dev/releases/desktop/latest-mac.yml` on startup and window focus
2. If version in manifest > current version, shows "Update Available" banner
3. User clicks Update → downloads ZIP in background
4. After download, app restarts and installs update

## Troubleshooting

### Notarization Fails

Check detailed log:
```bash
xcrun notarytool log <submission-id> --keychain-profile "21st-notarize"
```

Common issues:
- Unsigned binaries inside the app
- Hardened runtime not enabled
- Missing entitlements

### Claude Binary Missing

If x64 build shows "file source doesn't exist" for `darwin-x64`:

```bash
# Download all platform binaries
bun run claude:download:all
```

Note: Windows binary may 404 (not available from Anthropic).

### Old Files in Release

The `release` script starts with `rm -rf release` to clean old files. If you see old version files, ensure the script ran correctly.

### GitHub Release Shows as Draft

**This is expected!** The script creates releases as drafts. After stapling and uploading, publish it:
```bash
gh release edit v0.0.X --draft=false --latest
```

You can combine with changelog update (see Step 5).

## Scripts Reference

| Script | Description |
|--------|-------------|
| `bun run release` | Full release pipeline |
| `bun run build` | Build app only |
| `bun run package:mac` | Package for macOS |
| `bun run dist:manifest` | Generate update manifests |
| `bun run claude:download` | Download Claude binary (current arch) |
| `bun run claude:download:all` | Download Claude binary (all platforms) |
