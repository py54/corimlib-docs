# Forge Setup

!!! danger "Higher risk than the other two loaders"
    A real dependent mod's `:forge:runClient` currently crashes with a JPMS module-resolution error (`java.lang.module.ResolutionException: Modules crunch and main export package ... to module ...`) that has not been root-caused — it's a ForgeGradle/JPMS module-layer construction issue, not a Minecraft API problem. Forge support is **build-verified only** (compiles, correct metadata, unzipped and checked), not confirmed to actually launch. Treat everything below as correct-as-written but not yet proven at runtime. See [Limitations](../reference/limitations.md).

## 1. Implement `PlatformBridge`

Forge's event system is genuinely different from NeoForge's despite the shared ancestry — every event type exposes a static `EventType.BUS` field with `.addListener(...)`, no injected mod-event-bus parameter and no mod-bus/game-bus split to worry about for timing. A real, production-shipped implementation:

```java
package dev.creator.yourmodname.forge;

import dev.py54.corimlib.ChatFilter;
import dev.py54.corimlib.HudRenderer;
import dev.py54.corimlib.PlatformBridge;
import dev.py54.corimlib.TooltipHandler;
import net.minecraft.client.KeyMapping;
import net.minecraft.client.Minecraft;
import net.minecraft.network.chat.Component;
import net.minecraft.resources.Identifier;
import net.minecraftforge.client.event.AddGuiOverlayLayersEvent;
import net.minecraftforge.client.event.ClientChatReceivedEvent;
import net.minecraftforge.client.event.RegisterKeyMappingsEvent;
import net.minecraftforge.event.TickEvent;
import net.minecraftforge.event.entity.player.ItemTooltipEvent;
import net.minecraftforge.fml.loading.FMLPaths;

import java.nio.file.Path;
import java.util.function.Consumer;
import java.util.function.Predicate;

public final class ForgePlatformBridge implements PlatformBridge {
    private static boolean startedFired = false;

    @Override
    public String loaderName() {
        return "Forge";
    }

    @Override
    public Path configDir() {
        return FMLPaths.CONFIGDIR.get().resolve("mymod");
    }

    @Override
    public void registerKeybind(KeyMapping mapping) {
        RegisterKeyMappingsEvent.BUS.addListener(event -> event.register(mapping));
    }

    @Override
    public void onClientTick(Consumer<Minecraft> callback) {
        TickEvent.ClientTickEvent.Post.BUS.addListener(event -> callback.accept(Minecraft.getInstance()));
    }

    @Override
    public void onClientStarted(Consumer<Minecraft> callback) {
        TickEvent.ClientTickEvent.Post.BUS.addListener(event -> {
            if (!startedFired) {
                startedFired = true;
                callback.accept(Minecraft.getInstance());
            }
        });
    }

    @Override
    public void registerHudElement(Identifier id, HudRenderer renderer) {
        AddGuiOverlayLayersEvent.BUS.addListener(event -> event.getLayeredDraw().add(id, renderer::render));
    }

    @Override
    public void onItemTooltip(TooltipHandler handler) {
        ItemTooltipEvent.BUS.addListener(event ->
                handler.onTooltip(event.getItemStack(), null, event.getFlags(), event.getToolTip()));
    }

    @Override
    public void onChatReceived(ChatFilter filter) {
        // ClientChatReceivedEvent.BUS is a CancellableEventBus: listeners are predicates,
        // returning true cancels the event - there's no event.setCanceled(boolean) here.
        ClientChatReceivedEvent.BUS.addListener((Predicate<ClientChatReceivedEvent>) event -> !filter.test(event.getMessage()));
    }

    @Override
    public void onChatObserved(Consumer<Component> observer) {
        ClientChatReceivedEvent.BUS.addListener((Consumer<ClientChatReceivedEvent>) event -> observer.accept(event.getMessage()));
    }
}
```

The `dev.creator.yourmodname.forge` package is yours to choose — only the `dev.py54.corimlib.*` imports are CorimLib's actual, fixed API.

!!! note "The cast on `onChatReceived` is required, not stylistic"
    Because both `addListener(Consumer<T>)` and `addListener(Predicate<T>)` overloads exist on `CancellableEventBus`, a plain lambda is ambiguous — you must cast explicitly to `(Predicate<ClientChatReceivedEvent>)`, exactly as shown above, or the build won't compile.

## 2. Register it via `ServiceLoader`

```
dev.creator.yourmodname.forge.ForgePlatformBridge
```

## 3. `ItemTooltipEvent` has fewer parameters than Fabric/NeoForge

Forge's `ItemTooltipEvent` signature is `(ItemStack, Player, List<Component>, TooltipFlag)` — no `TooltipContext` parameter at all, which is why the implementation above passes `null` for that argument to `TooltipHandler.onTooltip`. If your own tooltip logic reads the context parameter, it will always be `null` on Forge.

## 4. Toolchain notes

- ForgeGradle `[7.0.17,8)` via `id "net.minecraftforge.gradle"` — do **not** also apply `id "java"` explicitly, ForgeGradle applies it itself.
- Dependency form: `implementation minecraft.dependency("net.minecraftforge:forge:${minecraft_version}-${forge_version}")`, with `repositories { minecraft.mavenizer(it); maven fg.forgeMaven; maven fg.minecraftLibsMaven; mavenCentral() }`.
- **`mods.toml`'s `@MC_VERSION@`/`@FORGE_SPEC_VERSION@`/`@MC_NEXT_VERSION@` tokens are not actually substituted by ForgeGradle at real build time** — a real, previously-shipped bug: release jars shipped literal unexpanded tokens, which crashes on load independent of the JPMS issue above. Fix it with a real template (`src/main/templates/META-INF/mods.toml`) plus an explicit Gradle `ProcessResources` task doing the property expansion yourself (mirroring how NeoForge's `mods.toml` has always worked) — don't trust ForgeGradle to do this for you.
- `corimlib`'s dependency on `net.fabricmc:fabric-loader` (needed purely for Mixin annotations) must be `compileOnly`, or it leaks onto Forge's classpath and fails to resolve at both shadowJar time and `runClient` time (Forge's repos don't have it).
