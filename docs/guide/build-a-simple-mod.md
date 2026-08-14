# Guide: Build a Simple Mod

This walks through a complete, minimal Fabric mod built on CorimLib — one toggleable feature with a setting, one HUD line, and a keybind to open the settings menu. Every class, method, and import used below is real (see [PlatformBridge API](../core/platform-bridge.md), [Features & Settings](../core/features-and-settings.md), and [HUD System](../hud/hud-system.md) for the full reference). Nothing here is Minecraft-version-specific beyond what CorimLib itself supports — see [Supported Versions & Loaders](../reference/supported-versions.md).

We'll build **"Compass Mod"**: a HUD line showing which direction the player is facing, toggleable and configurable from CorimLib's menu.

## 1. Project layout

```
compassmod/
└── src/main/
    ├── java/dev/example/compassmod/
    │   ├── CompassFeatures.java
    │   └── fabric/
    │       ├── CompassModClient.java
    │       └── FabricPlatformBridge.java
    └── resources/
        ├── fabric.mod.json
        └── META-INF/services/dev.py54.crunch.corimlib.PlatformBridge
```

## 2. Gradle dependency

See [Gradle Setup](../getting-started/gradle-setup.md) for the full explanation of the `compileOnly`/`implementation` split and the current lack of a public Maven repo:

```groovy
dependencies {
    minecraft "com.mojang:minecraft:${project.minecraft_version}"
    implementation "net.fabricmc:fabric-loader:${project.loader_version}"
    implementation "net.fabricmc.fabric-api:fabric-api:${project.fabric_api_version}"

    compileOnly "dev.py54.corimlib:corimlib:${project.corimlib_version}"
    implementation "dev.py54.corimlib:corimlib-fabric:${project.corimlib_version}"
}
```

## 3. Register the feature and HUD line

```java
package dev.example.compassmod;

import dev.py54.crunch.config.ConfigManager;
import dev.py54.crunch.corimlib.Corim;
import dev.py54.crunch.corimlib.CorimKeybinds;
import dev.py54.crunch.corimlib.gui.CrunchMainScreen;
import dev.py54.crunch.corimlib.hud.HudLineFeature;
import dev.py54.crunch.corimlib.hud.HudOverlayElement;
import com.mojang.blaze3d.platform.InputConstants;
import net.minecraft.client.KeyMapping;
import net.minecraft.client.Minecraft;
import net.minecraft.resources.Identifier;
import org.lwjgl.glfw.GLFW;

import java.nio.file.Path;

public final class CompassFeatures {
    private static final String[] DIRECTIONS = {"S", "SW", "W", "NW", "N", "NE", "E", "SE"};

    private CompassFeatures() {
    }

    public static void bootstrap() {
        // A HudLineFeature auto-registers itself with FeatureRegistry and HudRegistry -
        // no separate FeatureRegistry.register(...) call needed for it.
        new HudLineFeature("compass_direction", "Compass Direction",
                "Shows the compass direction you're facing.", true, 4, 4) {
            @Override
            public String text(Minecraft mc) {
                if (mc.player == null) return null;
                int index = Math.round(mc.player.getYRot() / 45f) & 7;
                return "Facing: " + DIRECTIONS[index];
            }
        };

        // One HudOverlayElement draws every registered HudLineFeature - register it once.
        Corim.bridge().registerHudElement(
                Identifier.fromNamespaceAndPath("compassmod", "hud_overlay"),
                new HudOverlayElement());

        // A keybind to open CorimLib's real settings menu.
        KeyMapping openMenu = new KeyMapping("key.compassmod.open_menu",
                InputConstants.Type.KEYSYM, GLFW.GLFW_KEY_G, CorimKeybinds.CATEGORY);
        Corim.bridge().registerKeybind(openMenu);
        Corim.bridge().onClientTick(mc -> {
            while (openMenu.consumeClick()) {
                if (mc.canInterruptScreen()) {
                    mc.setScreenAndShow(new CrunchMainScreen());
                }
            }
        });

        // Load only after the client has fully started - see Config System for why loading
        // any earlier (e.g. directly in onInitializeClient) touches Options before it exists.
        Corim.bridge().onClientStarted(mc -> ConfigManager.load(configFile()));
        Runtime.getRuntime().addShutdownHook(new Thread(() -> ConfigManager.save(configFile())));
    }

    private static Path configFile() {
        return Corim.bridge().configDir().resolve("compassmod.json");
    }
}
```

!!! note "`CrunchMainScreen` will be titled \"Crunch\""
    As covered in [GUI & Theming](../gui/gui-and-theming.md), opening `CrunchMainScreen` gives you a fully working settings UI for free, but it's titled and branded "Crunch," not "Compass Mod." That's an accurate tradeoff of reusing CorimLib's actual shipped screen rather than building your own — there's currently no supported way to relabel it.

## 4. Implement `PlatformBridge` for Fabric

```java
package dev.example.compassmod.fabric;

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
        return FabricLoader.getInstance().getConfigDir().resolve("compassmod");
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

`src/main/resources/META-INF/services/dev.py54.crunch.corimlib.PlatformBridge`:

```
dev.example.compassmod.fabric.FabricPlatformBridge
```

## 5. Entrypoint

```java
package dev.example.compassmod.fabric;

import dev.example.compassmod.CompassFeatures;
import net.fabricmc.api.ClientModInitializer;

public final class CompassModClient implements ClientModInitializer {
    @Override
    public void onInitializeClient() {
        CompassFeatures.bootstrap();
    }
}
```

## 6. `fabric.mod.json`

```json
{
  "schemaVersion": 1,
  "id": "compassmod",
  "version": "1.0.0",
  "name": "Compass Mod",
  "environment": "client",
  "entrypoints": {
    "client": ["dev.example.compassmod.fabric.CompassModClient"]
  },
  "depends": {
    "fabricloader": ">=0.19.3",
    "minecraft": "~26.2",
    "corimlib": ">=1.0.0"
  }
}
```

## What you get, and what you're opting into

- A working, positioned, auto-stacking HUD line, toggleable from a real settings screen — with **zero** custom GUI code written.
- Persistence: once the depending code calls `ConfigManager.load`/`save` against your own config path (see [Config System](../core/config-system.md)) at the right lifecycle point, your feature's enabled state is saved automatically.
- The tradeoff: `CrunchMainScreen` ships with fixed "Crunch" branding you can't relabel, `FeatureRegistry` is a single global list shared with any other CorimLib-dependent mod present, and `Corim.bridge()` is a single global `PlatformBridge` singleton — read [Limitations & Experimental APIs](../reference/limitations.md) before shipping something that depends on any of those three facts holding a particular way.
