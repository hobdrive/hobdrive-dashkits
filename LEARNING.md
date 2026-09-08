# Agent Learning Notes

Lessons and corrections discovered while building **classic90** and **sup4**.
For a complete photo/screenshot workflow, start with
[Reference dashkit authoring](docs/REFERENCE_DASHKITS.md). The rules below distinguish
purely graphical items from intentionally visible digital readouts.

---

## 1. Always add `description="hidden"` to gauge `<item>` elements

**Mistake:** gauge `<item>` elements were created without `description="hidden"`.  
**Effect:** HobDrive renders a default sensor widget (name label, numeric value, unit) on top of every custom gauge face.  
**Fix:** add `description="hidden"` to every `<item>` that is rendering its own visual (image decorator, arrow decorator, etc.).

```xml
<!-- WRONG – default sensor label/value renders on top of the gauge -->
<item id="RPM" period="50" text-values="0:" .../>

<!-- CORRECT -->
<item id="RPM" period="50" description="hidden" text-values="0:" .../>
```

Apply this to every item whose default description is replaced by custom artwork,
not just the outer item. Dedicated text widgets may retain descriptions if the
reference calls for them.

---

## 2. Also add `text-values="0:"` to suppress numeric overlay

Even when `description="hidden"` is set, the numeric readout can still appear unless `text-values="0:"` (an empty/null mapping) is also provided.  
Use both attributes together on visual-only gauge items, with `units="none"` when
no unit should appear. Omit `text-values="0:"` on live numeric readouts:

```xml
<item id="Speed" description="hidden" text-values="0:" .../>
```

---

## 3. Theme-aware images: prefer `$Theme_brightness` substitution

**Mistake:** an attempt was made to write:

```xml
bg-image-path='$${ daynight == "day" ? "images/foo-light.svg" : "images/foo-dark.svg" }'
```

**Observed problem:** this expression did not select the expected theme artwork.
Do not generalize that into a ban on conditional expressions: sup4 uses them in
`color-map`. For image theme selection, avoid an unverified `daynight` variable and
use the established brightness substitution.

**Correct approach:** use the built-in `$Theme_brightness` variable, which expands to `"light"` or `"dark"` at load time, and embed it directly in the path string:

```xml
bg-image-path="images/classic90/tach-bg-$Theme_brightness.svg"
```

The SVG asset files must be named accordingly:

- `tach-bg-light.svg`  (bright face for day/light theme)
- `tach-bg-dark.svg`   (dim face for night/dark theme)

---

## 4. Dashkit theming and orientation support

Create dashkits with dark/light theme variations always. Only if user explicitly asks for a single theme should a one-theme dashkit be made.

Also, try to have dashkit for both portrait and landscape orientations, unless user explicitly asks for one orientation only.
Reuse a section where the same composition adapts well. Separate sections guarded
by `LandscapeLayout` and `PortraitLayout` are appropriate when the compositions
differ substantially. Preserve a deferred existing orientation and its assets.

## 5. Add `ignore-gauges="true"` to the `<section>` element

When a section renders only custom gauge images (no auto-generated gauge widgets are wanted), add `ignore-gauges="true"` to the `<section>` tag so HobDrive does not auto-inject default gauge elements.

```xml
<section name="Classic 90" ignore-gauges="true" ...>
```

---

## 6. Use `<union>` + `<item/>` background trick for full-bleed cells

To stretch a sub-grid to fill its parent cell completely, wrap it in a `<union>` with a bare `<item/>` as the first child.  
The `<item/>` occupies the full cell area and establishes the bounding box that the grid inside the union is stretched to fill.

```xml
<union>
  <item/>          <!-- full-cell placeholder -->
  <grid rows="50,50" cols="100">
    ...
  </grid>
</union>
```

---

## 7. Needle rotation formula for 270° sweep gauges

For gauges with a 270° sweep (−135° to +135°):

```
angle = (sensor_value - val_min) / (val_max - val_min) * 270 - 135
      = sensor_value * (270 / range) - (val_min * 270 / range + 135)
```

Worked examples used in classic90:

| Gauge       | Range        | Formula                                              |
|-------------|--------------|------------------------------------------------------|
| Speed       | 0–240 km/h   | `Sensor_Value * 1.125 - 135`                         |
| RPM         | 0–8000       | `Sensor_Value * 0.03375 - 135`                       |
| Fuel        | 0–100 %      | `Sensor_Value * 2.7 - 135`                           |
| CoolantTemp | 40–120 °C    | `(Max(Min(120; Sensor_Value); 40) - 40) * 3.375 - 135` |

Clamp values with `Max(Min(max; val); min)` wherever the scale has stops. Start
angle depends on the artwork's unrotated needle direction; sup4 uses a different
origin (`180°`) and range (`0–9000`). Keep the needle and fill clamps consistent.

---

## 8. `inherit` attribute for shared sensor aliases

Use `inherit="<alias_id>"` to pull in a sensor alias defined elsewhere (e.g. `_FuelLevelJoint` normalises different OBD fuel-level sensors to a 0–100 % scale):

```xml
<item id="FuelInTank" inherit="_FuelLevelJoint" period="60000" description="hidden" .../>
```

---

## 9. `period` attribute controls polling interval (milliseconds)

| Sensor type        | Recommended `period` |
|--------------------|----------------------|
| RPM / Speed        | 50 ms  (20 Hz)       |
| Temperature        | 10 000 ms (10 s)     |
| Fuel level         | 60 000 ms (1 min)    |

These are starting points, not a guarantee of the adapter's actual update rate.

---

## 10. Let the reference determine whether digital readouts are needed

Do not add unrelated readout rows to an intentionally pure analogue composition.
If the reference contains digital readings, reproduce them with live text widgets;
the reference itself establishes that requirement. A graphical dial can also use
`type="text"` with its default text suppressed, so widget type alone does not tell
whether the result is intended to be analogue or digital.


## 11. Decorators

Please don't use the legacy way of defining decorators with "decorator-something".
Use inner xml elements instead, like in the official dashkits. It's more robust and easier to maintain.

Notice that decorators are stacked. Thats a powerful feature.

## 12. sup4 lessons to carry into the next reference project

- Measure curved alignments as well as the central dial. A separator belongs in the
  gap between labels and values; checking only numeric centers misses text overlap.
- Fit all artwork and live values in one coordinate system. The `0.98` width factor
  in sup4 accounts for its renderer margin; verify it before reusing it elsewhere.
- Keep the host wallpaper visible around the instruments. An opaque panel rectangle
  breaks theme integration; uniform transparency still leaves its boundary. Use a
  local gradient fading to zero alpha at every edge, and check both wallpaper modes.
- Keep the zero-padding wrapper between rotating filters and crops in sup4: the
  current crop implementation otherwise bypasses the target's transformation.
- SVGs are rasterized at 4× per axis by the current MAUI image provider. Choose
  logical dimensions deliberately and avoid unnecessary full-screen layers.
- Use explicit `type="text"` for compact missing-sensor placeholders and
  `units="none"` to hide units. Validate real sensor IDs and converted units.
- An `override-*` directory symlink worked for live source editing in Catalyst.
  Reload the UI after edits and wait for correlated Debug action completion.
- In-memory injection does not disable simulator formulas or change alias targets.
  Inspect the active profile when deterministic values fail to reach the dashboard.
- Capture both themes and multiple viewport shapes and scale positions. Restore
  temporary profile changes, stop the app, and deliver real app previews.

Examples, commands, and validation criteria are maintained in the
[reference workflow](docs/REFERENCE_DASHKITS.md) and the
[sup4 maintenance notes](community/sup4/README.md).
