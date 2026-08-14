# API Reference

A structured reference of CorimLib's public classes, grouped by module and package. "Public API" here means classes/members actually declared `public` and reachable from outside CorimLib's own packages — see [GUI & Theming](../gui/gui-and-theming.md) for a case where that distinction really matters (`CrunchTheme` and the widget classes are **not** public despite being central to the visual design).

## `common` module — zero Minecraft dependency

### `dev.py54.corimlib.feature`

| Class | Kind | Key members |
|---|---|---|
| `Feature` | class | `Feature(String id, String name, String description, Category category, boolean defaultEnabled)`, `addSetting`, `onToggle`, `id/name/description/category/settings`, `isEnabled`, `setEnabled`, `toggle`, `resetAll`, `matchesSearch` |
| `Category` | enum | `HUD, VISUAL, TOOLTIPS, CHAT, CONTROLS, ACCESSIBILITY, SCREENSHOTS, PERFORMANCE, INTERFACE, GENERAL` + `displayName()` |
| `FeatureRegistry` | final class, static | `register`, `get`, `all`, `byCategory`, `search`, `resetAll` |
| `Favorites` | final class, static | `isFavorite`, `toggleFavorite`, `favoriteIds`, `markRecentlyUsed`, `recentIds`, `setFavoriteIds`, `setRecentIds` |

### `dev.py54.corimlib.settings`

| Class | Kind | Key members |
|---|---|---|
| `Setting<T>` | abstract class | `id/name/description/type/defaultValue`, `get`, `set`, `reset`, `onChange` |
| `SettingType` | enum | `BOOLEAN, INTEGER, DECIMAL, ENUM, COLOR, KEYBIND, TEXT` |
| `BooleanSetting` | class extends `Setting<Boolean>` | constructor only |
| `IntSetting` | class extends `Setting<Integer>` | + `min()`, `max()`; `set` clamps |
| `DecimalSetting` | class extends `Setting<Double>` | + `min()`, `max()`, `step()`; `set` clamps |
| `EnumSetting<E extends Enum<E>>` | class extends `Setting<E>` | + `options()` |
| `ColorSetting` | class extends `Setting<Integer>` | packed ARGB |
| `KeybindSetting` | class extends `Setting<Integer>` | GLFW key code, `-1` = unbound |
| `TextSetting` | class extends `Setting<String>` | + `maxLength()` |

### `dev.py54.corimlib.config`

| Class | Kind | Key members |
|---|---|---|
| `ConfigManager` | final class, static | `toJson`, `applyJson`, `save(Path)`, `load(Path)` |
| `ConfigException` | class extends `RuntimeException` | thrown by `ConfigManager.save`/`ProfileManager` on I/O failure |

### `dev.py54.corimlib.profile`

| Class | Kind | Key members |
|---|---|---|
| `ProfileManager` | class | `ProfileManager(Path directory)`, `listProfiles`, `saveCurrentAs`, `apply`, `delete`, `rename`, `duplicate`, `exportTo`, `importFrom` |

## `corimlib` module — Minecraft-dependent, zero loader-API dependency

### `dev.py54.corimlib`

| Class | Kind | Key members |
|---|---|---|
| `PlatformBridge` | interface | `loaderName`, `configDir`, `registerKeybind`, `onClientTick`, `onClientStarted`, `registerHudElement`, `onItemTooltip`, `onChatReceived`, `onChatObserved` |
| `Corim` | final class, static | `bridge()` — lazy `ServiceLoader`-resolved singleton |
| `CorimPaths` | final class, static | `configDir`, `configFile`, `profilesDir`, `profileExchangeDir` — **hardcoded to fixed filenames, not parameterized by mod id**, see [Config System](../core/config-system.md) |
| `CorimKeybinds` | final class, static | `CATEGORY` (namespaced `"crunch"`), `openMenu` |
| `HudRenderer` | functional interface | `render(GuiGraphicsExtractor, DeltaTracker)` |
| `TooltipHandler` | functional interface | `onTooltip(ItemStack, Item.TooltipContext, TooltipFlag, List<Component>)` |
| `ChatFilter` | functional interface | `test(Component) -> boolean` |

### `dev.py54.corimlib.hud`

| Class | Kind | Key members |
|---|---|---|
| `HudLineFeature` | abstract class | `feature`, `x/y/scale/customPosition` settings, abstract `text(Minecraft)` |
| `HudRegistry` | final class, static | `LINES`, `STACK_BASE_X/Y`, `STACK_LINE_HEIGHT`, `nextOrder`, `autoStackOrder`, `effectivePositions` |
| `HudOverlayElement` | final class implements `HudRenderer` | `editorOpen` (static), `render` |
| `FrameStats` | class | `onFrame()`, `fps()` |

### `dev.py54.corimlib.gui` — see [GUI & Theming](../gui/gui-and-theming.md) for the public/internal distinction

| Class | Visibility | Notes |
|---|---|---|
| `CrunchMainScreen` | `public class extends Screen` | `CrunchMainScreen()` |
| `CrunchFeatureSettingsScreen` | `public class extends Screen` | `CrunchFeatureSettingsScreen(Screen, Feature)` |
| `CrunchProfilesScreen` | `public class extends Screen` | `CrunchProfilesScreen(Screen)` |
| `CrunchHudEditorScreen` | `public class extends Screen` | `CrunchHudEditorScreen(Screen)` |
| `CrunchPerformanceMonitorScreen` | `public class extends Screen` | `CrunchPerformanceMonitorScreen(Screen)` |
| `CardRow` | `public record` | `CardRow(Feature, int x, int y, int width, int height)`, `isOverGear` |
| `CrunchTheme` | package-private | **not usable outside CorimLib's own package** |
| `CrunchButtonWidget` | package-private | **not usable outside CorimLib's own package** |
| `CrunchSliderWidget` | package-private | **not usable outside CorimLib's own package** |

### Mixins (`dev.py54.corimlib.mixin`)

Internal implementation detail — these back specific shipped features (fullbright's `LightmapRenderStateExtractorMixin`, infinite chat's `ChatHistoryMixin`, scrollable tooltips, zoom-scroll, screenshot organization, weather-effect toggling, the pause-screen menu button, and the version-specific `VanillaHudMixin` covering the `Gui`→`Hud` rename). None are part of a documented extension point — they exist to implement the shipped concrete features, not as hooks for third-party mods.

## Loader wrapper modules

`fabric`, `neoforge`, and `forge` (in the CorimLib repo) contain no meaningful public API of their own — each is a thin bundling wrapper (see [Architecture Overview](../core/architecture.md)). The real, loader-specific classes you'll write when depending on CorimLib — a `PlatformBridge` implementation and an entrypoint — live in *your own* mod's loader modules. Full real-code examples: [Fabric](../getting-started/fabric-setup.md), [NeoForge](../getting-started/neoforge-setup.md), [Forge](../getting-started/forge-setup.md).
