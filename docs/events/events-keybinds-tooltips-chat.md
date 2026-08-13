# Events, Keybinds, Tooltips & Chat

All of these are exposed through [`PlatformBridge`](../core/platform-bridge.md) — you never touch a loader's event API directly from shared code.

## Tick and startup events

```java
Corim.bridge().onClientTick(mc -> {
    // runs every client tick
});

Corim.bridge().onClientStarted(mc -> {
    // runs once, after the client has fully finished starting - Options/etc. exist here.
    // This is where you should call ConfigManager.load(...) - see Config System.
});
```

## Keybinds

### `CorimKeybinds`

CorimLib defines one shared keybind category every registered `KeyMapping` should use:

```java
public final class CorimKeybinds {
    public static final KeyMapping.Category CATEGORY =
            KeyMapping.Category.register(Identifier.fromNamespaceAndPath("crunch", "menu"));

    public static KeyMapping openMenu; // set by CrunchFeatures.bootstrap()
}
```

`CorimKeybinds.CATEGORY` is namespaced `"crunch"` — if you're registering your own mod's keybinds (not just Crunch's), define your own `KeyMapping.Category` under your own namespace rather than reusing this one, since it will group your keybind under a category literally labeled for Crunch's menu in the vanilla Controls screen.

### Registering a keybind

```java
KeyMapping myKey = new KeyMapping("key.mymod.do_thing", InputConstants.Type.KEYSYM, GLFW.GLFW_KEY_G, myCategory);
Corim.bridge().registerKeybind(myKey);
Corim.bridge().onClientTick(mc -> {
    while (myKey.consumeClick()) {
        // handle the press
    }
});
```

This is the exact real pattern CorimLib's own bootstrap uses to open `CrunchMainScreen`:

```java
CorimKeybinds.openMenu = new KeyMapping("key.crunch.open_menu", InputConstants.Type.KEYSYM, GLFW.GLFW_KEY_K, CorimKeybinds.CATEGORY);
Corim.bridge().registerKeybind(CorimKeybinds.openMenu);
Corim.bridge().onClientTick(mc -> {
    while (CorimKeybinds.openMenu.consumeClick()) {
        if (mc.canInterruptScreen()) {
            mc.setScreenAndShow(new CrunchMainScreen());
        }
    }
});
```

## Tooltips

```java
@FunctionalInterface
public interface TooltipHandler {
    void onTooltip(ItemStack stack, Item.TooltipContext context, TooltipFlag flag, List<Component> lines);
}
```

```java
Corim.bridge().onItemTooltip((stack, context, flag, lines) -> {
    lines.add(Component.literal("Extra tooltip line"));
});
```

`context` is loader-dependent: Fabric and NeoForge both pass a real `Item.TooltipContext`; **Forge always passes `null`** here, because Forge's own `ItemTooltipEvent` has no equivalent parameter (its signature is `(ItemStack, Player, List<Component>, TooltipFlag)`). If your tooltip logic reads `context`, guard against `null` or accept that the feature degrades on Forge.

## Chat

Two separate hooks, run in order:

```java
@FunctionalInterface
public interface ChatFilter {
    /** Return false to hide the message. */
    boolean test(Component message);
}
```

```java
// Runs first - can hide the message.
Corim.bridge().onChatReceived(message -> {
    return !message.getString().contains("spam"); // false = hidden
});

// Runs after onChatReceived - read-only, for reactions like sounds or logging the last message.
// The message may already be hidden by the time this fires.
Corim.bridge().onChatObserved(message -> {
    // e.g. play a sound if the message mentions the player's name
});
```

!!! note "Forge's cancellation mechanism is different under the hood"
    On Forge, `ClientChatReceivedEvent.BUS` is a `CancellableEventBus` — there's no `event.setCanceled(boolean)` to call from a plain listener. The real `ForgePlatformBridge` implementation registers via the `addListener(Predicate<T>)` overload instead, where returning `true` cancels the event, and casts the lambda explicitly (`(Predicate<ClientChatReceivedEvent>) event -> !filter.test(event.getMessage())`) because the ambiguous overload won't otherwise compile. You don't need to know this to *call* `ChatFilter` — it's already handled inside the bridge implementation — but it explains why Forge's chat handling code looks different from Fabric's/NeoForge's if you go reading the source.

## HUD elements

Covered in full on the [HUD System](../hud/hud-system.md) page — `Corim.bridge().registerHudElement(Identifier id, HudRenderer renderer)` is the entry point.
