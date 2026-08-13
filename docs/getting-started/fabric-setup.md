# Fabric Setup

Fabric is CorimLib's original, most-verified target (`runClient` reaches a loaded world with no crash on both CorimLib and Crunch).

## 1. Implement `PlatformBridge`

Every loader module provides exactly one implementation of [`PlatformBridge`](../core/platform-bridge.md). This is Crunch's real, shipped Fabric implementation — every method maps to one Fabric API call:

```java
package dev.py54.crunch.fabric;

import dev.py54.crunch.corimlib.ChatFilter;
import dev.py54.crunch.corimlib.HudRenderer;
import dev.py54.crunch.corimlib.PlatformBridge;
import dev.py54.crunch.corimlib.TooltipHandler;
import net.fabricmc.fabric.api.client.event.lifecycle.v1.ClientLifecycleEvents;
import net.fabricmc.fabric.api.client.event.lifecycle.v1.ClientTickEvents;
import net.fabricmc.fabric.api.client.item.v1.ItemTooltipCallback;
import net.fabricmc.fabric.api.client.keymapping.v1.KeyMappingHelper;
import net.fabricmc.fabric.api.client.message.v1.ClientReceiveMessageEvents;
import net.fabricmc.fabric.api.client.rendering.v1.hud.HudElementRegistry;
import net.fabricmc.loader.api.FabricLoader;
import net.minecraft.client.KeyMapping;
import net.minecraft.client.Minecraft;
import net.minecraft.resources.Identifier;

import java.nio.file.Path;
import java.util.function.Consumer;

public final class FabricPlatformBridge implements PlatformBridge {

    @Override
    public String loaderName() {
        return "Fabric";
    }

    @Override
    public Path configDir() {
        return FabricLoader.getInstance().getConfigDir().resolve("mymod");
    }

    @Override
    public void registerKeybind(KeyMapping mapping) {
        KeyMappingHelper.registerKeyMapping(mapping);
    }

    @Override
    public void onClientTick(Consumer<Minecraft> callback) {
        ClientTickEvents.END_CLIENT_TICK.register(callback::accept);
    }

    @Override
    public void onClientStarted(Consumer<Minecraft> callback) {
        ClientLifecycleEvents.CLIENT_STARTED.register(callback::accept);
    }

    @Override
    public void registerHudElement(Identifier id, HudRenderer renderer) {
        HudElementRegistry.addLast(id, renderer::render);
    }

    @Override
    public void onItemTooltip(TooltipHandler handler) {
        ItemTooltipCallback.EVENT.register(handler::onTooltip);
    }

    @Override
    public void onChatReceived(ChatFilter filter) {
        ClientReceiveMessageEvents.ALLOW_CHAT.register(
                (message, signedMessage, sender, params, receptionTimestamp) -> filter.test(message));
    }

    @Override
    public void onChatObserved(Consumer<net.minecraft.network.chat.Component> observer) {
        ClientReceiveMessageEvents.CHAT.register(
                (message, signedMessage, sender, params, receptionTimestamp) -> observer.accept(message));
    }
}
```

(Only `configDir()`'s folder name — `"mymod"` above vs. Crunch's real `"crunch"` — needs to change for your own mod; everything else is loader glue that stays the same shape.)

Fabric API dependencies used above: `fabric-key-mapping-api-v1`, `fabric-lifecycle-events-v1`, `fabric-item-api-v1`, `fabric-message-api-v1`, `fabric-rendering-v1` (the `hud` sub-package). All are part of the standard `fabric-api` meta-artifact.

## 2. Register it via `ServiceLoader`

Create `src/main/resources/META-INF/services/dev.py54.crunch.corimlib.PlatformBridge` containing a single line — the fully-qualified name of your implementation:

```
dev.py54.crunch.fabric.FabricPlatformBridge
```

This is exactly Crunch's own file content (`fabric/src/main/resources/META-INF/services/dev.py54.crunch.corimlib.PlatformBridge`), just swap the class name for yours.

## 3. Call your bootstrap from the mod entrypoint

Crunch's real Fabric entrypoint is three lines:

```java
package dev.py54.crunch.fabric.client;

import dev.py54.crunch.corimlib.features.CrunchFeatures;
import net.fabricmc.api.ClientModInitializer;

public final class CrunchClient implements ClientModInitializer {
    @Override
    public void onInitializeClient() {
        CrunchFeatures.bootstrap();
    }
}
```

`CrunchFeatures.bootstrap()` is Crunch's own feature-registration method — in your own mod, this would be wherever you call `FeatureRegistry.register(...)` for your own [`Feature`s](../core/features-and-settings.md), register your keybinds via `Corim.bridge().registerKeybind(...)`, and register your HUD element via `Corim.bridge().registerHudElement(...)`. Wire it into `fabric.mod.json`'s `"client"` entrypoints list the normal Fabric way.

## 4. Declare the dependency in `fabric.mod.json`

If you're publishing your own mod that depends on CorimLib as a companion mod (the same relationship Crunch has with it), add it to `depends`:

```json
{
  "depends": {
    "fabricloader": ">=0.19.3",
    "minecraft": "~26.2",
    "corimlib": ">=1.0.0"
  }
}
```
