# release-binaries

This is the public home for prebuilt binaries of partrocks tools whose source lives in private
repos. Each tool's CI publishes its builds here when it releases a new version. Anything that
needs a tool downloads it from here, with no GitHub sign-in. The repo holds no source code, only
this README and the releases.

## Tools

| Tool | Prefix | Source repo | Targets |
|------|--------|-------------|---------|
| CrossFade | `xf` | `partrocks/crossfade` (private) | `aarch64-apple-darwin`, `x86_64-apple-darwin`, `x86_64-pc-windows-msvc`, `x86_64-unknown-linux-musl` |

Each tool owns its prefix. A prefix is short, lower case and unique in this table, and is
usually the binary's name.

## How it works

### Only new versions publish

A tool publishes here only when it tags a new version. Ordinary pushes and pull requests never
publish anything, and there are no nightly or edge builds. Each new version adds that version's
release and moves the tool's latest release, and nothing else ever changes here.

### Releases

Every tool has two kinds of release here, both named by its prefix:

- **`<prefix>-v<version>`**, for example `xf-v0.2.0`. There is one per version. It is created
  once and never changed, and its notes are that version's changelog section.
- **`<prefix>-latest`**, for example `xf-latest`. There is one per tool, and it moves. Each new
  version replaces its assets and retitles it `<Tool> latest (v<version>)`, with the same notes.

Every release is published with `--latest=false`. GitHub's own "latest release" covers the whole
repo, so it would point at whichever tool released last. `<prefix>-latest` is the latest release
of one tool. Releases are never drafts or prereleases.

All tags point at this repo's default branch, since there's no source to tag.

### Assets

Asset names carry the prefix and the target but never the version, so a link changes between
versions only by its tag:

- `<prefix>-<target>.tar.gz` for macOS and Linux, holding the binary and `LICENSE`.
- `<prefix>-<target>.zip` for Windows, holding `<binary>.exe` and `LICENSE`.
- `<prefix>-<target>.<ext>.sha256` next to each archive. It holds one line in `sha256sum`
  format (`<hash>  <archive name>`), so a download fetches just its own pair.

`<target>` is the Rust target triple, or the equivalent `<os>-<arch>` for a tool not built
with Rust.

### Download URLs

```
https://github.com/partrocks/release-binaries/releases/download/<prefix>-latest/<prefix>-<target>.<ext>
https://github.com/partrocks/release-binaries/releases/download/<prefix>-v<version>/<prefix>-<target>.<ext>
```

For example, the latest CrossFade for Apple silicon is always
`https://github.com/partrocks/release-binaries/releases/download/xf-latest/xf-aarch64-apple-darwin.tar.gz`.

## Fetching a tool

A consumer picks the target for its host and downloads the archive and its `.sha256` into a
temporary folder. It checks the checksum, and only then unpacks the archive where the tool
lives. Here is a sketch for macOS and Linux:

```sh
prefix=xf
if [ -n "$VERSION" ]; then tag="$prefix-v$VERSION"; else tag="$prefix-latest"; fi
case "$(uname -s)-$(uname -m)" in
  Darwin-arm64)  target=aarch64-apple-darwin ;;
  Darwin-x86_64) target=x86_64-apple-darwin ;;
  Linux-x86_64)  target=x86_64-unknown-linux-musl ;;
  *) echo "No $prefix build for $(uname -s) $(uname -m)" >&2; exit 1 ;;
esac
asset="$prefix-$target.tar.gz"
base="https://github.com/partrocks/release-binaries/releases/download/$tag"
tmp=$(mktemp -d)
curl -fsSL -o "$tmp/$asset" "$base/$asset"
curl -fsSL -o "$tmp/$asset.sha256" "$base/$asset.sha256"
if command -v sha256sum >/dev/null; then check="sha256sum -c"; else check="shasum -a 256 -c"; fi
(cd "$tmp" && $check "$asset.sha256") || exit 1
tar -xzf "$tmp/$asset" -C "$tmp"
mkdir -p bin && mv "$tmp/$prefix" bin/
```

- Never install an archive that fails its checksum. A download made while `<prefix>-latest`
  is being replaced can fail the check, so try again a minute later.
- On Windows under Git Bash or MSYS, `uname -s` starts with `MINGW` or `MSYS`. Fetch the `.zip`
  there and unzip it. Windows can't overwrite a running `.exe` but can rename one, so move the
  old binary aside before putting the new one in place.
- Pin a version when a setup must be reproducible, and use `-latest` to stay current.

CrossFade's consumer is fade-platform's `make crossfade` target, which fetches `xf` this way into
a git-ignored `bin/`.

## Adding a tool

1. **Pick a prefix.** Add a row to the [Tools](#tools) table in a PR to this repo.
2. **Create a token.** Make a fine-grained personal access token with these settings:
   - Resource owner `partrocks`.
   - Repository access limited to `partrocks/release-binaries`.
   - The **Contents: Read and write** permission only.
3. **Add the token as a secret.** On the tool's source repo, add it as an Actions secret named
   `RELEASES_TOKEN`, under Settings, Secrets and variables, Actions. Note the token's expiry
   date somewhere you'll see it, because a release fails once the token lapses.
4. **Build in CI on a new version only.** Run the build from the job that creates the tool's
   new version tag. For CrossFade, that's the job that tags `xf-v<version>` after a version
   bump lands on main. Don't publish on any other trigger. Build each target, pack it as
   described above with its `.sha256`, and upload the files as workflow artifacts. Publish in
   one job after every build has passed, so a release never has some targets and not others.
5. **Publish** with the GitHub CLI and the token:

   ```sh
   export GH_TOKEN="$RELEASES_TOKEN"
   repo=partrocks/release-binaries
   # The versioned release is created once. A re-run skips it.
   gh release view "$prefix-v$version" --repo "$repo" >/dev/null 2>&1 ||
     gh release create "$prefix-v$version" dist/* --repo "$repo" --target main \
       --title "$Tool v$version" --notes-file notes.md --latest=false
   # The latest release is created the first time, then its assets and notes are replaced.
   if gh release view "$prefix-latest" --repo "$repo" >/dev/null 2>&1; then
     gh release upload "$prefix-latest" dist/* --repo "$repo" --clobber
     gh release edit "$prefix-latest" --repo "$repo" \
       --title "$Tool latest (v$version)" --notes-file notes.md --latest=false
   else
     gh release create "$prefix-latest" dist/* --repo "$repo" --target main \
       --title "$Tool latest (v$version)" --notes-file notes.md --latest=false
   fi
   ```

   If a release fails, re-run it only for the newest version. Re-running an older one would
   move `<prefix>-latest` back to that older version.
6. **Check the first release.** Fetch it on each OS with the URLs above.

Never publish a release here by hand, and never reuse another tool's prefix.
