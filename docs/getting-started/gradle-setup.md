# Gradle Setup

## Coordinates

CorimLib's Gradle group is `dev.py54.corimlib`. Each module publishes a separate artifact:

| Artifact | Module | What it is |
|---|---|---|
| `dev.py54.corimlib:corimlib:1.0.0` | `corimlib/` | Plain classes jar — compile against this. No Minecraft loader API, but does depend on Minecraft itself and `common`. |
| `dev.py54.corimlib:corimlib-fabric:1.0.0` | `fabric/` | The real, deployable Fabric mod jar (jar-in-jar bundles `common` + `corimlib`). |
| `dev.py54.corimlib:corimlib-neoforge:1.0.0` | `neoforge/` | The real, deployable NeoForge mod jar (shadowJar bundles `common` + `corimlib`). |
| `dev.py54.corimlib:corimlib-forge:1.0.0` | `forge/` | The real, deployable Forge mod jar (shadowJar bundles `common` + `corimlib`). |

There's also a `dev.py54.corimlib:common:1.0.0` artifact (the loader-agnostic `Feature`/`Setting`/`ConfigManager` layer with zero Minecraft dependency), but you generally don't need to depend on it directly — `corimlib` already depends on it and re-exposes its classes.

!!! warning "No public Maven repository yet"
    As of today, CorimLib's build only publishes to `mavenLocal()` — there is no Modrinth Maven, GitHub Packages, or Maven Central listing. The coordinates above are real (this is exactly what Crunch itself depends on), but to resolve them in your own build you currently need either:

    - Access to the CorimLib source, so you can run `./gradlew publishToMavenLocal` yourself, or
    - The distributable jars placed in a local flat-dir repository (see below).

    If you only have the jars (not the source), point Gradle at the folder containing them instead of a Maven repository:

    ```groovy
    repositories {
        flatDir {
            dirs "libs" // put corimlib-1.0.0.jar and corimlib-fabric-1.0.0.jar here
        }
    }

    dependencies {
        compileOnly "dev.py54.corimlib:corimlib:1.0.0"
        implementation "dev.py54.corimlib:corimlib-fabric:1.0.0"
    }
    ```

## The compileOnly / implementation split

This is exactly how Crunch's own `fabric/build.gradle` depends on CorimLib — copy this pattern:

```groovy
dependencies {
    minecraft "com.mojang:minecraft:${project.minecraft_version}"
    implementation "net.fabricmc:fabric-loader:${project.loader_version}"
    implementation "net.fabricmc.fabric-api:fabric-api:${project.fabric_api_version}"

    // compileOnly against the plain classes jar (what your source actually compiles
    // against); implementation against the real deployable corimlib-fabric.jar for
    // runClient/runtime - its classes are only reachable there via Fabric Loader's
    // jar-in-jar unpacking, not as loose classpath entries, so both are needed.
    compileOnly "dev.py54.corimlib:corimlib:${project.corimlib_version}"
    implementation "dev.py54.corimlib:corimlib-fabric:${project.corimlib_version}"
}
```

The same split applies on NeoForge and Forge, just swapping `corimlib-fabric` for `corimlib-neoforge` / `corimlib-forge`:

```groovy
dependencies {
    compileOnly "dev.py54.corimlib:corimlib:${project.corimlib_version}"
    implementation "dev.py54.corimlib:corimlib-neoforge:${project.corimlib_version}"
}
```

**Why two dependencies instead of one:** the plain `corimlib` artifact is what your code actually compiles against. The `-fabric`/`-neoforge`/`-forge` artifact is a *self-contained mod jar* — its `common`/`corimlib` classes are only reachable at runtime through the loader's own jar-in-jar unpacking mechanism, not as loose classpath entries. You need `implementation` on the real mod jar so `runClient` and your final build actually have CorimLib present at runtime, and `compileOnly` on the plain jar so you're not trying to compile against a bundled/shaded jar directly.

## Minecraft version

CorimLib currently builds against Minecraft `26.2`, `26.1.2`, `26.1.1`, and `26.1` (Fabric only for the last one — see [Supported Versions & Loaders](../reference/supported-versions.md)). Set `minecraft_version` in your `gradle.properties` to match whichever CorimLib build you're depending on; the API surface is identical across the whole 26.1.x/26.2 line except one internal mixin target (not part of the public API), so your own code doesn't need version-specific branches for this reason alone.

Next: pick your loader — [Fabric](fabric-setup.md), [NeoForge](neoforge-setup.md), or [Forge](forge-setup.md).
