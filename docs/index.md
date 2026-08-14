# CorimLib

**CorimLib** is a small, cross-loader Java library for Minecraft client mods. It gives you one `Feature`/`Setting` toggle framework, one JSON config/profile system, one HUD-line system, and a themed GUI — written once, and driven on Fabric, NeoForge, and Forge through a single `PlatformBridge` interface instead of per-loader `if` branches or a bytecode-remapping toolchain like Architectury.

This site documents CorimLib exactly as it exists today, verified directly against its real source — including which parts are genuinely reusable by any mod and which parts are one specific shipped implementation that happens to ship in the same jar. See [Limitations & Experimental APIs](reference/limitations.md) before you build on anything non-obvious.

## Why CorimLib exists

Minecraft 26.2 ships official Mojang mappings with no per-loader intermediary layer, so the same compiled class shapes (`Screen`, `GuiGraphicsExtractor`, `KeyMapping`, …) are valid from any loader. That means shared GUI/HUD/feature code doesn't need `@ExpectPlatform` or a Loom/Architectury remapping pipeline to stay loader-agnostic — it only needs a handful of registration calls (keybinds, tick events, HUD elements, tooltip/chat hooks) abstracted behind one small interface. CorimLib is that interface, plus everything built on top of it.

```java
// Shared code never imports a loader API directly - it goes through Corim.bridge().
Corim.bridge().registerKeybind(myKeyMapping);
Corim.bridge().onClientTick(mc -> { /* ... */ });
```

## What's in this library

<div class="grid cards" markdown>

- **`PlatformBridge`** — keybind registration, tick/client-started events, HUD element registration, tooltip/chat hooks, and config directory resolution, loaded per-loader via `java.util.ServiceLoader`.
- **`Feature` / `Setting<T>` / `FeatureRegistry`** — a loader-agnostic toggle framework with seven setting types (boolean, int, decimal, enum, color, keybind, text).
- **`ConfigManager` / `ProfileManager`** — Gson-based JSON persistence for the whole `FeatureRegistry`, plus named profile snapshots.
- **`HudLineFeature` / `HudRegistry`** — self-positioning HUD text lines that auto-stack with no gaps, or pin to a dragged position.
- **Five real `Screen` classes** — `CrunchMainScreen`, `CrunchFeatureSettingsScreen`, `CrunchProfilesScreen`, `CrunchHudEditorScreen`, `CrunchPerformanceMonitorScreen` — a complete, working dark-modal-card GUI you can open as-is.

</div>

## Quick example

A minimal feature registration, using only real CorimLib/common classes:

```java
import dev.py54.crunch.feature.Category;
import dev.py54.crunch.feature.Feature;
import dev.py54.crunch.feature.FeatureRegistry;
import dev.py54.crunch.settings.BooleanSetting;

Feature myFeature = new Feature(
        "mymod.example", "Example Feature",
        "Does something useful.", Category.GENERAL, false);

BooleanSetting verbose = new BooleanSetting(
        "verbose", "Verbose Logging", "Log extra detail.", false);
myFeature.addSetting(verbose);

myFeature.onToggle(enabled -> {
    // react to the toggle
});

FeatureRegistry.register(myFeature);
```

Continue to [Installation](getting-started/installation.md) to set up a real project, or jump straight to [Build a Simple Mod](guide/build-a-simple-mod.md) for a full worked example.

## Where to start

| I want to... | Go to |
|---|---|
| Add CorimLib as a Gradle dependency | [Gradle Setup](getting-started/gradle-setup.md) |
| Understand the module layout | [Architecture Overview](core/architecture.md) |
| Implement `PlatformBridge` for my loader | [PlatformBridge API](core/platform-bridge.md) |
| Register a feature toggle | [Features & Settings](core/features-and-settings.md) |
| Add a HUD line | [HUD System](hud/hud-system.md) |
| Open or extend the settings GUI | [GUI & Theming](gui/gui-and-theming.md) |
| See a full working mod | [Build a Simple Mod](guide/build-a-simple-mod.md) |
| Check what's *not* supported | [Limitations & Experimental APIs](reference/limitations.md) |
