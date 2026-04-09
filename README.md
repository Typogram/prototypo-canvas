# prototypo-canvas

**Role:** The canvas rendering layer on top of prototypo.js. It wraps an `HTMLCanvasElement` with a Paper.js scene (sourced from the plumin/Paper.js bundled inside prototypo.js — never a separate copy), manages font loading via a SharedWorker, handles pan/zoom/node-drag mouse interaction, draws typographic frame guides, and exposes an event-based API for the host application to react to user edits. It has no paper.js of its own — all `paper.*` access goes through `prototypo.paper`.

---

## Exposed API

### Static methods

| Method                              | Description                                                                             |
| ----------------------------------- | --------------------------------------------------------------------------------------- |
| `PrototypoCanvas.init(opts)`        | Factory. Creates the SharedWorker, sets up Paper.js, returns `Promise<PrototypoCanvas>` |
| `PrototypoCanvas.stopRaf(instance)` | Cancel the `requestAnimationFrame` render loop on an instance                           |

### Instance methods

| Method                                                  | Description                                                                      |
| ------------------------------------------------------- | -------------------------------------------------------------------------------- |
| `instance.loadFont(name, fontSource)`                   | Load a parametric font by URL or JSON string; returns `Promise<PrototypoCanvas>` |
| `instance.displayChar(charOrCode)`                      | Display a glyph by character string or unicode code point                        |
| `instance.displayGlyph(glyph)`                          | Display a raw glyph object from `font.glyphMap` / `font.charMap`                 |
| `instance.displayComponents(glyph)`                     | Render all components of a composite glyph                                       |
| `instance.displayComponentList(list)`                   | Render a specific list of component glyphs                                       |
| `instance.changeComponent(componentId, componentName)`  | Switch a component variant on the current glyph                                  |
| `instance.update(values)`                               | Re-evaluate all formulas and redraw (queued through the worker pipeline)         |
| `instance.setupCanvas(canvas)`                          | Reassign the instance to a different canvas element                              |
| `instance.setupEvents(instance)`                        | (Re)attach mouse/keyboard event handlers                                         |
| `instance.getGlyphsProperties(properties, callback)`    | Read specific properties across all glyphs                                       |
| `instance.setAlternateFor(unicode, glyphName)`          | Swap in an alternate glyph for a unicode codepoint                               |
| `instance.download(name, merged, values)`               | Download the current font as an OTF file                                         |
| `instance.getBlob(cb, name, merged, values)`            | Get the font as a Blob (used internally by download/export)                      |
| `instance.generateOtf(name, merged, values)`            | Generate an OTF ArrayBuffer via the worker                                       |
| `instance.openInGlyphr(cb, name, merged, values, user)` | Open the font in Glyphr Studio                                                   |
| `instance.enqueue(message)`                             | Push a job onto the worker queue                                                 |
| `instance.dequeue()`                                    | Process the next job in the worker queue                                         |
| `instance.emptyQueue()`                                 | Discard all pending queued jobs                                                  |

### Instance properties (gettable/settable)

| Property              | Description                                             |
| --------------------- | ------------------------------------------------------- |
| `instance.zoom`       | Canvas zoom level                                       |
| `instance.fill`       | `true` = filled glyphs, `false` = outlines only         |
| `instance.showNodes`  | Show built-in Paper.js skeleton node handles            |
| `instance.showCoords` | Draw coordinate labels on nodes                         |
| `instance.subset`     | Restrict active glyph subset (string or array)          |
| `instance.allowMove`  | `false` disables canvas pan on drag (use in edit modes) |
| `instance.font`       | The live compiled `Font` object (set after `loadFont`)  |
| `instance.currGlyph`  | The glyph currently rendered on canvas                  |
| `instance.view`       | The Paper.js `View` — call `.update()` to redraw        |

### Events (via EventEmitter)

| Event                | Payload                | Description                                                 |
| -------------------- | ---------------------- | ----------------------------------------------------------- |
| `manualchange`       | `(cursors, isConfirm)` | User dragged a node; cursors is a map of per-node overrides |
| `manualreset`        | `(contourId, nodeId)`  | User reset a node to its formula value                      |
| `worker.fontCreated` | —                      | Worker finished building the font                           |
| `worker.fontLoaded`  | —                      | Font buffer added to the browser font stack                 |
| `fonterror`          | `(error)`              | Font loading or parsing failed                              |

---

## Install

```bash
npm install prototypo-canvas
```

prototypo-canvas expects `prototypo.js` as a peer. In a bundled environment, add both as dependencies. When loading via script tags, ensure `prototypo.js` is loaded first — the canvas module references it as `global:prototypo`.

---

## Quick start

