# Features & Settings

The `Feature`/`Setting`/`FeatureRegistry` classes live in the `common` module (`dev.py54.corimlib.feature` and `dev.py54.corimlib.settings` packages) and have **zero Minecraft dependency** — this is the one part of CorimLib you could theoretically use outside a Minecraft context entirely.

## `Feature`

A single toggleable feature: id, display name, description, category, enabled state, and a list of `Setting<?>`s.

```java
public class Feature {
    public Feature(String id, String name, String description, Category category, boolean defaultEnabled);

    public Feature addSetting(Setting<?> setting);       // fluent, returns this
    public Feature onToggle(Consumer<Boolean> listener);  // fluent, returns this

    public String id();
    public String name();
    public String description();
    public Category category();
    public List<Setting<?>> settings();
    public boolean isEnabled();
    public void setEnabled(boolean enabled); // fires onToggle synchronously
    public void toggle();
    public void resetAll(); // resets this feature and every one of its settings to defaults
    public boolean matchesSearch(String query); // case-insensitive, checks name/description/category/settings
}
```

Example — a real one from CorimLib's own `VisualFeatures`:

```java
Feature particleReduction = new Feature("visual.particles", "Particle Reduction",
        "Reduces or disables particle rendering.", Category.VISUAL, false);
EnumSetting<ParticleStatus> particleLevel = new EnumSetting<>("level", "Level", "Particle amount.",
        ParticleStatus.DECREASED, ParticleStatus.values());
particleReduction.addSetting(particleLevel);
particleReduction.onToggle(enabled ->
        Minecraft.getInstance().options.particles().set(enabled ? particleLevel.get() : ParticleStatus.ALL));
particleLevel.onChange(v -> { if (particleReduction.isEnabled()) Minecraft.getInstance().options.particles().set(v); });
FeatureRegistry.register(particleReduction);
```

## `Category`

A fixed enum, not extensible per-mod:

```java
public enum Category {
    HUD, VISUAL, TOOLTIPS, CHAT, CONTROLS, ACCESSIBILITY, SCREENSHOTS, PERFORMANCE, INTERFACE, GENERAL;
    public String displayName();
}
```

If your feature doesn't fit one of the existing ten categories, the closest match (usually `GENERAL`) is your only option — see [Limitations](../reference/limitations.md).

## `FeatureRegistry`

A single, static, process-wide lookup — **not namespaced per mod**:

```java
public final class FeatureRegistry {
    public static Feature register(Feature feature); // throws IllegalStateException on duplicate id
    public static Optional<Feature> get(String id);
    public static List<Feature> all();
    public static List<Feature> byCategory(Category category);
    public static List<Feature> search(String query);
    public static void resetAll();
}
```

Because it's a static singleton, every mod that depends on CorimLib and calls `FeatureRegistry.register(...)` adds to the *same* shared list — including any other CorimLib-dependent mod's features, if one happens to be installed alongside yours. Prefix your feature ids to avoid collisions (a common convention is `<category>.<name>`, e.g. `hud.fps`, `visual.particles`) — see [Limitations](../reference/limitations.md#global-featureregistry) for the full implication of this.

## `Setting<T>`

The abstract base every concrete setting type extends:

```java
public abstract class Setting<T> {
    protected Setting(String id, String name, String description, SettingType type, T defaultValue);

    public String id();
    public String name();
    public String description();
    public SettingType type();
    public T defaultValue();
    public T get();
    public void set(T newValue);   // fires onChange
    public void reset();           // set(defaultValue)
    public Setting<T> onChange(Consumer<T> listener); // fluent, returns this
}
```

## The seven concrete setting types

All in package `dev.py54.corimlib.settings`:

| Class | Value type | Extra fields | Notes |
|---|---|---|---|
| `BooleanSetting` | `Boolean` | — | Simple on/off. |
| `IntSetting` | `Integer` | `min()`, `max()` | `set()` clamps to `[min, max]`. |
| `DecimalSetting` | `Double` | `min()`, `max()`, `step()` | Used for sliders, e.g. Fullbright intensity. `set()` clamps to `[min, max]`. |
| `EnumSetting<E extends Enum<E>>` | `E` | `options()` | Backed by any enum's `values()`. |
| `ColorSetting` | `Integer` | — | Packed ARGB int, e.g. `0xFFAAAAAA`. |
| `KeybindSetting` | `Integer` | — | A GLFW key code for a feature's own hotkey, separate from the vanilla keybind screen entry. `-1` means unbound. |
| `TextSetting` | `String` | `maxLength()` | Free text, e.g. a chat filter keyword. |

```java
public enum SettingType { BOOLEAN, INTEGER, DECIMAL, ENUM, COLOR, KEYBIND, TEXT }
```

Real construction examples, taken directly from CorimLib's own feature registrations:

```java
new BooleanSetting("verbose", "Verbose Logging", "Log extra detail.", false);

new IntSetting("x", "X", "Horizontal position.", 4, 0, 3000);

new DecimalSetting("scale", "Scale", "Text scale", 1.0, 0.5, 3.0, 0.05);

new EnumSetting<>("preset", "Preset", "Which preset to apply.", Preset.BALANCED, Preset.values());

new ColorSetting("level1_color", "Level I Color", "Color for level I enchantments.", 0xFFAAAAAA);

new KeybindSetting("hotkey", "Hotkey", "This feature's own hotkey.", -1);

new TextSetting("keyword", "Keyword", "Message must contain this text.", "", 64);
```

## Putting it together

```java
Feature myFeature = new Feature("mymod.example", "Example Feature",
        "Does something useful.", Category.GENERAL, false);

BooleanSetting verbose = new BooleanSetting("verbose", "Verbose Logging", "Log extra detail.", false);
myFeature.addSetting(verbose);

myFeature.onToggle(enabled -> {
    if (enabled) {
        // start doing the thing
    } else {
        // stop doing the thing
    }
});
verbose.onChange(v -> { /* react to the setting changing independently of the toggle */ });

FeatureRegistry.register(myFeature);
```

Once registered, your feature is automatically visible in [`CrunchMainScreen`](../gui/gui-and-theming.md) (if a dependent mod opens it), persisted by [`ConfigManager`](config-system.md), and included in [`ProfileManager`](config-system.md) snapshots — you don't build any of that plumbing yourself.
