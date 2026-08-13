# GUI & Theming

CorimLib ships a complete, working dark-modal-card GUI as five real `Screen` classes — the same one Crunch's own menu uses. Before using it, it's important to know **which parts are actually public API and which parts are internal implementation** — this page is deliberately precise about that distinction, because it's easy to assume otherwise from the class names.

## The five screens (public, reusable as-is)

All in package `dev.py54.crunch.corimlib.gui`, all `public class ... extends Screen` with real public constructors:

| Class | Constructor | What it shows |
|---|---|---|
| `CrunchMainScreen` | `new CrunchMainScreen()` | The main menu: search, sidebar (All/Favorites/every `Category`), a scrollable feature card grid reading from `FeatureRegistry`. |
| `CrunchFeatureSettingsScreen` | `new CrunchFeatureSettingsScreen(Screen parent, Feature feature)` | Per-feature settings, generic over any `Setting<?>` — boolean/int/decimal/enum/color/keybind/text widgets. |
| `CrunchProfilesScreen` | `new CrunchProfilesScreen(Screen parent)` | Save/apply/duplicate/delete + import, backed by `ProfileManager`. |
| `CrunchHudEditorScreen` | `new CrunchHudEditorScreen(Screen parent)` | Drag `HudLineFeature`s to reposition; suppresses the live overlay while open. |
| `CrunchPerformanceMonitorScreen` | `new CrunchPerformanceMonitorScreen(Screen parent)` | FPS/frame time/memory plus a rolling FPS graph from `FrameStats`. |

Opening the menu is one line, exactly as CorimLib's own bootstrap does it:

```java
if (mc.canInterruptScreen()) {
    mc.setScreenAndShow(new CrunchMainScreen());
}
```

Because `CrunchMainScreen` reads directly from the global `FeatureRegistry`, any `Feature` your mod registers automatically appears in it — you get a complete settings UI for free, without building your own screen, *as long as you're comfortable with the UI being titled and branded "Crunch"* (see below).

!!! warning "These screens are literally Crunch's own UI, not a generic white-label menu"
    `CrunchMainScreen`'s constructor sets its title to the hardcoded string `Component.literal("Crunch")`, and its translation keys live under the `crunch.gui.*` namespace in CorimLib's own `assets/crunch/lang/en_us.json`. There's no constructor parameter or theme hook to relabel it. If you use these screens as-is, your feature toggles will appear inside a menu titled "Crunch" — appropriate if you're building a Crunch add-on, less appropriate if you want your own mod's distinct branding. There is currently no supported way to reuse the *layout* with your own title/branding without copying/forking the screen source.

## `CrunchTheme` and the custom widgets — package-private, not part of the public API

This is the important, easy-to-miss finding this page exists to flag: `CrunchTheme`, `CrunchButtonWidget`, and `CrunchSliderWidget` are all declared **without** the `public` modifier:

```java
final class CrunchTheme { /* package-private */ }
final class CrunchButtonWidget { /* package-private */ }
final class CrunchSliderWidget { /* package-private */ }
```

They live in `dev.py54.crunch.corimlib.gui` — a package your own mod's code is not part of — so `CrunchTheme.panel(...)`, `CrunchTheme.card(...)`, `new CrunchButtonWidget(...)`, etc. **will not compile** from outside CorimLib itself, regardless of what version of the jar you're compiling against. This is why the five screens above are documented as "use as-is," not "use as a component toolkit" — the dark chamfered-corner panel look, the custom-drawn buttons, and the custom-drawn sliders exist, and they're exactly what gives Crunch's real menu its visual identity, but they are internal implementation detail today, not a published widget library.

If you want your own screen to visually match, your only real option right now is reading the actual source for reference and reimplementing the same drawing calls in your own package — `CrunchTheme`'s real methods (`panel`, `card`, `chamferedRect`, `backdrop`, `diamond`, `gearIcon`, `isOver`) are all built from plain `GuiGraphicsExtractor.fill(...)` calls, since 26.2's GUI API has no native rounded-rect fill.

## `CardRow` — public, but tightly coupled

```java
public record CardRow(Feature feature, int x, int y, int width, int height) {
    public boolean isOverGear(double mouseX, double mouseY);
    // ...
}
```

`CardRow` *is* public, unlike the theme/widget classes above — but it's a record specifically shaped around laying out one `Feature`'s card (with its star/gear hit-testing baked in), not a general-purpose layout primitive. It's realistically only useful if you're building something that mimics `CrunchMainScreen`'s exact card-grid layout.

## What this means in practice

| You want to... | Can you do it today? |
|---|---|
| Open the exact Crunch-branded settings menu, showing your own registered features | **Yes** — `new CrunchMainScreen()`, no extra work. |
| Open per-feature settings, profiles, HUD editor, or the performance monitor for your own features | **Yes** — all five constructors are public and take real, usable arguments. |
| Draw your own screen with the same dark chamfered-corner look, under your own title/branding | **No, not by calling CorimLib** — `CrunchTheme` and the widget classes are package-private. You'd need to reimplement the drawing calls yourself. |
| Reuse `CardRow` for an unrelated card-grid UI | Technically public, but narrow — built specifically for `Feature` cards. |

See [Limitations & Experimental APIs](../reference/limitations.md) for how this fits into CorimLib's broader "extracted mid-project from Crunch" history.
