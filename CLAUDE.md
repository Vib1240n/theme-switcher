# theme-switcher

Unified theming system for the user's Hyprland desktop. One command swaps colors and visual style across every GTK4 app, kitty, hyprland, hyprlock, and waybar.

## What this is

A Python + Jinja2 renderer that reads:

1. A **palette** TOML (mocha, tokyonight, etc.) — pure colors, mode flag (dark/light)
2. A **visual** TOML (apple, material, i3-flat, etc.) — radii, padding, font sizes, opacities (split by mode)

...and renders **app-specific** config files for every app it manages, then sends each app its reload signal. State (current palette + visual) is persisted to `state.toml`.

## Project layout

```
~/Development/theme-switcher/
├── CLAUDE.md                     # this file
├── README.md                     # user-facing usage
├── theme-apply.py                # renderer + dispatcher
├── state.toml                    # auto-written: current palette + visual
├── palettes/<name>.toml          # color-only definitions
├── visuals/<name>.toml           # radii, sizes, opacities (dark+light blocks)
└── templates/<app>.<ext>.j2      # Jinja2 templates per output file
```

## How the renderer works

`theme-apply.py` is a single script. On invocation:

1. Parse CLI flags (`--palette`, `--visual`, `--list-*`, `--current`, `--random`).
2. Load palette TOML + visual TOML.
3. **Flatten the visual's opacity**: pick `opacity.dark` or `opacity.light` based on `palette.meta.dark`, write back as `opacity.*` so templates only reference `{{ opacity.bg }}`, never `{{ opacity.dark.bg }}`.
4. Backup all destination files to `~/Development/Logs/theme-switcher-backups/<timestamp>/` (mirrors the source paths).
5. For each app in the dispatch table:
   - Render its template with `{ color, opacity, radius, sizing, palette, visual }` context.
   - Write to the app's output path.
   - Run the reload command (subprocess, non-blocking, errors logged not fatal).
6. Save state to `state.toml`.

## Dispatch table

| App           | Template                         | Output path                              | Reload command                       |
| ------------- | -------------------------------- | ---------------------------------------- | ------------------------------------ |
| ags           | `templates/ags.css.j2`           | `~/.config/ags/style.css`                | `pkill ags; ~/.config/ags/run-bar.sh`|
| rust-widgets  | `templates/rw.css.j2`            | `~/.config/rw/style.css`                 | `rw reload`                          |
| cliphist-gui  | `templates/cliphist-gui.css.j2`  | `~/.config/cliphist-gui/style.css`       | `cliphist-gui --reload`              |
| launch-gui    | `templates/launch-gui.css.j2`    | `~/.config/launch-gui/style.css`         | `launch-gui --reload`                |
| kitty         | `templates/kitty.conf.j2`        | `~/.config/kitty/colors.conf`            | `kill -SIGUSR1 $(pgrep -x kitty)`    |
| hyprland      | `templates/hypr-colors.conf.j2`  | `~/.config/hypr/colors.conf`             | `hyprctl reload`                     |
| hyprlock      | `templates/hyprlock-colors.conf.j2` | `~/.config/hypr/hyprlock-colors.conf` | (none - read at lock time)           |
| waybar        | `templates/waybar.css.j2`        | `~/.config/waybar/colors.css`            | `pkill -SIGUSR2 waybar`              |

## Wiring (one-time setup by user)

The script renders files; it does **not** edit the user's main configs. The user adds these include lines manually:

- `~/.config/kitty/kitty.conf` -> `include colors.conf`
- `~/.config/hypr/hyprland.conf` -> `source = ~/.config/hypr/colors.conf`
- `~/.config/hypr/hyprlock.conf` -> `source = ~/.config/hypr/hyprlock-colors.conf`
- `~/.config/waybar/style.css` -> `@import "colors.css";` (at top)

GTK4 apps (ags, rw, cliphist-gui, launch-gui) read `~/.config/<app>/style.css` directly, no wiring.

## Schema

### Palette (`palettes/<name>.toml`)

