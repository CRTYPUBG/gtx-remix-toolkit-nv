# GTX REMIX

GTX-class post stack that approximates RTX-style lighting with **Vulkan compute**, **screen-space techniques**, and **temporal accumulation**. There is **no** hardware ray tracing, **no** DXR/VK ray tracing pipelines, and **no** DLSS dependency.

## Layout

| Path | Role |
|------|------|
| `engine/` | Native C++ prototype: modular compute dispatch, hook stub, JSON/YAML ingest |
| `shaders/` | GLSL `.comp` modules (SSR, SSGI, temporal denoise, upscale, pseudo bounce pass) |
| `config/` | `render_pipeline.json` (ordered passes + resolution), `feature_flags.yaml`, merged `render_pipeline.resolved.json` |
| `presets/` | `low_end_gtx1060.yaml` and other quality envelopes |
| `core/` | Python tooling host (`python -m core.engine …`) |
| `tools/` | Config merge/validation helpers |

## Build (Windows)

1. Install the [LunarG Vulkan SDK](https://vulkan.lunarg.com/) so `Vulkan_INCLUDE_DIR` resolves in CMake and `glslangValidator` is on `PATH`.
2. Configure and build:

```powershell
cmake -S . -B build
cmake --build build --config Release
```

If the SDK is missing, configure with `-DGTXREMIX_ENABLE_VULKAN=OFF` to build `gtxremix_toolcheck.exe` only (config validation, no GPU dispatch).

Run the GPU prototype (after a full build):

```powershell
.\build\Release\gtxremix_engine.exe --pipeline config\render_pipeline.json --flags config\feature_flags.yaml --shaders build\shaders\
```

Run the host-side checker without Vulkan:

```powershell
.\build\Release\gtxremix_toolcheck.exe --pipeline config\render_pipeline.json --flags config\feature_flags.yaml
```

## Tooling (Python)

```powershell
py -3 -m pip install -r requirements-tooling.txt
py -3 tools\validate_render_config.py
py -3 tools\merge_render_config.py
py -3 -m core.engine validate
py -3 -m core.engine merge
```

## Architecture

`Game → Hook Layer → Compute Pipeline → Post Processing → Output Frame`

See `ARCHITECTURE.md` for the module map. Feature flags are mandatory for every optional stage (`config/feature_flags.yaml`).

## D3D9 Hook Overlay (F9)

The D3D9 hook includes a minimal fullscreen overlay pass (tint / gamma MVP) plus an in-game menu:

- Menu toggle: `F9`
- Overlay default: enabled
- Override overlay default: set `GTXREMIX_OVERLAY=0` to start disabled (you can re-enable from the menu).

Optional tuning (floats) for visibility/testing:

- `GTXREMIX_OVERLAY_GAMMA` (default `1.15`)
- `GTXREMIX_OVERLAY_TINT_R`, `GTXREMIX_OVERLAY_TINT_G`, `GTXREMIX_OVERLAY_TINT_B` (defaults `1.0`, `0.92`, `0.82`)

Note: some D3D9 games present via `IDirect3DSwapChain9::Present` (not `IDirect3DDevice9::Present`), so the hook installs a swapchain present hook too.

## Roadmap: DX7/8/10/11/12

Current implementation targets D3D9 injection (`d3d9.dll` proxy). To support DX7/8/10/11/12, we need additional proxy/hook layers:

- DX7: `ddraw.dll` + `d3dim.dll` proxy (capture final blit/present)
- DX8: `d3d8.dll` proxy (hook `Present` equivalent)
- DX10/11/12: `dxgi.dll` hook (hook `IDXGISwapChain::Present` / `ResizeBuffers`)