```js
var PrototypoCanvas = require("prototypo-canvas");

PrototypoCanvas.init({
	canvas: document.getElementById("my-canvas"),
	workerUrl: "/path/to/worker.js", // or omit to build from source
})
	.then(function (instance) {
		return instance.loadFont("elzevir", "/fonts/elzevir.json");
	})
	.then(function (instance) {
		instance.displayChar("A");
		instance.font.update({
			thickness: 80,
			xHeight: 500,
			contrast: 50,
			width: 100,
		});
		instance.view.update();
	});
```

---

## Initialization

### `PrototypoCanvas.init(opts)` → `Promise<PrototypoCanvas>`

Static factory. Creates a SharedWorker (for off-thread font compilation), sets up the Paper.js scope, and returns a resolved instance.

| Option            | Type                 | Default              | Description                                                     |
| ----------------- | -------------------- | -------------------- | --------------------------------------------------------------- |
| `canvas`          | `HTMLCanvasElement`  | required             | The canvas element to render into                               |
| `workerUrl`       | `string`             | auto-built blob      | URL to `worker.js`. Omit to build the worker inline from source |
| `workerDeps`      | `string \| string[]` | —                    | Additional scripts to load inside the worker                    |
| `fill`            | `boolean`            | `true`               | Whether glyphs are filled or outline-only                       |
| `showNodes`       | `boolean`            | `false`              | Show Paper.js skeleton node handles                             |
| `zoomFactor`      | `number`             | `0.05`               | Mouse-wheel zoom step                                           |
| `jQueryListeners` | `boolean`            | `true`               | Attach jQuery-based mouse wheel listeners                       |
| `glyphrUrl`       | `string`             | Glyphr Studio online | URL used for glyph export                                       |

---

## Loading fonts

### `instance.loadFont(name, fontSource)` → `Promise<PrototypoCanvas>`

Loads a parametric font. `fontSource` can be a URL string or a raw JSON string.

```js
instance.loadFont("elzevir", "/fonts/elzevir.json").then(function () {
	instance.displayChar("A");
});
```

Internally:

1. Fetches/parses the font JSON
2. Posts it to the SharedWorker to precompute `solvingOrder` for each glyph
3. Calls `prototypo.parametricFont(fontObj)` to compile the live font
4. Stores the result in `instance.font` and `instance.fontsMap[name]`

---

## Displaying glyphs

### `instance.displayChar(char)`

Display a glyph by its character (string) or unicode code point (number).

```js
instance.displayChar("A");
instance.displayChar(65); // same as above
```

### `instance.displayGlyph(glyph)`

Display a raw glyph object directly from `instance.font.glyphMap` or `instance.font.charMap`.

### `instance.displayComponents(glyph)`

Render all component glyphs that make up a composite glyph.

---

## Updating parameters

After loading, call `instance.font.update(params)` to re-evaluate all formulas and redraw:

```js
instance.font.update({
	thickness: 80,
	xHeight: 500,
	contrast: 50,
	width: 100,
	slant: 0,
});
instance.view.update();
```

For glyph-restricted updates (better performance when only one glyph is visible):

```js
instance.font.update(params, [instance.currGlyph]);
instance.view.update();
```

---

## Instance properties

| Property             | Type                | Description                                |
| -------------------- | ------------------- | ------------------------------------------ |
| `instance.font`      | `Font`              | The live compiled font object              |
| `instance.currGlyph` | `Glyph`             | The glyph currently displayed on canvas    |
| `instance.view`      | `paper.View`        | Paper.js view — call `.update()` to redraw |
| `instance.canvas`    | `HTMLCanvasElement` | The canvas element                         |
| `instance.scope`     | `paper.PaperScope`  | The Paper.js paper scope                   |
| `instance.project`   | `paper.Project`     | The active Paper.js project                |
| `instance.worker`    | `SharedWorker`      | The background worker                      |

### Settable properties

```js
instance.zoom = 1.5; // zoom level (0.1 – ~10)
instance.fill = true; // filled vs. outline rendering
instance.showNodes = false; // show built-in Paper.js node handles
instance.showCoords = false; // draw coordinate labels
instance.subset = "ABC"; // restrict active glyph subset
instance.allowMove = false; // disable canvas pan on drag (use false in edit modes)
```

---

## Coordinate systems

The Paper.js layer has `scale(1, -1)` applied so that font-space (y-up) coordinates render correctly on screen (y-down).

**Critical:** `node.point.x/y` are in **layer-local (y-up font) space**. `instance.view.viewToProject([px, py])` returns **project space (y-down)**. To compare them:

```js
// Convert a DOM pointer event to project space:
var bcr = canvas.getBoundingClientRect();
var pt = instance.view.viewToProject([
	e.clientX - bcr.left,
	e.clientY - bcr.top,
]);

// Convert a node's position to project space for hit-testing:
var nodeProjectY = -node.point.y; // negate y
var dx = node.point.x - pt.x;
var dy = nodeProjectY - pt.y;
var dist = Math.sqrt(dx * dx + dy * dy);
```