```toml
[meta]
name = "catppuccin-mocha"   # display name
dark = true                  # selects opacity.dark vs opacity.light

[color]
# Surfaces
bg              # main container
bg_elevated     # inputs, sections, icon backgrounds
bg_critical     # destructive/critical card bg

# Foregrounds
fg              # primary text
fg_pure         # highest-contrast text (notification headers)

# Structure
border
shadow          # box-shadow color (usually #000)

# Accent (main brand color)
accent

# Semantic
red             # destructive, error
yellow          # warning
green           # success
blue            # info

# Vim mode indicators (cliphist, launch)
vim_normal
vim_insert
vim_bg
```

### Visual (`visuals/<name>.toml`)

```toml
[meta]
name = "apple"

[radius]
container container_pill section card row input icon badge small circle

[sizing]
border_width font_base font_bar font_small font_tiny font_section icon_box
font_family font_family_mono font_family_bar

[opacity.dark]
bg bg_container bg_widget bg_popup border bg_elevated input section
hover selected fg_strong fg_normal fg_muted fg_dim fg_faint shadow

[opacity.light]
# same keys, tuned for light palettes (higher container opacity,
# lower fg_dim/fg_faint to maintain legibility)
```

After the renderer flattens, templates reference `{{ opacity.bg }}` etc. — no `.dark` or `.light` prefix.

## Conventions

- **All colors as hex strings**, GTK4 `alpha(#hex, 0.55)` syntax in templates (not `rgba()`).
- **All radii are integers** (no `px` in TOML). Template appends `px`. `circle = 50` is the exception — used as `border-radius: 50%`.
- **opacities are floats 0.0–1.0**.
- **No emojis in code, output, or logs.** Ever.
- **All Python in `~/Development/python_scripts/`** convention does NOT apply to this project's `theme-apply.py` — it lives in the repo root because it's the entrypoint.
- **Concise variable names** (i, j, k, v, ctx). Standard for this user.

## Adding a new palette

Drop `palettes/<name>.toml` with all 16 color keys and `meta.dark`. Done. `theme-apply --list-palettes` picks it up automatically (glob).

## Adding a new visual

Drop `visuals/<name>.toml` with all radius/sizing/opacity keys. Both `opacity.dark` and `opacity.light` blocks must be present.

## Adding a new app

1. Write `templates/<app>.<ext>.j2`.
2. Add an entry to the `APPS` dispatch table in `theme-apply.py` (output path + reload command).
3. Test with `--dry-run` first.

## Backup behavior

Before any write, every destination file (if it exists) is copied to:
`~/Development/Logs/theme-switcher-backups/<YYYY-MM-DD_HHMMSS>/<original-relative-path>`

So `~/.config/cliphist-gui/style.css` -> `~/Development/Logs/theme-switcher-backups/2026-05-16_123045/.config/cliphist-gui/style.css`.

This runs on EVERY invocation, not just first run. Disk is cheap; reverting is priceless.

## Debugging

- `theme-apply --palette X --visual Y --dry-run` -> renders to stdout, writes nothing, runs no reloads.
- `theme-apply --verbose` -> prints every read/write/reload + timing.
- Renders that fail Jinja errors -> log to stderr, that one app is skipped, others continue.
- `state.toml` shows the last successful apply (palette + visual + timestamp).

## What this project is NOT

- Not a hot-reload daemon. One-shot CLI tool. Re-run to re-apply.
- Not a GUI (yet — planned as a separate rust-widget `theme-switcher` later).
- Not a wallpaper manager. Doesn't touch swww/awww.
- Not pywal. Doesn't extract colors from images. Palettes are hand-authored.

## Known issues / TODO

- cliphist-gui / launch-gui `--theme <name>` flag is ephemeral (overridden on reload by config). theme-switcher bypasses this by writing directly to the apps' style.css, so it doesn't matter for our purposes — but the underlying rust bug should be fixed upstream eventually.
- rust-widgets notification-center module historically had its own palette (whites + tailwind semantics). Now unified with the main theme. May look different from pre-theme-switcher state — by design.
- Hyprlock has no live reload; new colors apply on next lock.
