# PlatformBridge API

`PlatformBridge` (package `dev.py54.crunch.corimlib`) is everything CorimLib's shared feature/GUI code needs from a specific loader. Each loader module provides exactly one implementation, registered via `META-INF/services/dev.py54.crunch.corimlib.PlatformBridge`, so feature code in `corimlib` never imports a Fabric/NeoForge/Forge API directly.

## The interface

This is the complete, real interface:

```java
package dev.py54.crunch.corimlib;

import net.minecraft.client.KeyMapping;
import net.minecraft.client.Minecraft;
import net.minecraft.resources.Identifier;

import java.nio.file.Path;
import java.util.function.Consumer;

public interface PlatformBridge {
    String loaderName();

    /** The loader's per-mod config directory, e.g. {@code .minecraft/config/crunch}. */
    Path configDir();

    void registerKeybind(KeyMapping mapping);

    void onClientTick(Consumer<Minecraft> callback);

    /** Fires once, after the client has fully finished starting (Options/etc. exist). */
    void onClientStarted(Consumer<Minecraft> callback);

    void registerHudElement(Identifier id, HudRenderer renderer);

    void onItemTooltip(TooltipHandler handler);

    /** Return false to hide the message. */
    void onChatReceived(ChatFilter filter);

    /** Runs after {@link #onChatReceived}; use for read-only reactions (sounds, logging last message). */
    void onChatObserved(Consumer<net.minecraft.network.chat.Component> observer);
}
```

## `Corim.bridge()`

```java
package dev.py54.crunch.corimlib;

import java.util.ServiceLoader;

public final class Corim {
    public static PlatformBridge bridge() { /* ... */ }
}
```

`Corim.bridge()` lazily resolves and caches the current loader's implementation via `ServiceLoader.load(PlatformBridge.class, ...).findFirst()`. If no implementation is registered, it throws `IllegalStateException` with a message telling you exactly what's missing. Call it from anywhere in shared code:

```java
Corim.bridge().registerKeybind(myKeyMapping);
Corim.bridge().onClientTick(mc -> { /* every client tick */ });
```

!!! warning "Single global singleton — read this before depending on CorimLib from more than one mod"
    `Corim.bridge()` caches the *first* `PlatformBridge` found via `ServiceLoader.findFirst()`. It has no concept of "which mod is calling me." If two independent mods both depend on CorimLib and both ship their own `PlatformBridge` implementation, `ServiceLoader` resolution order across the combined classpath is not something you control, and both mods' `Corim.bridge()` calls resolve to the *same* implementation instance. See [Limitations](../reference/limitations.md#single-global-platformbridge-singleton) for the full explanation before you rely on this in a multi-mod scenario.

## Method-by-method

| Method | Purpose | Example Fabric implementation |
|---|---|---|
| `loaderName()` | A human-readable loader name, useful in log lines (e.g. `"MyMod initialized ({}): ..."`). | Returns the literal string `"Fabric"` / `"NeoForge"` / `"Forge"`. |
| `configDir()` | Where to read/write your mod's config JSON. | `FabricLoader.getInstance().getConfigDir().resolve("mymod")` |
| `registerKeybind(KeyMapping)` | Registers a `KeyMapping` with the loader so it appears in the vanilla Controls screen and receives input. | `KeyMappingHelper.registerKeyMapping(mapping)` |
| `onClientTick(Consumer<Minecraft>)` | Runs every client tick. | `ClientTickEvents.END_CLIENT_TICK.register(callback::accept)` |
| `onClientStarted(Consumer<Minecraft>)` | Runs once, after the client has fully finished starting. **Load your config here, not at mod-init time** — see the callout below. | `ClientLifecycleEvents.CLIENT_STARTED.register(callback::accept)` |
| `registerHudElement(Identifier, HudRenderer)` | Registers a HUD overlay layer. | `HudElementRegistry.addLast(id, renderer::render)` |
| `onItemTooltip(TooltipHandler)` | Runs when an item tooltip is being built, letting you read/mutate the line list. | `ItemTooltipCallback.EVENT.register(handler::onTooltip)` |
| `onChatReceived(ChatFilter)` | Runs on every incoming chat message; returning `false` hides it. | `ClientReceiveMessageEvents.ALLOW_CHAT.register(...)` |
| `onChatObserved(Consumer<Component>)` | Runs *after* `onChatReceived`, for read-only reactions (sounds, logging) — the message may already be hidden by that point. | `ClientReceiveMessageEvents.CHAT.register(...)` |

!!! danger "Don't touch `Minecraft.getInstance().options` from an `onToggle`/`onChange` handler without a null guard"
    `ConfigManager.load()` at entrypoint time fires every `Feature`'s `onToggle`/`Setting`'s `onChange` handler synchronously (via `Feature.setEnabled`), and `Minecraft.getInstance().options` doesn't exist yet at that point. This is exactly why `onClientStarted` exists as a separate hook from mod-init — a correct bootstrap defers `ConfigManager.load(...)` until `Corim.bridge().onClientStarted(...)` fires, specifically because this was found the hard way in production use (see [Limitations](../reference/limitations.md)).

## Example implementations, side by side

CorimLib itself ships **no** `PlatformBridge` implementation — every consuming mod provides its own, in its own `fabric`/`neoforge`/`forge` modules. Full, real, production-grade code for each loader is on the corresponding setup page:

- [Fabric implementation](../getting-started/fabric-setup.md)
- [NeoForge implementation](../getting-started/neoforge-setup.md) (has the extra mod-bus-event queueing wrinkle)
- [Forge implementation](../getting-started/forge-setup.md) (has the `Predicate` cast wrinkle on `onChatReceived`)
