# Dashkit authoring guide

For work based on an instrument-panel photo or screenshot, read
[the reference workflow](docs/REFERENCE_DASHKITS.md) before editing. It records the
sup4 visual-loop experience, including geometry, themes, masks, and sensor fixtures.
[LEARNING.md](LEARNING.md) contains additional layout-language lessons from classic90.

- Inspect the reference itself and the current rendered dashkit. Record dial centers,
  scale endpoints, curved alignments, label/value boundaries, and intended live data.
  Reproduce the instrument design; reflections, perspective, and camera blur are not
  artwork requirements. Resolve unclear details from context and report assumptions.
- Use editable SVG and layout XML for geometric instrument artwork. Keep sensor
  readings live. A reference containing digital readings already calls for them.
- Integrate the panel with the app's wallpaper. Keep unused space transparent;
  use local, feathered backing for contrast instead of an opaque full-section
  rectangle. Check the result over matching light and dark wallpapers.
- Support light and dark themes unless the user scopes the work otherwise. Follow
  the requested orientation scope; preserve an existing deferred orientation and
  give new artwork separate names when it must not affect that layout.
- Use nested XML decorators. Keep each dial, mask, and needle in one coordinate
  system with a uniform fit. Document non-obvious wrappers and renderer assumptions.
- Validate changes in the real Mac Catalyst Debug app through the Visual Loop.
  Inspect each capture and iterate on visible defects; XML validity or an SVG
  preview alone does not establish the rendered result. Reuse a running app and an
  `override-*` source link for layout/artwork edits, with `ForceReloadUI` between passes.
- Use deterministic sensor values and wait for correlated Debug action completion.
  Check active aliases and simulator formulas when injected values do not appear.
  Back up any profile files that need temporary changes, stop the app before editing
  or restoring them, and verify restoration after the pass. Do not reset the profile.
- Check both themes, normal and wide viewports, scale endpoints and clamp behavior,
  long/negative/decimal values, missing sensors, and supported units as applicable.
  Save actual app previews and document bindings, fixture values, and limitations.
- Keep changes within the target dashkit and relevant authoring documentation.
  Preserve unrelated and staged work. Publishing and remote sync require explicit
  user approval under the workspace guide.

Mac Catalyst commands and window-capture recovery are documented in
[the workspace Visual Loop guide](../hobd/docs/maccatalyst-visual-loop.md).
