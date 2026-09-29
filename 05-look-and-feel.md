# FlatLaf look and feel

Part of the [STOV GUI tree performance handover](../gui-tree-performance-handover.md).

---

## 11. FlatLaf look and feel

Numbered 11 to stay clear of the sections carried over from the combined
document (1–10). This work is not a performance change; it landed alongside the
tree work because both touch what the user sees on startup.

### 11.1 Is it safe?

Yes, with the caveats below. FlatLaf is a normal Swing look and feel: it
implements `LookAndFeel` and is installed through `UIManager.setLookAndFeel`.
Nothing in the application compiles against it — the class is named as a string
and resolved at runtime — so the only build coupling is one jar on the
classpath.

- **Java 8.** FlatLaf 3.x still targets Java 8, so version **3.7.2** was chosen
  deliberately rather than the latest. Later majors may not stay on 8; check
  before bumping.
- **It cannot brick the client.** `setLookAndFeel` is wrapped so a missing jar,
  an incompatible class file or a broken theme falls back to `SUN_WINDOWS`
  rather than aborting startup. The catch covers `LinkageError` as well as
  `Exception`, because a class compiled for a newer bytecode level throws
  `UnsupportedClassVersionError`, which is an `Error` and would otherwise
  escape.
- **It is reversible at runtime.** `stov.laf` selects the look and feel, so a
  user or a support engineer can put the old Windows appearance back by editing
  a property. This is why none of the fixes below touch the dialog layouts —
  see §11.5.

### 11.2 How it is wired

The jar was fetched with the dependency plugin rather than adding a Maven build
to an Ant project:

```
mvn org.apache.maven.plugins:maven-dependency-plugin:3.6.1:get \
    -Dartifact=com.formdev:flatlaf:3.7.2 -Dtransitive=false
```

and copied to `lib/flatlaf-3.7.2/flatlaf-3.7.2.jar`. Four places then reference
it, and **all four are required** — missing the `.classpath` entry is what
produced the `ClassNotFoundException: com.formdev.flatlaf.FlatLightLaf` seen on
the first attempt, because Eclipse builds and runs from its own classpath rather
than from `build.xml`.

| File | Entry |
|---|---|
| `build.properties` | `jar.flatlaf=flatlaf-3.7.2/flatlaf-3.7.2.jar` |
| `build.xml` | `<file name="${jar.flatlaf}"/>` in the `app.jar` filelist |
| `.classpath` | `<classpathentry kind="lib" path="lib/flatlaf-3.7.2/flatlaf-3.7.2.jar"/>` |
| `application.properties` (`src/conf`, `src/envs/base`, `src/envs/dist`) | `stov.laf = FLATLAF_INTELLIJ` |

`LookAndFeelTransformer` gained six constants and `StovLookAndFeelTransformer`'s
`lafMap` six matching keys:

| `stov.laf` value | Class |
|---|---|
| `FLATLAF_LIGHT` | `com.formdev.flatlaf.FlatLightLaf` |
| `FLATLAF_INTELLIJ` | `com.formdev.flatlaf.FlatIntelliJLaf` |
| `FLATLAF_MAC_LIGHT` | `com.formdev.flatlaf.themes.FlatMacLightLaf` |
| `FLATLAF_DARK` | `com.formdev.flatlaf.FlatDarkLaf` |
| `FLATLAF_DARCULA` | `com.formdev.flatlaf.FlatDarculaLaf` |
| `FLATLAF_MAC_DARK` | `com.formdev.flatlaf.themes.FlatMacDarkLaf` |

An unknown key logs an error and falls back to `SUN_WINDOWS` rather than
throwing.

### 11.3 Three adjustments, and why each was needed

FlatLaf out of the box does not drop cleanly into layouts measured against the
Windows look and feel. Three problems appeared, each fixed in
`StovLookAndFeelTransformer` rather than in the affected forms.

**Everything looked white and borderless.** FlatLaf derives border colours from
the background (`Component.borderColor = shade(@background, 20%)`) and ships
with table grid lines and tab separators switched off. On a white panel that
20% shade is nearly invisible. `applyFlatLafBorders()` recomputes the border,
disabled-border, separator and grid colours from the *actual* `Panel.background`
and `Table.background`, and re-enables `Table.showHorizontalLines`,
`Table.showVerticalLines` and `TabbedPane.showTabSeparators`. It shades on light
themes and tints on dark ones, choosing by perceived brightness, so the dark
themes are not left with black-on-black borders.

**Text fields clipped their descenders.** The forms pin field heights in
pixels — the login dialog forces 21 — and those numbers were measured against
the Windows look and feel. The IntelliJ theme spends 10 of those 21 pixels on
decoration: `Component.focusWidth = 2` top and bottom, a one-pixel line each
side, and a `2,6,2,6` margin. That leaves about 11 pixels for a font whose line
box needs about 15. `applyFlatLafMetrics()` sets `Component.focusWidth` and
`Component.innerFocusWidth` to 0 and trims the vertical margin to `1,6,1,6`
across the text, combo and spinner keys. The horizontal padding is left alone;
it costs nothing in these layouts.

**Dialogs grew a tall empty band.** `AbstractDialogComponentBuilder.addValidationView`
reserves `top:80dlu` for the validation report. JGoodies dialog units are derived
from the font metrics of the installed look and feel, so a larger default font
silently inflates every dlu-sized row. `applySystemDefaultFont()` reads the
`win.messagebox.font` desktop property and pins FlatLaf's `defaultFont` to it,
which is the same font the Windows look and feel used, so the rows keep their
previous pixel height.

Ordering matters: **`applySystemDefaultFont()` must run before
`setLookAndFeel`**, because FlatLaf reads `defaultFont` while building its
defaults table. The border and metric overrides must run *after*, because
installing a look and feel replaces the defaults table and would discard them.
`init()` is sequenced accordingly.

### 11.4 What was deliberately not done

The dialog builders (`LoginDialogComponentBuilder`, `ChangePasswordBuilder`) do
call `setPreferredSize(new Dimension(10, 21))` on their fields, and removing
those hardcoded heights was the obvious fix for the clipping. It was rejected:
`stov.laf` can be switched back to `SUN_WINDOWS` or a JGoodies theme at any
time, and those layouts have to keep working unchanged under the old metrics.
Adjusting what FlatLaf consumes is reversible; editing the forms is not.

### 11.5 Reverting

Set `stov.laf = SUN_WINDOWS` in `application.properties`. All three adjustment
methods are guarded by `flatLafActive`, so they do not run for a non-FlatLaf
look and feel and no `UIManager` key is left overridden. The jar can stay on the
classpath unused.
