# Supported Versions & Loaders

## Compatibility matrix

This reflects what has actually been built, not what's theoretically possible. "Build-verified" means the module compiles and its loader metadata (`fabric.mod.json` / `neoforge.mods.toml` / `mods.toml`) was manually checked for the correct Minecraft version range. "`runClient`-verified" means the loader was actually launched and reached a loaded world with no crash.

| Minecraft | Fabric | NeoForge | Forge |
|---|---|---|---|
| 26.2 | Build + `runClient` verified | Build + `runClient` verified | Builds; `runClient` attempted, crashes with a JPMS module-resolution error (see [Limitations](limitations.md)) |
| 26.1.2 | Build-verified | Build-verified | Build-verified |
| 26.1.1 | Build-verified | **Not published** — no NeoForge build exists upstream for this patch | Build-verified |
| 26.1 | Build-verified | **Not published** — no NeoForge build exists upstream for this patch | **Not published** — no Forge build exists upstream for this patch |

1.21.x has not been attempted at all — it's a genuinely older, different Minecraft API generation (not just a build-property swap the way the 26.1.x line was relative to 26.2), and would need its own real decompilation-based investigation before any claim of support.

## Why the matrix is uneven

Fabric API tags a release for essentially every Minecraft patch version. NeoForge and Forge don't — they only publish a loader build for a given patch when they choose to, and neither published one for bare `26.1`, and NeoForge additionally skipped `26.1.1`. This was verified directly against each loader's real `maven-metadata.xml`, not assumed — there is no NeoForge loader you could install on Minecraft `26.1` or `26.1.1` today, for *any* mod, not just CorimLib.

## What's identical across the whole 26.1.x/26.2 line

Verified by actually decompiling and building against each version's real mapped jar: the entire public API surface described on this site — `PlatformBridge`, `Feature`/`Setting`, `ConfigManager`, the HUD system, all five GUI screens — is identical across `26.1`, `26.1.1`, `26.1.2`, and `26.2`. The only confirmed difference anywhere in `corimlib`'s source is one internal mixin target (`Gui` renamed to `Hud` between `26.1.x` and `26.2`), which is not part of the public API and requires no changes on your end.

## Java & toolchain

- Java 25 (all loaders, all supported Minecraft versions).
- Fabric: Fabric Loader ≥0.19.3, Fabric Loom 1.17-SNAPSHOT.
- NeoForge: `net.neoforged.moddev` plugin, version varies by Minecraft version (`26.2` uses `2.0.143`).
- Forge: ForgeGradle `[7.0.17,8)`.

See the per-loader setup pages ([Fabric](../getting-started/fabric-setup.md), [NeoForge](../getting-started/neoforge-setup.md), [Forge](../getting-started/forge-setup.md)) for exact version numbers and known toolchain pitfalls.
