# HUD System

## `HudRenderer`

The lowest-level abstraction — anything that can draw onto the HUD:

```java
@FunctionalInterface
public interface HudRenderer {
    void render(GuiGraphicsExtractor extractor, DeltaTracker deltaTracker);
}
```

Register one via `Corim.bridge().registerHudElement(Identifier id, HudRenderer renderer)`. If you just need a single custom HUD element, you can implement `HudRenderer` directly and stop there. For a full text-line system with positioning, dragging, and auto-stacking (what CorimLib itself uses for its ~21 HUD lines), keep reading.

## `HudLineFeature`

An abstract base class combining a [`Feature`](../core/features-and-settings.md) (so the line is toggleable and shows up in the settings GUI) with `x`/`y`/`scale`/`customPosition` settings and a `text(Minecraft)` method you override:

```java
public abstract class HudLineFeature {
    public final Feature feature;
    public final IntSetting x;
    public final IntSetting y;
    public final DecimalSetting scale;
    public final BooleanSetting customPosition;

    protected HudLineFeature(String id, String name, String description,
                              boolean defaultEnabled, int defaultX, int defaultY);

    /** Returns the text to draw this frame, or null/blank to draw nothing. */
    public abstract String text(Minecraft mc);
}
```

Constructing a `HudLineFeature` subclass automatically calls `FeatureRegistry.register(feature)` and adds itself to `HudRegistry.LINES` — you don't register it separately. The `feature.id()` is `"hud." + id` (the constructor prepends the prefix for you).

Real example — CorimLib's own FPS line, in full:

```java
new HudLineFeature("fps", "FPS", "Shows current frames per second.", true, 4, 4) {
    @Override
    public String text(Minecraft mc) {
        return "FPS: " + FrameStats.fps();
    }
};
```

And one that returns `null` conditionally (so it draws nothing rather than an empty line):

```java
new HudLineFeature("ping", "Ping", "Shows your connection latency in milliseconds.", false, 4, 14) {
    @Override
    public String text(Minecraft mc) {
        if (mc.player == null || mc.getConnection() == null) return null;
        var info = mc.getConnection().getPlayerInfo(mc.player.getUUID());
        return info == null ? null : "Ping: " + info.getLatency() + "ms";
    }
};
```

## Auto-stacking, no gaps

By default, HUD lines don't render at fixed positions forever — they **auto-stack** with the other enabled, non-custom-positioned lines, ordered by whichever order they were most recently *enabled* in, re-derived every frame. A line that's currently disabled — or currently returns `null`/blank text, like Ping above when disconnected — doesn't leave a gap where it would have been. Dragging a line in the HUD Editor sets its `customPosition` setting to `true`, which pins it at its dragged x/y instead of participating in the stack; toggling `customPosition` back off (or using "Reset All Positions" in the editor) rejoins the auto stack.

## `HudRegistry`

The registry backing all of this — also where the stacking math actually lives:

```java
public final class HudRegistry {
    public static final List<HudLineFeature> LINES;
    public static final int STACK_BASE_X; // 4
    public static final int STACK_BASE_Y; // 4
    public static final int STACK_LINE_HEIGHT; // 10

    public static int nextOrder();
    public static List<HudLineFeature> autoStackOrder();
    public static Map<HudLineFeature, int[]> effectivePositions();
}
```

`effectivePositions()` is shared by the live HUD renderer *and* the HUD Editor's drag preview/hit-testing, specifically so what you see while dragging a line is guaranteed to match what actually renders — there's no separate "preview math" that could drift out of sync with the real thing.

## `HudOverlayElement`

The actual `HudRenderer` implementation that draws every enabled `HudLineFeature` — this is what gets registered once via `Corim.bridge().registerHudElement(...)`, rather than registering one HUD element per line:

```java
public final class HudOverlayElement implements HudRenderer {
    public static boolean editorOpen; // set true while CrunchHudEditorScreen is open to suppress the live overlay
    @Override
    public void render(GuiGraphicsExtractor extractor, DeltaTracker deltaTracker) { /* ... */ }
}
```

Registering it:

```java
Corim.bridge().registerHudElement(
        Identifier.fromNamespaceAndPath("mymod", "hud_overlay"),
        new HudOverlayElement());
```

If you're building your own mod, register your own `HudOverlayElement()` under your own namespace, or implement `HudRenderer` directly if you don't need the full auto-stacking line system.

## `FrameStats`

A small self-counted (not read from a vanilla field) FPS/frame-time tracker used by the FPS HUD line and `CrunchPerformanceMonitorScreen`'s rolling graph:

```java
Corim.bridge().onClientTick or per-render: FrameStats.onFrame();
int fps = FrameStats.fps();
```

`HudOverlayElement.render(...)` already calls `FrameStats.onFrame()` on every frame, so if you're using the standard `HudOverlayElement`/`HudLineFeature` flow, `FrameStats` is already being fed for you.
