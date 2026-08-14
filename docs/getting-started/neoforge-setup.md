# NeoForge Setup

NeoForge is fully verified on Minecraft 26.2 (`runClient` reaches a loaded world). See [Supported Versions & Loaders](../reference/supported-versions.md) for exactly which Minecraft patches have a NeoForge build at all.

## 1. Implement `PlatformBridge`

NeoForge's event bus has a wrinkle the other two loaders don't: `RegisterKeyMappingsEvent` and `RegisterGuiLayersEvent` are **mod-bus events that fire once, early** — before your feature-registration code (which is what actually calls `registerKeybind`/`registerHudElement`) has run. A correct implementation queues into static lists and drains them when the event fires:

```java
package dev.creator.yourmodname.neoforge;

import dev.py54.crunch.corimlib.ChatFilter;
import dev.py54.crunch.corimlib.HudRenderer;
import dev.py54.crunch.corimlib.PlatformBridge;
import dev.py54.crunch.corimlib.TooltipHandler;
import net.minecraft.client.KeyMapping;
import net.minecraft.client.Minecraft;
import net.minecraft.network.chat.Component;
import net.minecraft.resources.Identifier;
import net.neoforged.fml.loading.FMLPaths;
import net.neoforged.neoforge.client.event.ClientChatReceivedEvent;
import net.neoforged.neoforge.client.event.ClientTickEvent;
import net.neoforged.neoforge.client.event.RegisterGuiLayersEvent;
import net.neoforged.neoforge.client.event.RegisterKeyMappingsEvent;
import net.neoforged.neoforge.common.NeoForge;
import net.neoforged.neoforge.event.entity.player.ItemTooltipEvent;

import java.nio.file.Path;
import java.util.ArrayList;
import java.util.List;
import java.util.function.Consumer;

public final class NeoForgePlatformBridge implements PlatformBridge {
    private static final List<KeyMapping> PENDING_KEYBINDS = new ArrayList<>();
    private static final List<Runnable> PENDING_HUD_LAYERS = new ArrayList<>();
    private static boolean startedFired = false;
    private static RegisterGuiLayersEvent pendingEvent;

    static void onRegisterKeyMappings(RegisterKeyMappingsEvent event) {
        PENDING_KEYBINDS.forEach(event::register);
    }

    static void onRegisterGuiLayers(RegisterGuiLayersEvent event) {
        pendingEvent = event;
        PENDING_HUD_LAYERS.forEach(Runnable::run);
        pendingEvent = null;
    }

    @Override
    public String loaderName() {
        return "NeoForge";
    }

    @Override
    public Path configDir() {
        return FMLPaths.CONFIGDIR.get().resolve("mymod");
    }

    @Override
    public void registerKeybind(KeyMapping mapping) {
        PENDING_KEYBINDS.add(mapping);
    }

    @Override
    public void onClientTick(Consumer<Minecraft> callback) {
        NeoForge.EVENT_BUS.addListener(ClientTickEvent.Post.class, event -> callback.accept(Minecraft.getInstance()));
    }

    @Override
    public void onClientStarted(Consumer<Minecraft> callback) {
        NeoForge.EVENT_BUS.addListener(ClientTickEvent.Post.class, event -> {
            if (!startedFired) {
                startedFired = true;
                callback.accept(Minecraft.getInstance());
            }
        });
    }

    @Override
    public void registerHudElement(Identifier id, HudRenderer renderer) {
        PENDING_HUD_LAYERS.add(() -> pendingEvent.registerAboveAll(id, renderer::render));
    }

    @Override
    public void onItemTooltip(TooltipHandler handler) {
        NeoForge.EVENT_BUS.addListener(ItemTooltipEvent.class, event ->
                handler.onTooltip(event.getItemStack(), event.getContext(), event.getFlags(), event.getToolTip()));
    }

    @Override
    public void onChatReceived(ChatFilter filter) {
        NeoForge.EVENT_BUS.addListener(ClientChatReceivedEvent.class, event -> {
            if (!filter.test(event.getMessage())) {
                event.setCanceled(true);
            }
        });
    }

    @Override
    public void onChatObserved(Consumer<Component> observer) {
        NeoForge.EVENT_BUS.addListener(ClientChatReceivedEvent.class, event -> observer.accept(event.getMessage()));
    }
}
```

The `dev.creator.yourmodname.neoforge` package is yours to choose — only the `dev.py54.crunch.corimlib.*` imports are CorimLib's actual, fixed API.

## 2. Register it via `ServiceLoader`

Same mechanism as every loader — `META-INF/services/dev.py54.crunch.corimlib.PlatformBridge`:

```
dev.creator.yourmodname.neoforge.NeoForgePlatformBridge
```

## 3. Wire the mod-bus listeners *before* bootstrapping

This is the part that's easy to get wrong on NeoForge: `RegisterKeyMappingsEvent`/`RegisterGuiLayersEvent` must be registered on the **mod event bus**, and that registration must happen *before* your feature-bootstrap code runs (even though the pending lists are only read once the event actually fires later):

```java
package dev.creator.yourmodname.neoforge;

import net.neoforged.api.distmarker.Dist;
import net.neoforged.bus.api.IEventBus;
import net.neoforged.fml.ModContainer;
import net.neoforged.fml.common.Mod;

@Mod(value = "mymod", dist = Dist.CLIENT)
public final class MyModNeoForge {
    public MyModNeoForge(IEventBus modEventBus, ModContainer modContainer) {
        modEventBus.addListener(NeoForgePlatformBridge::onRegisterKeyMappings);
        modEventBus.addListener(NeoForgePlatformBridge::onRegisterGuiLayers);

        MyModFeatures.bootstrap(); // your own feature-registration entrypoint
    }
}
```

## 4. Toolchain notes

- `net.neoforged.moddev` Gradle plugin, ModDevGradle's own `jarJar` task is for external Maven artifacts with version ranges, **not** sibling `project()` modules — use the `com.gradleup.shadow` plugin scoped to a dedicated configuration instead if you need to bundle sibling subprojects.
- `runClient`/mixin resource discovery only looks at sourceSets explicitly listed in `neoForge { mods { "yourmod" { sourceSet(...) } } }`.
- Add `id "org.gradle.toolchains.foojay-resolver-convention" version "1.0.0"` to your root `settings.gradle` — ModDevGradle's `downloadAssets` task needs an auto-provisioned Java 21 for an internal tool.
- `neoforge.mods.toml` should be a real Gradle-property-expanded template (a `ProcessResources` task reading from `src/main/templates/`) — ModDevGradle does not expand tokens in a plain `src/main/resources/neoforge.mods.toml` the way Fabric Loom expands `fabric.mod.json`.
