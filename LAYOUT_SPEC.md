# HobDrive Layout Specification

This document collects the layout tags, attributes and options used by HobDrive UI layout files.

Purpose: provide a concise, practical reference for creating and modifying UI layouts (dash panels, ECU screens, hybrid/GPS panels, etc.)

## Contract
- Inputs: XML-like layout files (tags: `ui`, `section`, `grid`, `item`, `union`, `stack-vertical`, `stack-horizontal`, etc.)
- Output: Rendered UI screens in HobDrive; conditional visibility and decorators influence runtime behavior
- Error modes: invalid attributes are ignored or fallback to defaults; missing sensors show as empty/unavailable
- Success criteria: layout renders and respects conditional `if` attributes, grid placement, decorators and widget attributes

## Top-level tags
- `ui` — root element for a layout file. Contains `section` elements and other layout constructs.

- `include` — allows embedding another layout file or fragment. Attributes:
  - `file` — relative path to the included layout (e.g., `file="default-tripcomp.layout"`).
  - `if` — optional condition which controls whether the included file is processed (same expression syntax as `section` `if`).
  - Includes are evaluated when the file is loaded and are commonly used to compose main layouts from reusable fragments (trip computer, HUD, dashkits).

Example:
```xml
<include file="default-tripcomp.layout" if='${!sexists("GasTrigger")}'/>
```

## section
Defines a logical screen/page block. Common attributes:
- name — human-readable section name (used in UI lists).
- fixed — `true/false` (whether the section is fixed in navigation).
- if — conditional expression (a sensor name or expression). Section is shown only if condition evaluates truthy. Examples: `if="Oxygen_b1s1"`, `if="PortraitLayout"`, `if="Renault_01_11103.ControlModuleVoltage"`.
- class — CSS-like class used by system to apply predefined behaviors (e.g., `SensorList`).
- ns — namespace for sensor filtering when using `SensorList`.
- exclude — regex to exclude sensor IDs in generated sensor lists
- maximize-focus — (from changelog) `true/false` remove optional controls when focused
- controls — `autohide` or other values to control UI buttons

Example:
<section name="GPS">
  ...
</section>

<section name="Fuel Trims" if="STFT1">
  ...
</section>

`config` elements
- `config` — declare UI-configurable options that appear in settings dialogs. Typical attributes:
  - `id` — unique identifier accessible from layout expressions (e.g., `${Layout_ECUFuel}`).
  - `type` — `bool`, `int`, `string`, etc.
  - `default` — default value.
  - `title`, `descr` — text shown in UI; localized variants like `title-ru`, `descr-ru` are supported.

Example:
```xml
<config id="Layout_ECUFuel" type="bool" default="false"
        title="Use Fuel Level data from ECU"
        descr="If enabled - fuel level will be shown from ECU. Otherwise - estimated from fuel consumption"
        title-ru="Показывать уровень топлива из ЭБУ"
        descr-ru="Если включено - уровень топлива будет браться из ЭБУ. Иначе - показывается оценочный уровень топлива"/>
```

`config` elements
- `config` — declare UI-configurable options that appear in settings dialogs. Typical attributes:
  - `id` — unique identifier accessible from layout expressions (e.g., `${Layout_ECUFuel}`).
  - `type` — `bool`, `int`, `string`, etc.
  - `default` — default value.
  - `title`, `descr` — text shown in UI; localized variants like `title-ru`, `descr-ru` are supported.

Example:
```xml
<config id="Layout_ECUFuel" type="bool" default="false"
        title="Use Fuel Level data from ECU"
        descr="If enabled - fuel level will be shown from ECU. Otherwise - estimated from fuel consumption"
        title-ru="Показывать уровень топлива из ЭБУ"
        descr-ru="Если включено - уровень топлива будет браться из ЭБУ. Иначе - показывается оценочный уровень топлива"/>
```

## Layout containers
- `grid` — places child `item`s into a table-like grid.
  - Attributes: rows, cols — comma-separated size hints. Example: `rows=",," cols=",35"`.
  - rows and cols can include numeric heights/widths or empty entries for flexible sizing.
  - grids can be nested.

