# Gridwright

Gridwright is a block-based game maker for the desktop, with built-in 3D, a player/exporter for standalone Linux and
Windows apps, and a configurable stage resolution. One monorepo holds the editor, the desktop app and the exporter.
It is based on open-source software from the Scratch project: see `NOTICE.md`.

| Package | What it is |
| --- | --- |
| `packages/scratch-gui`, `scratch-vm`, `scratch-render`, `scratch-paint`, `scratch-storage`, `scratch-svg-renderer`, `task-herder` | The Scratch editor, from [scratchfoundation/scratch-editor](https://github.com/scratchfoundation/scratch-editor) |
| `packages/scratch-desktop` | Electron app around the editor, from [scratchfoundation/scratch-desktop](https://github.com/scratchfoundation/scratch-desktop), now using the local `scratch-gui` and with a **player mode** |
| `packages/scratch-exporter` | Packages an `.sb3` into a Linux **AppImage** or a single-file Windows **exe** |

Because the editor, VM and renderer are Scratch's own code, project behaviour is Scratch's own behaviour.

## Setup

Requires Node 22+ (`.nvmrc` says 24).

```sh
npm install
npm run build                 # builds the editor packages (several minutes)
npm run build:dist-files      # compiles the desktop app into packages/scratch-desktop/dist
```

If your network blocks `downloads.scratch.mit.edu` (the micro:bit firmware download), set
`SKIP_MICROBIT_HEX=1` for `npm install` and `npm run build`; the micro:bit firmware file will then be empty.
If your network blocks Playwright's CDN, set `PLAYWRIGHT_SKIP_BROWSER_DOWNLOAD=1` (only needed for GUI tests).

## Run the editor

```sh
npm start
```

## Export a project as a game

**From the editor:** choose **File → Export as app (AppImage / exe)...**, pick Linux, Windows or both, then pick an
output folder. A banner shows progress and a dialog appears when the files are ready. This needs the app to be run
from this repository (it calls `packages/scratch-exporter`), and `npm run build:dist-files` to have been run.

**From the command line:**

```sh
npm run export -- examples/hello.sb3 --target linux,win --name "Hello" --out out
```

* `linux` produces an AppImage (`chmod +x` it and run it).
* `win` produces one portable `.exe`. It can be built from Linux; the first run downloads electron-builder's
  Windows tooling and the Electron runtime, so it needs internet access.
* `--fullscreen` starts the game in a fullscreen window; F11 toggles it either way.
* The Windows build compresses with 7-zip and takes several minutes on a single CPU.
* Both targets have been built from Linux. The AppImage has been launched and runs the embedded project; the Windows
  exe has been built but not yet run on Windows.

An exported game is this same desktop app started in player mode: it opens straight into the embedded project and
starts it automatically. There is no editor, no green flag / stop / fullscreen bar, no start overlay and no loading
animation, and the stage fills the window. F11 still toggles fullscreen.

You can try player mode without exporting: `npm start -- --player=examples/hello.sb3`.

## 3D

The **3D** blocks are always available (the black category at the bottom of the block list). They use Three.js and is drawn into the stage as
a layer, so it works with sprites, backdrops, the pen and exported apps. About 50 blocks:

* **Scene:** background colour, ambient light, sun direction and brightness, fog, shadows, put 3D behind or in front
  of the sprites, reset.
* **Objects:** cube, sphere, cylinder, cone, plane, torus, capsule; imported models; load a model from a URL; duplicate, delete;
  position, rotation, size, point at, move forward; colour, material (matte, shiny, flat, wireframe, glowing),
  opacity, texture from a costume or an image URL; show and hide.
* **Physics:** gravity, floor, velocity and bounciness per object.
* **Camera:** position, point at, orbit around a point, move forward, field of view.
* **Sensing:** position and size of an object, distance, touching, mouse over, object at a stage position,
  camera position, number of objects, render time.

Coordinates: x right, y up, z towards the viewer; a new shape is 100 units across; angles are in degrees.

3D is the only extension, and extensions cannot be added. Projects made elsewhere that use Pen, Music or other
extensions open, but those blocks do nothing.

## Importing 3D models and animating them

**File → 3D models...** imports `.obj`, `.glb` and `.gltf` files (glTF files must be self-contained, which `.glb` always
is). Models are saved inside the project file, so they travel with exported apps.

* **Import models (one per file):** each file becomes a model with one frame.
* **Import as animation (one frame per file):** the files become the frames of one model, in natural filename order
  (`walk_1`, `walk_2`, ... `walk_10`). Frames can be reordered or removed afterwards.
* **Add frames to selected:** appends more frames to the selected model.

Animation works like costumes: a model is a list of frames, and each frame is a complete pose (a stop-motion
animation). Blocks:

* `create imported model [model] named [name] at x y z`
* `switch [name] to frame [number or name]`, `next frame of`, `previous frame of`
* `play [name] animation at [fps] frames per second [looping / once / back and forth]`, `stop [name] animation`
* `frame number of`, `frame name of`, `number of frames of`

All frames of a model are centred and scaled from its first frame (the longest side becomes 100 units), so frames
line up. Imported models keep their own materials and textures until you use `set color`, `set material`,
`set opacity` or `set texture` on them. Skeletal glTF animation clips are not played; export each pose as a frame.

## Game blocks (the **Game** category)

* **Saves:** `save [value] as [name]`, `saved [name]`, `[name] is saved?`, `delete saved [name]`,
  `delete everything saved in this slot`, `use save slot [number]`. Saved values survive closing the game and are kept
  per game and per slot, in the app's data folder (Linux: `~/.config/<game name>/saves/<game>.json`). In the editor
  they are kept under "editor". Values can be text, numbers or booleans.
* **Window:** `turn fullscreen on/off`, `toggle fullscreen`, `fullscreen?`, `set window title to`,
  `turn mouse pointer on/off`, `quit game`. F11 or Alt+Enter toggles fullscreen in the editor and in games, and the
  File menu has **Fullscreen (F11)**. In fullscreen the stage fills the screen with black bars where the shape does
  not match.
* **Gamepad:** `gamepad connected?`, `gamepad [A, B, X, Y, bumpers, triggers, back, start, sticks, d-pad] pressed?`,
  `[left/right] stick [x/y]` (-100 to 100, up is positive, small drift ignored). Uses the first controller the
  system reports.
* **Timing:** `frames per second`, `seconds since last frame`, `seconds since the game started`.

In 3D, `glide [name] to x y z in [seconds] [easing]` moves an object smoothly (linear, ease in, ease out, ease in and
out, bounce).

## Dark mode

Settings → Color Mode → Light, Dark or High Contrast. The choice is remembered; with no saved choice the system
setting is used. Dark mode recolours the whole editor and the block palette, and the colours live in
`packages/scratch-gui/src/css/theme-vars.css`.

## Performance

* **3D:** the scene only renders when something changed; "set 3D render quality" renders at a lower resolution;
  "limit 3D rendering to N frames per second"; anti-aliasing and shadows are switchable (shadows are off by default).
* **Frame rate:** File → Stage resolution and frame rate lets a project run at 30, 45, 60, 90, 120 or 144 fps.
* **GPU:** the app asks Chromium to ignore its GPU blocklist and use GPU rasterisation. Exported games ask for the
  fast GPU on laptops with two, and keep running at full speed when the window is covered.

## Stage resolution

File → Stage resolution and frame rate. Choose a preset (4:3, 16:9, 16:10, square) or type a width and height
from 16 to 4096. The resolution and frame rate are saved in the project (only when they differ from Scratch's
480×360 at 30 fps, so ordinary projects stay identical to official Scratch files) and exported apps use them.
Sprites keep their pixel size, and costumes and backdrops keep theirs, so a larger stage needs larger backdrops.

## Not done yet

* Online features (cloud variables, project sharing, accounts).
* The in-editor export button only works when running from the repository, not from an installer-packaged app.
* The File menu stays open after clicking the export item.
* Custom icons for exported games, and macOS export.

## Licence

AGPL-3.0-only, as in the upstream Scratch repositories. Scratch's name and logo are trademarks of the Scratch
Foundation (see `TRADEMARK`); rebrand before distributing.
