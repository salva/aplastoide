# Implicit Surface Explorer

Standalone browser app for exploring the implicit 3D surface:

```text
x^2 + y^2 + z^2 = cos^2(a * pi * x * y * z)
```

The app renders the main implicit object and overlays level lines for fixed values of `x`, `y`, and `z`.

## Spec

### Mathematical Object

The app visualizes the implicit equation:

```text
F(x, y, z, a) = x^2 + y^2 + z^2 - cos^2(a * pi * x * y * z) = 0
```

Although initially described as a curve, this equation has three spatial variables and one scalar equation, so it generally defines a two-dimensional implicit surface in 3D.

The current sampling domain is:

```text
-1 <= x <= 1
-1 <= y <= 1
-1 <= z <= 1
```

This domain is natural because the right-hand side is at most `1`, so any solution must satisfy:

```text
x^2 + y^2 + z^2 <= 1
```

### Parameter `a`

The parameter `a` is user-controlled with a slider.

Range:

```text
0 <= a <= 12
```

### Interaction

The figure must be interactively rotatable.

Current controls:

- Drag to rotate.
- Scroll or pinch to zoom.
- Pan behavior is provided by the underlying orbit-control library.

### Level Lines

The app draws contour/level lines where one coordinate is fixed:

```text
x = constant
y = constant
z = constant
```

The user chooses a single number of level lines. That same number is used for all three axes.

For example, if the user chooses `7`, the app attempts to draw:

```text
7 fixed-x contour slices
7 fixed-y contour slices
7 fixed-z contour slices
```

The level lines are overlaid on the main implicit object.

## Algorithm

The app uses numerical sampling rather than symbolic geometry.

### Implicit Function

The equation is represented as a scalar field:

```js
F(x, y, z, a) = x*x + y*y + z*z - cos(a * PI * x * y * z)^2
```

Points on the surface are locations where `F = 0`.

### Main Surface Approximation

The main object is currently rendered as a point-cloud approximation.

Algorithm:

1. Divide the cube `[-1, 1]^3` into a regular 3D grid.
2. Evaluate `F` at the eight corners of each grid cell.
3. If the cell contains both negative and positive values, the zero surface crosses that cell.
4. For each crossed cell edge, linearly interpolate an approximate zero-crossing point.
5. Add the zero-crossing points to a Three.js `Points` object.

This is similar to the sampling phase of marching cubes, but without building triangles. It is fast enough for interactive exploration and avoids mesh-topology complexity.

### Level Line Approximation

Level lines are computed by slicing the implicit surface with fixed-coordinate planes.

For each selected fixed value:

```text
x = c
```

the app samples the 2D plane in the remaining coordinates:

```text
(y, z)
```

It then finds where `F(c, y, z, a) = 0`.

The same approach is used for:

```text
y = c -> sample (x, z)
z = c -> sample (x, y)
```

Algorithm:

1. Divide each slice plane into a 2D grid.
2. Evaluate `F` at the four corners of each grid square.
3. If the square contains a sign change, the contour crosses that square.
4. Interpolate along crossed edges to find approximate zero-crossing points.
5. Connect pairs of crossing points as line segments.

This is a marching-squares style contour algorithm.

### Rendering Tradeoffs

The current implementation favors simplicity and responsiveness:

- The surface is a point cloud, not a triangle mesh.
- Contours are line segments, not globally stitched polylines.
- Sampling resolution is adjustable from the UI.
- Higher resolution improves quality but costs CPU time.

## App Architecture

The app is a single static HTML file:

```text
index.html
```

It can be opened directly in a browser or served by any static file server.

### External Runtime Dependencies

The page imports Three.js from a CDN through an import map:

```text
three
three/addons/controls/OrbitControls.js
```

No build step is required.

### Main Components

The implementation is organized into these logical areas inside `index.html`:

- HTML controls: sliders for `a`, level-line count, and sampling resolution.
- CSS layout: fullscreen canvas with a translucent control panel.
- Three.js setup: scene, camera, renderer, lights, grid, axes, and orbit controls.
- Scalar-field code: `implicitValue(x, y, z, a)`.
- Surface sampler: `makeSurface(a, resolution)`.
- Slice contour sampler: `makeLevelLines(a, count, resolution)`.
- Rebuild loop: regenerates geometry when slider values change.
- Animation loop: updates orbit controls and renders each frame.

### Data Flow

User input flows through the app as follows:

```text
slider input
-> scheduleRebuild()
-> read current a / level count / resolution
-> recompute point cloud
-> recompute level lines
-> replace Three.js geometry
-> render updated scene
```

### Visual Encoding

The app uses separate colors for the fixed-axis contours:

```text
fixed x lines: pink
fixed y lines: green
fixed z lines: yellow
```

The main implicit object is drawn as a cyan point cloud.

## Implementation Plan

### Current Implementation

- Create a standalone `index.html`.
- Use Three.js for 3D rendering.
- Use `OrbitControls` for rotation and zoom.
- Implement the implicit scalar field.
- Render the main object as a sampled point cloud.
- Overlay fixed-`x`, fixed-`y`, and fixed-`z` level lines.
- Use one shared level-line-count slider for all three axes.
- Allow `a` to vary over `0 <= a <= 12`.
- Allow sampling resolution to be adjusted.

### Near-Term Improvements

- Add checkboxes to show or hide each axis family of level lines.
- Add a legend explaining the contour colors.
- Add a reset-camera button.
- Preserve UI state in the URL hash or local storage.
- Add a loading indicator for high-resolution rebuilds.

### Geometry Improvements

- Replace the point-cloud surface with a true triangle mesh using marching cubes.
- Compute normals for better lighting.
- Add transparency controls for the surface.
- Stitch contour segments into longer polylines.
- Add optional slice-plane previews.

### Performance Improvements

- Move sampling into a Web Worker so UI interaction remains smooth during rebuilds.
- Cache scalar-field evaluations where possible.
- Use typed arrays more directly to reduce allocations.
- Add adaptive sampling near high-curvature regions.

### Validation Ideas

- Verify that all rendered samples satisfy `abs(F(x, y, z, a))` below a chosen tolerance.
- Test known special case `a = 0`, where the equation simplifies to:

```text
x^2 + y^2 + z^2 = 1
```

This should render a unit sphere.

- Compare fixed-coordinate contours at `a = 0` with circle slices of the unit sphere.

## Running

Open `index.html` in a modern browser.

If browser module restrictions prevent direct file loading, serve the directory with any static server, for example:

```bash
python3 -m http.server 8000
```

Then open:

```text
http://localhost:8000
```
