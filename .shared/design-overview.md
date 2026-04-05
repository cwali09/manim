# manim (3b1b) — Design Overview

> Grant Sanderson's animation engine for explanatory math videos. GPU-accelerated via ModernGL.

## Entry Points

| Command | Description |
|---------|-------------|
| `manimgl scene_file.py SceneName` | Render a scene to video or open interactively |
| `manim-render` | Alternative entry point |
| `manimlib/__main__.py::main()` | Python entry — parses CLI, loads config, runs scene |

**CLI flags:** `-w` write file, `-o` save + open, `-s` skip animations, `-f` fullscreen, `-n N` skip to animation N.

## Data Flow

```
CLI args + config YAML → Scene instantiation → construct() → play(animations) → Camera.render() → FFmpeg → video
```

1. **Config** (`config.py`): merges `default_config.yml` + `custom_config.yml` + CLI args
2. **Scene discovery** (`extract_scene.py`): finds Scene subclass in user's file, instantiates it
3. **Scene.run()**: calls `setup()` → `construct()` → `interact()` → `tear_down()`
4. **Per animation**: `Animation.begin()` → interpolation loop (alpha 0→1) → `finish()`
5. **Per frame**: Camera renders mobjects via ModernGL FBO → FileWriter encodes via FFmpeg

## Key Abstractions

| Abstraction | Role | Location |
|-------------|------|----------|
| **Scene** | Top-level orchestrator — holds mobjects, camera, drives animations | `manimlib/scene/scene.py` |
| **Mobject** | Base renderable object — points, colors, shaders, updaters | `manimlib/mobject/mobject.py` |
| **VMobject** | 2D vector graphics via Bezier curves (most visible objects) | `manimlib/mobject/types/vectorized_mobject.py` |
| **Animation** | Time-interpolated transformation of mobjects | `manimlib/animation/animation.py` |
| **Camera** | ModernGL context, FBO management, 3D→2D projection | `manimlib/camera/camera.py` |
| **CameraFrame** | Viewport geometry — position, euler angles, zoom | `manimlib/camera/camera_frame.py` |
| **ValueTracker** | Observable numeric value for reactive scene updates | `manimlib/mobject/value_tracker.py` |
| **SceneFileWriter** | Frame capture + FFmpeg video encoding + audio mixing | `manimlib/scene/scene_file_writer.py` |

## Module Layout

```
manimlib/
├── __main__.py          # CLI entry
├── config.py            # Config system (YAML + CLI)
├── constants.py         # Colors, directions, defaults
├── extract_scene.py     # Scene class discovery
├── window.py            # Pyglet display window
├── animation/           # Animation types + rate functions
├── camera/              # Camera + CameraFrame
├── event_handler/       # Keyboard/mouse events
├── mobject/
│   ├── types/           # VMobject, Surface, PointCloud, DotCloud
│   ├── svg/             # SVGMobject, TexMobject, TextMobject
│   ├── geometry.py      # Dot, Line, Arrow, Circle, Rectangle, Polygon
│   ├── coordinate_systems.py  # Axes, NumberPlane, ComplexPlane
│   └── value_tracker.py
├── scene/               # Scene, InteractiveScene, FileWriter
├── shaders/             # GLSL vertex/fragment programs
└── utils/               # bezier, space_ops, color, rate_functions, tex
```

## Dependencies

| Category | Packages |
|----------|----------|
| **GPU rendering** | ModernGL, PyOpenGL, Pyglet |
| **Math** | NumPy, SciPy, SymPy |
| **Vector/Text** | svg-elements, skia-pathops, manimpango |
| **Image/Media** | Pillow, pydub, FFmpeg (external) |
| **Config** | PyYAML, addict |
| **Interactive** | IPython, rich, tqdm |

## Design Patterns

- **Subclass-based scenes**: users create `class MyScene(Scene)` and override `construct()`
- **Updater pattern**: `mobject.add_updater(fn)` for reactive animations tied to frame time
- **Animation composition**: `AnimationGroup` (parallel), `Succession` (sequential), `LaggedStart` (staggered)
- **Shader-based rendering**: each Mobject type has associated GLSL shaders; rendering is GPU-first via ModernGL
- **Config cascade**: defaults → user YAML → CLI flags, merged into a single dict