- `stack-vertical` / `stack-horizontal` — stack items vertically/horizontally; `columns` may be used.
  - Example: `<stack-vertical columns="${LandscapeLayout ? 2 : 1}">`

- `union` — allows compositing several alternate layouts into same area (used in TPMS layout).
  - Use `union` when multiple mutually-exclusive sub-layouts should occupy the same grid area; each child typically contains its own `grid`.

 - `optional` / `volatile` (section attributes) — lightweight hints used by some default layouts: `optional="true"` marks screens that can be hidden, `volatile` marks temporary screens.

- `switch` — selects one of its child elements to show based on an `index` expression. Useful for small cycling lists or conditional items.
  - Attributes:
    - `index` — an expression or integer selecting which child to show (0-based). Can reference sensor values via `svalue(...)` or any TinyExe expression.
    - `interval` — when present, `index` may be evaluated periodically using this interval (milliseconds) to cycle children.
    - `if` — optional condition to entirely enable the switch.
  - Children are typical `item`s (or nested grids); when `index` resolves to a child index that child is displayed.

Example (cycle temperature sensors every 4 seconds):
```xml
<switch if='${Layout_RollTemp = true}' index='$${ svalue("DateTime") / 4000 }'>
  <item id="ATFTemp" .../>
  <item id="CVTTemp" .../>
  <item id="EngineOilTemp" .../>
</switch>
```

## item
Represents a widget in a grid or stack. Common attributes and usages:
- id — sensor or special control identifier. Examples: `RPM`, `Latitude`, `ControlModuleVoltage`.
- type — `button`, `static`, etc. Defaults to a sensor/value widget when omitted.
- text — for buttons or static items, visible text.
- action — for `button`: action string such as `invoke(FAN_ON)` or `run(...)`.
- size — visual size hint: `micro`, `small`, `normal`, `large`, `huge`, `extralarge` or numeric.
- precision — number of decimal digits or special tags like `6fixed`.
- inherit — refers to a decorator or style name, e.g., `inherit="_ColoredOnOff"` or `_MB_Battery`.
- colspan / rowspan — span cells in the grid.
- description — extra attribute used to style or position small descriptive labels (e.g., `description="aside"`).
- period — update period in milliseconds (e.g., `period='5000'` on `ControlModuleVoltage`).
- wrap — filter expression for smoothing or prediction: examples from changelog:
  - `wrap="predict(50,0.7)"`
  - `wrap="smooth(400,100,5)"`
  - `wrap="average(30)"`
- custom attributes: `text-values`, `custom-units`, `colored-descr`, `text-evaluator` (see changelog entries where used)

- onclick — item-level click handler (navigates or runs named actions). Examples: `onclick="NewFueling"`, `onclick="Go(Efficiency)"`.
- actions — comma-separated list of actions available in sensor info dialog. Example: `actions="SetupOdometer"`.
- units — visual placement of unit text: `units="below"`, `units="aside"` or `units="hidden"`.
- interval — an update interval used for dynamic UI behaviour or internal timers (different from sensor `period`).

Example usages found in `default-tripcomp.layout`:
```xml
<item id="FuelLevel" if='${Layout_ECUFuel != true}' inherit="_FuelIndicator, _EditAction" onclick="NewFueling"/>
<item id="Odometer" if="${Layout_ECU_Odometer != true}" actions="SetupOdometer" size="large" inherit="_EditAction"/>
```

- type specifics and chart attributes:
  - `type="chart"` — renders a chart for the sensor; common chart attributes: `chart='line'`, `time-scale`, `min-limit`, `max-limit`, `cover-description`.
  - `border-opacity` — visual border control (used on charts/gauges in dashkits).
  - `custom-description` / `description` / `custom-units` — allow custom labels and units.

- item extradata access:
  - `id="sensor[extradata]"` — render additional data stored with a sensor (e.g., `TripTime[fillup]`).

Examples:
<item id='Latitude' precision='6fixed' size="normal"/>
<item type="button" text="Fan On" action="invoke(FAN_ON)" size='extralarge' />
<item id='Lambda' colspan="2"/>

