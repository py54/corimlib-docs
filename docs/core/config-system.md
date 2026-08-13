# Config System

## `ConfigManager`

`dev.py54.crunch.config.ConfigManager` (in `common`, package `dev.py54.crunch.config`) serializes the *entire* `FeatureRegistry` — every registered feature's enabled state and every setting's value, plus `Favorites`' favorite/recent lists — to and from a single JSON file via Gson.

```java
public final class ConfigManager {
    public static JsonObject toJson();
    public static void applyJson(JsonObject root);
    public static void save(Path path);
    public static void load(Path path); // corrupt files are backed up, not crashed on
}
```

`save`/`load` take a `Path` you supply — `ConfigManager` itself is loader-agnostic and doesn't know where your config file should live. That's [`PlatformBridge.configDir()`](platform-bridge.md)'s job:

```java
Path configFile = Corim.bridge().configDir().resolve("mymod.json");
ConfigManager.save(configFile);
ConfigManager.load(configFile);
```

### Corrupt-file handling

If the JSON at `path` fails to parse, `load()` does **not** throw or crash — it copies the bad file to `<name>.corrupt-<epoch>.bak` next to it (best-effort; a failed backup is silently ignored) and falls through to defaults, leaving every `Feature`/`Setting` at whatever value it already had (its constructor default, if this is a fresh boot). Malformed individual setting values are handled the same defensive way — a bad value for one setting is skipped, not fatal to the whole load.

### When to call `load()`

!!! danger "Not at mod-init time"
    Loading the config fires every `Feature`'s `onToggle` and every `Setting`'s `onChange` handler *synchronously*, and several real handlers touch `Minecraft.getInstance().options`, which doesn't exist yet at plain mod-init time. Call `ConfigManager.load(...)` from inside `Corim.bridge().onClientStarted(...)`, not directly in your entrypoint. This is a real bug CorimLib hit and fixed — see [`PlatformBridge`](platform-bridge.md) and [Limitations](../reference/limitations.md).

### Real bootstrap shape

This is (trimmed) exactly how Crunch wires save/load — copy this shape for your own mod:

```java
Corim.bridge().onClientStarted(mc -> {
    ConfigManager.load(CorimPaths.configFile());
    LOGGER.info("MyMod initialized ({}): {} features registered",
            Corim.bridge().loaderName(), FeatureRegistry.all().size());
});

Runtime.getRuntime().addShutdownHook(new Thread(() -> ConfigManager.save(CorimPaths.configFile())));
```

## `CorimPaths`

```java
public final class CorimPaths {
    public static Path configDir();            // Corim.bridge().configDir()
    public static Path configFile();            // configDir().resolve("crunch.json")
    public static Path profilesDir();            // configDir().resolve("profiles")
    public static Path profileExchangeDir();      // configDir().resolve("profile-exchange")
}
```

!!! warning "`CorimPaths` is hardcoded to Crunch's own filenames"
    `configFile()` resolves to `crunch.json` unconditionally — it is not parameterized by mod id. If you're building your own mod on CorimLib (not just adding features that show up inside Crunch's own menu), don't reuse `CorimPaths` directly; call `Corim.bridge().configDir().resolve("yourmod.json")` yourself instead. See [Limitations](../reference/limitations.md).

## `ProfileManager`

Named, full-registry snapshots — save/apply/rename/duplicate/delete/import/export — each stored as one JSON file (reusing `ConfigManager`'s same JSON shape) in a directory you supply:

```java
public final class ProfileManager {
    public ProfileManager(Path directory);

    public List<String> listProfiles();
    public void saveCurrentAs(String name);
    public boolean apply(String name);
    public void delete(String name);
    public boolean rename(String from, String to);
    public boolean duplicate(String from, String to);
    public boolean exportTo(String name, Path destination);
    public boolean importFrom(Path source, String asName);
}
```

Profile names are sanitized (`[^a-zA-Z0-9 _-]` stripped) before being used as filenames. Construct one pointing at `CorimPaths.profilesDir()` (or your own equivalent directory) and it's ready to use — this is exactly what powers `CrunchProfilesScreen` (see [GUI & Theming](../gui/gui-and-theming.md)).

## `Favorites`

Tracks favorited and recently-opened feature ids — a small static helper, persisted automatically as part of `ConfigManager`'s JSON (`favorites`/`recent` arrays), not something you call `save`/`load` on separately:

```java
public final class Favorites {
    public static boolean isFavorite(String featureId);
    public static void toggleFavorite(String featureId);
    public static Set<String> favoriteIds();
    public static void markRecentlyUsed(String featureId);   // caps at 8, most-recent-first
    public static List<String> recentIds();
    public static void setFavoriteIds(Iterable<String> ids);
    public static void setRecentIds(Iterable<String> ids);
}
```

Like `FeatureRegistry`, this is a single global static — favorites aren't namespaced per mod either.