---

## Drawing custom overlays

Use `paper.Shape.*` factory methods (not `new paper.Path.*`) to position items at font-unit coordinates. Shapes keep their transform matrix separate from the layer, which means the layer's `scale(1,-1)` is applied correctly.

```js
var paper = instance.scope;

// Circle at a skeleton node position:
var circle = paper.Shape.Circle(node.point, radius);
circle.fillColor = "#4af";
circle.strokeColor = "none";

// Thin line between two points A and B (both in layer-local / y-up space):
var dx = B.x - A.x,
	dy = B.y - A.y;
var len = Math.sqrt(dx * dx + dy * dy);
var cx = (A.x + B.x) / 2,
	cy = (A.y + B.y) / 2;
var angleDeg = (Math.atan2(dy, dx) * 180) / Math.PI;

var bone = paper.Shape.Rectangle({
	x: cx - len / 2,
	y: cy - 1,
	width: len,
	height: 2,
});
bone.rotation = angleDeg;
bone.fillColor = "#4af";
```

Group overlays in a `paper.Group` so you can remove them cleanly:

```js
var overlayGroup = new paper.Group();
// add shapes to overlayGroup
overlayGroup.remove(); // cleans up all children
```

---

## Events

`PrototypoCanvas` extends `EventEmitter`. Listen with `.addListener(event, handler)`.

| Event                | Payload                | Description                                                   |
| -------------------- | ---------------------- | ------------------------------------------------------------- |
| `manualchange`       | `(cursors, isConfirm)` | User dragged a node; `cursors` is a map of per-node overrides |
| `manualreset`        | `(contourId, nodeId)`  | User reset a node override to formula default                 |
| `worker.fontCreated` | —                      | Worker finished building the font                             |
| `worker.fontLoaded`  | —                      | Font buffer was added to the browser's font stack             |
| `fonterror`          | `(error)`              | Font loading/parsing failed                                   |

---

## Manual node overrides

Formula values are overwritten on every `font.update()` call. To persist manual edits, store overrides keyed by `"contourIndex-nodeIndex"` and re-apply them in a post-update step:

```js
var expandOverrides = {}; // { "0-1": { width, angle, distr } }
var xyOverrides = {}; // { "0-1": { x, y } }

function applyOverrides() {
	var naive = window.prototypo.Utils.naive;

	instance.currGlyph.contours.forEach(function (contour, ci) {
		if (!contour.skeleton) return;
		contour.nodes.forEach(function (node, ni) {
			var key = ci + "-" + ni;

			if (xyOverrides[key]) {
				node.point.x = xyOverrides[key].x;
				node.point.y = xyOverrides[key].y;
			}
			if (expandOverrides[key]) {
				node.expand = Object.assign({}, expandOverrides[key]);
			}

			var w = node.expand && node.expand.width;
			if (node.expandedTo) {
				naive.expandedNodeUpdater(node.expandedTo[0], true, w);
				naive.expandedNodeUpdater(node.expandedTo[1], false, w);
				naive.skeletonCopier(node);
			}
		});
	});
}

// Usage:
instance.font.update(params);
applyOverrides();
instance.view.update();
```

---

## Reading node values

After `font.update()` has run, read current evaluated values directly from node objects:

```js
var glyph = instance.currGlyph;

glyph.contours.forEach(function (contour, ci) {
	if (!contour.skeleton) return;
	contour.nodes.forEach(function (node, ni) {
		// Position (font-space, y-up):
		console.log(node.point.x, node.point.y);

		// Stroke shape:
		console.log(node.expand.width); // stroke thickness
		console.log(node.expand.angle); // expansion angle (radians)
		console.log(node.expand.distr); // distribution 0–1

		// Curve style:
		console.log(node.type); // 'smooth' | 'corner'
		console.log(node.tensionIn, node.tensionOut);
	});
});
```

---

## Typographic frame

The canvas automatically draws guide lines for baseline, x-height, and cap-height, and spacing side-bearings. These are driven by values passed to `font.update()`:

- `xHeight` — height of the x-height guide
- `capDelta` — cap height = xHeight + capDelta
- `instance.currGlyph.ot.advanceWidth` — right side-bearing

The frame updates on every animation frame via `requestAnimationFrame`.

---

## Worker

prototypo-canvas offloads font compilation (dependency-tree solving) to a `SharedWorker`. The worker is shared across all `PrototypoCanvas` instances on the same page.

To use a pre-built worker file instead of the inline blob:

```js
PrototypoCanvas.init({
	canvas: myCanvas,
	workerUrl: "/static/worker-v2.js",
});
```

---

## Stopping the render loop

```js
PrototypoCanvas.stopRaf(instance);
```

Call this before unmounting the canvas to cancel the `requestAnimationFrame` loop.

---

## Build

```bash
npm install

npm run gulp    # build to dist/
```

---

## License

MIT