## Special item types
- type="button" — clickable button. Use `action`="invoke(NAME)" to call ECU/service action.
- type="static" — static text block. E.g. `type="static" size="micro"`
- gauge / chart / image decorators — declared either as `item` with nested decorators or in gauge/theme files; can accept decorators like `image`, and attributes `image-path`, `image-scale`, `image-width`, `image-height`, `image-rotate`.

## Decorators and nested elements
- `item` may contain nested decorators (per changelog). Example syntax:
<item id="Speed">
  <image image-zorder="bottom" image-path="images/dashboard-svgrepo-com.svg" image-scale="0.5"/>
  <image image-zorder="bottom" image-path="images/arrow.png" image-scale="0.5"/>
</item>

- Common decorator attributes: `image-zorder`, `image-path`, `image-scale`, `image-width`, `image-height`, `image-rotate` (can contain dynamic expressions).

Other decorators exist for progress/tile/padding etc., and many can accept dynamic expressions such as `image-rotate='$${ Sensor_Value - 45 }'`.

## Layout-level attributes and features from changelog
- `if` — conditional rendering for section or item. Can use sensor names, boolean flags like `PortraitLayout`, `LandscapeLayout`, or compound expressions.
- `ignore-gauges` in section — ignores global gauge attributes for all elements inside the section.
- `gauge` local definitions — gauge definitions can be declared inside `section` to scope them locally.
- `swap-direction` on gauges: `swap-direction="true"` to change direction of bar or roundbar.
- `horizontal-offset` in roundbar.
- `break` decorator — `<break/>` splits rendering into independent stages.
- `maximize-focus`, `controls="autohide"` — useful for dash panels.
- `time-scale` — used in charts to compress/expand time axis.
- `text-evaluator` — advanced gauge attribute (returns arbitrary dynamic text), used in tank/description rendering.

Additional attributes and behaviors observed in layout files and changelogs:

- `optional` / `volatile` on `section` (controls visibility rules in some hosts).
- `period` on `item` — refresh period in milliseconds.
- `precision` — numeric formatting or special formats (e.g., `6fixed`).
- `colspan` / `rowspan` — grid spanning.

Example of a chart item with filters and custom description:

<item id="Speed" wrap="predict(50,0.7)" type="chart" chart='line' period="100" precision="0" custom-description="Predictive speed" border-opacity="1"/>

## Sensor and expression features
- Sensors can be referenced directly by `id` in `if` attributes or as item ids.
- TinyExe helpers and `eval` expressions available in ecuxml. Variables and functions accessible include: Pi, E, PositiveInfinity, NegativeInfinity and other globals from themes or gauge definitions.
- Filters for sensors (wrap) as above.

TinyExe / eval example (from changelogs):

`image-rotate='$${ Sensor_Value - 45 }'`

Sensor formatted access example:

`<item id="TripTime[fillup]"/>` — uses extradata from sensor history APIs.

## Sensor lists and autogenerated panels
- `section` with `class="SensorList"` and `ns="<namespace>"` generates a screen listing sensors for a namespace. Use `exclude` regex to filter out unwanted sensor IDs.
  Example:
  <section name="Ford" class="SensorList" ns="Ford_MondeoDuratorq2003" if="Ford_MondeoDuratorq2003.Speed" exclude="\\bCACT_|...">
  </section>

## Examples (extracted)
- Simple GPS screen (from `ecu-gps.layout`):

<section name="GPS">
  <grid rows=",," cols=",,">
    <item id='Latitude' precision='6fixed' size="normal"/>
    <item id='Longitude' precision='6fixed' size="normal"/>
    <item id='Satellites'/>
    <item id='Altitude' precision='1'/>
    <item id='GPSSpeed' precision='1'/>
    <item id='ActiveSatellites'/>
    <item id='Direction'/>
    <item id='GPSOBD2SpeedDiff'/>
    <item id='FixedPosition'/>
  </grid>
</section>

- Fan control (buttons + inherit decorator):

<section name="FAN" fixed="false" if="PortraitLayout">
  <grid rows=",," cols="">
    <item id="FAN_Status" inherit="_ColoredOnOff" size='large'/>
    <item type="button" text="Fan On" action="invoke(FAN_ON)" size='extralarge' />
    <item type="button" text="Fan Off" action="invoke(FAN_OFF)" size='extralarge' />
  </grid>
