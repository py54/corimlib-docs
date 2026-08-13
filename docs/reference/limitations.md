# Limitations & Experimental APIs

CorimLib was extracted out of an existing mod (Crunch) mid-project, rather than designed from scratch as a public library. That history shows up in a few real, structural ways — this page lists them plainly so you can design around them instead of discovering them at runtime.

## Single global `PlatformBridge` singleton

`Corim.bridge()` resolves via `ServiceLoader.load(PlatformBridge.class, ...).findFirst()` and caches the result forever. It has no concept of "which mod is asking." If your mod ships its own `PlatformBridge` implementation *and* another CorimLib-dependent mod (e.g. Crunch itself) is also installed, both implementations are on the same classpath under the same service registration file (`META-INF/services/dev.py54.crunch.corimlib.PlatformBridge`), and `ServiceLoader` resolution order across a combined classpath is not something either mod controls. In practice, whichever implementation is found first "wins," and **every** `Corim.bridge()` call from **any** CorimLib-dependent mod present routes through that one instance.

This has not been a real-world problem yet because, as of this writing, Crunch is the only mod known to depend on CorimLib and ship a `PlatformBridge` implementation. It becomes a real problem the moment two independent mods both do. There is currently no per-mod scoping mechanism — this is an accurate limitation of the design, not a bug in any one implementation.

## Global `FeatureRegistry`

`FeatureRegistry` is a single static map keyed by feature id, shared by every mod that calls `FeatureRegistry.register(...)` in the same game process. There's no per-mod namespace enforced — `register()` only throws on an exact id collision. Two independent mods both depending on CorimLib will have their features intermixed in the same `FeatureRegistry.all()` list (and therefore the same `CrunchMainScreen` card grid, if that's what's opened). Prefix your feature ids defensively (Crunch's own convention is `<category>.<name>`, e.g. `hud.fps`) — but this is a convention, not something the framework enforces for you.

## `CrunchTheme` and the custom widgets are not public

`CrunchTheme`, `CrunchButtonWidget`, and `CrunchSliderWidget` — the classes that actually implement the dark chamfered-corner visual style — are declared package-private (`final class`, no `public` modifier). They compile fine inside CorimLib's own `dev.py54.crunch.corimlib.gui` package and are completely inaccessible from any other package, including your mod's. See [GUI & Theming](../gui/gui-and-theming.md) for the full breakdown of what is and isn't actually reusable in the GUI layer. The five `Screen` classes themselves *are* public and fully usable — it's specifically the low-level drawing primitives that aren't exposed.

## `CorimPaths` is hardcoded to Crunch's own filenames

`CorimPaths.configFile()` always resolves to `crunch.json`, unconditionally — there's no mod-id parameter. If you use `CorimPaths` directly from your own mod, you will read and write the *same file Crunch itself uses*, which is almost certainly not what you want. Build your own path from `Corim.bridge().configDir().resolve("yourmod.json")` instead — see [Config System](../core/config-system.md).

## `Category` is a fixed enum

`Category` (`HUD, VISUAL, TOOLTIPS, CHAT, CONTROLS, ACCESSIBILITY, SCREENSHOTS, PERFORMANCE, INTERFACE, GENERAL`) cannot be extended by a depending mod. If none of the ten existing categories fit, your closest option is `GENERAL`.

## No public Maven repository yet

CorimLib's build only publishes to `mavenLocal()` today. There is no Modrinth Maven, GitHub Packages, or Maven Central listing. See [Gradle Setup](../getting-started/gradle-setup.md) for the practical workaround (publish it yourself from source, or use a `flatDir` repository pointing at the distributable jars).

## The main CorimLib source repository is private

The repository this documentation describes is not itself public. This documentation site is intentionally the *only* public artifact — it describes real classes and real method signatures verified against the actual source, without reproducing that source wholesale.

## Forge's `runClient` crash (unresolved)

`:forge:runClient` on the consuming mod (Crunch) currently crashes with `java.lang.module.ResolutionException: Modules crunch and main export package ... to module ...` — a JPMS split-package conflict in ForgeGradle's dev-mode module-layer construction. Root cause has not been found; this affects only local `runClient` testing, and Forge's *build* output (correct compile, correct `mods.toml` metadata) is otherwise verified. Treat Forge as build-verified-only, higher-risk than Fabric or NeoForge, until this is actually root-caused. See [Forge Setup](../getting-started/forge-setup.md).

## `Item.TooltipContext` is `null` on Forge

Forge's real `ItemTooltipEvent` signature has no context parameter, so the real `ForgePlatformBridge` implementation always passes `null` for it into `TooltipHandler.onTooltip`. Code that reads `context` needs a null guard if it needs to work on Forge — see [Events, Keybinds, Tooltips & Chat](../events/events-keybinds-tooltips-chat.md).

## No `1.21.x` support

Only the `26.1.x`/`26.2` Minecraft line is supported. `1.21.x` is a genuinely different, older Minecraft API generation, not a drop-in — see [Supported Versions & Loaders](supported-versions.md).

## Config loading must be deferred past mod-init

`ConfigManager.load(...)` fires every `Feature.onToggle`/`Setting.onChange` handler synchronously, and real handlers touch `Minecraft.getInstance().options`, which doesn't exist at plain mod-init time. This isn't a design flaw so much as a footgun worth flagging explicitly: always call `ConfigManager.load(...)` from inside `Corim.bridge().onClientStarted(...)`, never earlier. See [Config System](../core/config-system.md).
