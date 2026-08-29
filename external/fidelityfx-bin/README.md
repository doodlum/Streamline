# FidelityFX prebuilt VK backend

`amd_fidelityfx_vk.dll` (+ import lib/exp) is the FFX-API Vulkan backend that
`sl.fsr` resolves at runtime for FSR upscaling and frame generation. It is built
from `external/fidelityfx-sdk`, the FidelityFX-SDK submodule, which tracks
upstream v1.1.4 (`c6efa6b` — the last release before the 2.0 "Kits" restructure
removed `ffx-api/`) plus two committed Vulkan fixes:

- `05ef92d` **fix(vulkan): initialize scRGB luminance** — seeds a finite
  `minLuminance`/`maxLuminance` for an scRGB surface when the application
  supplies no static HDR metadata.
- `3822481` **fix(vulkan): initialize PQ luminance** — the same guard for an
  HDR10/PQ surface. Without it `RawRGBToLinear()` divides by a zero
  `MaxLuminance()`, `CalculateStaticContentFactor()` collapses to 0, and
  interpolated frames are presented with no UI composited onto them — the HUD
  strobes at half the presented rate. The DX12 backend sidesteps this by
  querying the monitor luminance range at swapchain creation; Vulkan has no
  equivalent output query.

One further patch is applied only at DLL-build time and is deliberately *not*
committed to the submodule:

- `../patches/ffx-api-try2-macro.patch` — wraps the `TRY2` macro in
  `ffx-api/src/ffx_provider.h` in `do { } while(0)` for macro hygiene.

## Rebuilding

```
cd external/fidelityfx-sdk
git apply ../patches/ffx-api-try2-macro.patch      # from the Streamline root
cd ffx-api && mkdir build && cd build
cmake .. -DFFX_API_BACKEND=VK_X64 -A x64
cmake --build . --config Release --parallel 4
```

Copy `ffx-api/bin/amd_fidelityfx_vk.{dll,lib,exp}` here, then revert the TRY2
patch so the submodule tree stays clean. (`BuildFfxApiDll.bat`, kept here, is
AMD's original driver script; it originally lived inside the vendored `ffx-api/`
tree and builds all three configurations.)

`sl.fsr` itself compiles against `external/fidelityfx-sdk/ffx-api/include` only,
which is byte-identical to upstream v1.1.4 — both fixes above are backend
implementation, so they affect DLL rebuilds, not the plugin build.