</section>

- Fuel trims (conditional section with grid, colspan example):

<section name="Fuel Trims" if="STFT1">
  <grid rows="," cols=",">
    <item id="STFT1" size='large' precision="1" />
    <item id="STFT2" size='large' precision="1" />
    <item id="LTFT1" size='large' precision="1" />
    <item id="LTFT2" size='large' precision="1" />
  </grid>
</section>

## Conditional expressions
- The `if` attribute can be a simple sensor presence test (e.g., `if="STFT1"`) or a full expression referencing theme variables, day/night flags, layout flags, and sensor values. Example seen: `if="SY_ABS_TEVES.ABS_ControlModuleVoltage, Chery_A21_ABS_TRW.ABS_ControlModuleVoltage"` (comma-separated OR?)

Notes: `if` can contain comma-separated identifiers which act as OR conditions in practice (layout files often list multiple sensors/namespaces separated by commas).

## Grid sizing hints
- `rows` and `cols` accept comma separated values. Empty entries are flexible cells, numbers provide fixed size hints. Example: `rows="30,,," cols="30,,30"`.

You can nest `grid` elements to create complex responsive regions.

## Other useful attributes and behaviors (from changelog and examples)
- `period` on `item` — sets refresh period for the widget.
- `precision` — formatting precision or special tags like `6fixed`.
- `description="aside"` — places descriptive text aside.
- `inherit` — reuses widget styling (colors/formatting) from theme or other definitions.
- `text-values` and `custom-units` allow mapping numeric values to textual labels or alternate unit systems.
- `min-limit` / `max-limit` — clamp values for charts.

## Widget and decorator attributes (observed in `user.layout`)

- type (examples): `bar`, `roundbar`, `chart`, `text`, `static` — many visual widgets are selected via `type` on the `item`.

- cache-interval — UI cache/refresh hint (milliseconds or unit) used to control smoothing/plotting; frequently used on `chart` and gauge-like widgets. Example: `cache-interval="6"`.

- red-green — color threshold list; comma-separated numbers indicating color boundaries (interpretation depends on widget). Examples: `red-green="100,5"`, `red-green="10,120,150"`.

- bars-count — number of bars or segments used by bar/roundbar widgets. Example: `bars-count="10"`.

- transparent — `true/false` whether background is transparent for the widget.

- border-width / border-opacity — visual border control. Example: `border-width="3"`, `border-opacity="1"`.

- colored-value — whether value text is colored according to thresholds: `colored-value="true"`.

- fade-coeff — visual fading coefficient used on charts/gauges to control trail transparency: `fade-coeff="0.8"`.

- width-compression — compress drawing horizontally: `width-compression="0.9"`.

- item-size / item-width / item-height / item-spacing — sizing controls for roundbars and similar widgets (absolute/logical units). Example: `item-size="25"`, `item-width="10"`, `item-height="10"`, `item-spacing="20"`.

- small-radis-delta / small-radis-delta (typo variants observed) — small radius adjustments used for roundbars.

- start-angle / end-angle / radius / radius / vertical-offset — roundbar geometry attributes to control arc start/end and offset.

- item-width/item-height vs item-size — some widgets use `item-size` while others use width/height pair.

- corn-angle / middle-angle — additional roundbar geometry attributes (observed in test layouts).

- decorator-p (padding) and p-padding — decorator shorthand for padding decorator. Example: `decorator-p="padding" p-padding="10 10 10 10"`.

- decorator-i / decorator-iN pattern — numbered decorators for multiple images on a single item. Use `decorator-i1="image"` and `i1-image-path`, `i1-image-rotate`, `i1-image-zorder`, `i1-image-rotate-align`, etc. The unnumbered `decorator-i="image"` with nested `<image>` elements is also supported.

  Examples:
```xml
<item id="Speed" decorator-i1="image" i1-image-zorder="bottom" i1-image-path="images/gauges/without_arrow_circle.png" i1-image-rotate='$${...}' i1-image-rotate-align="center"/>

<item id="Speed">
  <image zorder="bottom" path="images/gauges/without_arrow_circle.png"/>
  <image zorder="bottom" path="images/gauges/arrow.png" rotate='$${...}' scale="1"/>
</item>
```

