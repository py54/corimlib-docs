# Installation

CorimLib has two audiences, and "installing" means something different for each.

## As a player

If you're installing a mod that *depends on* CorimLib, CorimLib is a required **companion mod** — it is not bundled into the depending mod's jar. Both jars go in your `mods/` folder together:

```
mods/
├── mymod-1.0.0-26.2-fabric.jar
└── corimlib-fabric-1.0.1.jar
```

The dependent mod's loader metadata declares CorimLib as a hard dependency (e.g. Fabric's `fabric.mod.json` has `"corimlib": ">=1.0.0"` under `depends`), so Fabric Loader / NeoForge / Forge will refuse to load without it, rather than silently crashing later.

!!! warning "No public distribution channel yet"
    CorimLib's source repository is currently **private**, and it has not yet been published to a public mod distribution site (Modrinth, CurseForge) or a public Maven repository. If you've received a CorimLib jar directly from the developer, install it as shown above. There is no public download link to share here yet — see [Limitations](../reference/limitations.md).

## As a mod developer

To build against CorimLib, you need:

1. Java 25.
2. The CorimLib Gradle artifacts (`corimlib`, and one of `corimlib-fabric` / `corimlib-neoforge` / `corimlib-forge`) resolvable from a Maven repository your build can reach — see [Gradle Setup](gradle-setup.md) for exactly how CorimLib itself publishes these (currently `mavenLocal()` only).
3. A loader-specific setup for whichever of Fabric, NeoForge, or Forge you're targeting — see the loader setup pages.

Once the dependency is resolvable, continue to [Gradle Setup](gradle-setup.md).
