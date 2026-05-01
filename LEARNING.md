# Agent Learning Notes

Lessons and corrections discovered while building the **classic90** dashkit.
Intended to be folded into official documentation.

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

This applies to **all** items inside a gauge layout (speedometer, tachometer, fuel, temperature, etc.), not just the "outer" ones.

---

## 2. Also add `text-values="0:"` to suppress numeric overlay

Even when `description="hidden"` is set, the numeric readout can still appear unless `text-values="0:"` (an empty/null mapping) is also provided.  
Use both attributes together on every gauge item:

```xml
<item id="Speed" description="hidden" text-values="0:" .../>
```

---

## 3. Theme-aware images: use `$Theme_brightness`, not a conditional expression

**Mistake:** an attempt was made to write:

```xml
bg-image-path='$${ daynight == "day" ? "images/foo-light.svg" : "images/foo-dark.svg" }'
```

**Problem:** the ternary conditional expression syntax is not supported (or unreliable) in image-path attributes.  
**Correct approach:** use the built-in `$Theme_brightness` variable, which expands to `"light"` or `"dark"` at load time, and embed it directly in the path string:

```xml
bg-image-path="images/classic90/tach-bg-$Theme_brightness.svg"
```

The SVG asset files must be named accordingly:
- `tach-bg-light.svg`  (bright face for day/light theme)
- `tach-bg-dark.svg`   (dim face for night/dark theme)

---

## 4. Add `ignore-gauges="true"` to the `<section>` element

When a section renders only custom gauge images (no auto-generated gauge widgets are wanted), add `ignore-gauges="true"` to the `<section>` tag so HobDrive does not auto-inject default gauge elements.

```xml
<section name="Classic 90" ignore-gauges="true" ...>
```

---

## 5. Use `<union>` + `<item/>` background trick for full-bleed cells

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

## 6. Needle rotation formula for 270° sweep gauges

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

Note: clamp temperature with `Max(Min(max; val); min)` to avoid needle going past the stops.

---

## 7. `inherit` attribute for shared sensor aliases

Use `inherit="<alias_id>"` to pull in a sensor alias defined elsewhere (e.g. `_FuelLevelJoint` normalises different OBD fuel-level sensors to a 0–100 % scale):

```xml
<item id="FuelInTank" inherit="_FuelLevelJoint" period="60000" description="hidden" .../>
```

---

## 8. `period` attribute controls polling interval (milliseconds)

| Sensor type        | Recommended `period` |
|--------------------|----------------------|
| RPM / Speed        | 50 ms  (20 Hz)       |
| Temperature        | 10 000 ms (10 s)     |
| Fuel level         | 60 000 ms (1 min)    |

---

## 9. Do not add textual readout rows in pure gauge dashkits

Including `<item type="text" ...>` rows for a dashkit that is meant to be purely graphical clutters the layout and may break the intended visual. Omit text rows entirely when all information is conveyed through gauge faces and needles.