- rotate / scale attributes on nested `<image>` decorators — `rotate` can contain dynamic TinyExe expressions and `scale` applies an image scale.

- custom-units — used on `DateTime` to show time variants: `custom-units="time12"`, `time`, `timesec`, `datetime`, `seconds`.

- border-opacity used on static/test items to preview borders: `border-opacity="1"`.

- numeric `size` on `item` — static items can use numeric `size` values (e.g., `size="72"`, `size="32"`) besides named sizes (`small`, `normal`, `large`).


Common `type` values seen in dashkits:

- `type="text"` — textual rendering of a sensor with `text-values` mapping.
- `type="chart"` — time-series chart for sensor.
- `type="button"` — user-invokable action.

Example: `text-values` and `type=text` usage in dashb3 kits:

`<item id="RPM" wrap="predict(50, 0.7)" text-values="0:" type="text" description="hidden">`

## Advanced: ecuxml and sensor definitions
- sensor definitions and `eval` expressions live in ecuxml files. Key features mentioned across changelogs:
  - `<eval>` expressions for computed sensors
  - `base-data` / `base-raw` for raw bytes
  - functions `get(offset)`, `get_word()` etc.
  - `text` tag with offset/length for string sensors

## Best practices and tips
- Use `if` broadly to keep layout files modular; layout files are only loaded for relevant ECUs, but `if` helps avoid rendering empty sections.
- Use `SensorList` sections for generic sensor browsing screens and `exclude` to filter noisy sensors.
- Prefer nested decorators for complex visuals; they are simpler and localized.
- Use `wrap` filters on noisy sensors (GPS, speed, MAF) for smoother rendering.
- Keep gauges local to section when possible using `ignore-gauges` and local `gauge` declarations to reduce global side-effects.

## Appendix — common attributes quick reference
- section: name, fixed, if, class, ns, exclude, maximize-focus, controls
- grid: rows, cols
- item: id, type, text, action, size, precision, inherit, colspan, rowspan, description, period, wrap
- decorators: image (image-path, image-scale, image-zorder, image-rotate, image-width/height)


---

If you want, I can:
- scan all `hobd/config/*.layout` files and produce a table of all observed attributes and which files used them (helpful for completeness), or
- extend this doc with a small set of template layouts (GPS, Fan control, Hybrid panel) ready to copy into a new skin.

Completion: created `hobdrive-dashkits/LAYOUT_SPEC.md` with collected tags and examples.

## Try it — ready templates
Copy these minimal examples into a skin file to try the patterns.

GPS panel (minimal)
```xml
<ui>
  <section name="GPS">
    <grid rows=",," cols=",,">
      <item id='Latitude' precision='6fixed' size="normal"/>
      <item id='Longitude' precision='6fixed' size="normal"/>
      <item id='GPSSpeed' precision='1' />
    </grid>
  </section>
</ui>
```

Fan control (buttons + inherit)
```xml
<ui>
  <section name="FAN" fixed="false" if="PortraitLayout">
    <grid rows=",," cols="">
      <item id="FAN_Status" inherit="_ColoredOnOff" size='large'/>
      <item type="button" text="Fan On" action="invoke(FAN_ON)" size='extralarge' />
      <item type="button" text="Fan Off" action="invoke(FAN_OFF)" size='extralarge' />
    </grid>
  </section>
</ui>
```

Hybrid summary example
```xml
<ui>
  <section name="Hybrid">
    <grid rows="30,,," cols="30,,30">
      <item id="Efficiency_Battery" size='huge' />
      <item id="FuelEconomy_NoBattery" size='huge' />
      <item id="BatteryCurrent" />
      <item id="SoC" />
    </grid>
  </section>
</ui>
```

If you'd like, I can also:
- generate a CSV of all attributes used across `hobd/config/*.layout` (file-by-file), or
- create a smaller 'quick-reference' cheat-sheet suitable for printing.

---

Done — `hobdrive-dashkits/LAYOUT_SPEC.md` is now saved in the workspace.