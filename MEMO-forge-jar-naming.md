# Memo: Forge release jar filename is now build-specific

**To:** mamo-Connector team
**Date:** 2026-08-17
**Status:** Shipped in `killriam/forge` `replay-Features`, commit `d2bc12ba3dc`.
**Full writeup:** `spec/forge-integration-guide.md` §11 (options considered, rationale).

## What changed

The desktop release jar inside `MaMoForge-portable.zip` used to have a **static** filename:

```
forge-gui-desktop-2.0.14-SNAPSHOT-jar-with-dependencies.jar
```

This never changed between releases (only bumps when `versionCode` changes, which is rare) - the
direct cause of the stale-build confusion we hit earlier. As of this build, the filename carries a
build timestamp, matching what Forge's own startup log already reports as its version:

```
forge-gui-desktop-2.0.14-SNAPSHOT-08.17-0700-jar-with-dependencies.jar
                              ^^^^^^^^^^^^^^^ MM.dd-HHmm, in whatever timezone the build
                                               machine's clock is set to (GitHub Actions runners
                                               default to UTC; don't assume a fixed offset)
```

## What you need to do

**Stop hardcoding the exact jar filename.** After extracting `MaMoForge-portable.zip`, glob for it
instead:

```
forge-gui-desktop-*-jar-with-dependencies.jar
```

This is the same approach Forge's own release workflow already uses internally to find the jar it
just built - it doesn't hardcode the exact name either, for the same reason.

## What did NOT change

- **The download URL is unaffected.** The GitHub Release is still published under the
  `replay-features-latest` tag, force-moved on every push:
  `.../releases/download/replay-features-latest/MaMoForge-portable.zip`. Only the filename
  *inside* the zip changed.
- **The `res/` folder, `CHANGES.txt`, and zip structure are unchanged** - same extraction logic
  otherwise.
- **`java -jar <name> ...` invocation, CLI flags, everything at runtime** - unaffected. Only the
  `<name>` part is now variable.

## If you want to detect staleness without parsing the filename

The jar's manifest already carries the same version string as `Implementation-Version`, readable
at runtime without touching the filename at all (this worked before this change too, and still
does): `forge.util.BuildInfo.getVersionString()` / `.getGitCommit()` in `forge-core`. Useful if you
ever want to compare an installed build against "latest" programmatically rather than by
re-parsing a filename string.

## Questions

Ping the Forge side (this repo's `forge-integration-guide.md` §11 has the full option comparison
and rationale) if the glob approach doesn't fit how mamo-Connector currently locates/installs the
jar - happy to adjust.
