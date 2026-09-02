# Running Vulkan validation against sk_renderer

`example/main.c` always creates a real window/swapchain (no headless mode),
and on this dev machine's WSL install there is no functional real-GPU Vulkan
ICD (`vulkaninfo` fails with `ERROR_INCOMPATIBLE_DRIVER` for both the default
loader search and an explicit `virtio_icd` override), so it always falls
back to lavapipe (`lvp_icd`, software rasterizer).

Despite that, WSLg *is* functional here — `sk_app`'s Wayland backend
successfully creates a real window (`DISPLAY`/`WAYLAND_DISPLAY` both work,
`libdecor` draws frames) — so the windowed example gets much further than
this doc used to imply: full device init, extension/feature logging, the
first scene switch, and render-target creation all complete, and real
validation errors do surface (this is how the Google-extension issue below
was found) before anything crashes. The crash that remains is a `SIGSEGV`
inside the llvmpipe JIT itself (see "Known limitation" further down) — it
happens later, on first shader/pipeline creation for the first scene, not
immediately at startup. See "Running sk_renderer's own C example directly"
below for how to drive it under validation, and why it's a useful
*complement* to the .NET harness rather than a redundant path.

The reliable way to exercise `sk_renderer`'s Vulkan validation to
**completion** (i.e. past the llvmpipe JIT crash) is still through
**StereoKit's own .NET test suite, in headless mode**, which never creates a
window/swapchain and so never hits that crash path:

```bash
export WSL_UTF8=1
wsl -d Ubuntu-22.04 -- bash -lc '
cd /home/kurtis/StereoKit_repro/Examples/StereoKitTest
export PATH="$PATH:/home/kurtis/.dotnet"
export VK_LAYER_PATH=/home/kurtis/vklayer_local/extracted/usr/share/vulkan/explicit_layer.d
export LD_LIBRARY_PATH=/home/kurtis/vklayer_local/extracted/usr/lib/x86_64-linux-gnu:$LD_LIBRARY_PATH
export VK_INSTANCE_LAYERS=VK_LAYER_KHRONOS_validation
setsid dotnet run -c Debug -- -headless -test > /home/kurtis/test_validation.log 2>&1 < /dev/null
echo "exit code: $?"
'
```

Notes:
- The validation layer used here is a standalone sideload extracted to
  `/home/kurtis/vklayer_local/extracted` — there is no apt-installed
  `vulkan-validationlayers` package in this WSL image (`dpkg -l` confirms
  it's absent), so `VK_LAYER_PATH`/`LD_LIBRARY_PATH` must point at that
  sideload explicitly.
- No `VK_ICD_FILENAMES` override is set — the loader's default ICD search
  is what's used. Since real-GPU ICDs aren't functional here, this run
  still ends up on lavapipe, but `-headless` avoids the window/swapchain
  code path that otherwise crashes it.
- Grep the resulting log for the specific VUID/message you're chasing, e.g.:
  `grep -c "QueryNotReset" test_validation.log`,
  `grep -c "vkDestroyImage-image-01000" test_validation.log`.
- `[SK error] ... Validation Error: [ <VUID> ]` lines are real failures.
  `Validation Performance Warning` / `Validation Information` lines
  (e.g. `UNASSIGNED-CoreValidation-Shader-OutputNotConsumed`,
  `UNASSIGNED-cache-file-error`) are pre-existing and unrelated to any
  renderer bug — don't confuse them with real errors.
- The dotnet process's stdout redirect loses its last partially-buffered
  line when the process exits (harmless truncation of the final log line,
  seen even on a clean pre-fix run) — not a sign of a crash.
- **Blind spot**: this harness only ever compiles/creates StereoKit's own
  built-in shaders. Some validation issues are shader-*content*-dependent —
  e.g. `VUID-VkShaderModuleCreateInfo-pCode-04147` (missing
  `VK_GOOGLE_hlsl_functionality1`/`decorate_string` device extensions) only
  fires for shaders whose HLSL uses `StructuredBuffer`/`RWStructuredBuffer`-
  style constructs, which glslang's HLSL front-end stamps with a
  reflection-only `SPV_GOOGLE_*` SPIR-V extension. None of StereoKit's own
  `.sks` shaders do this (confirmed by grepping the compiled binaries for
  `SPV_GOOGLE`), but plenty of `sk_renderer`'s own example shaders do
  (`pbr.hlsl` and ~20 others — compute/GI/particle work in particular). This
  harness cannot catch that class of bug; use the standalone C example below
  for that kind of coverage.

## Iterating on sk_renderer source without republishing a release

StereoKit's CMake normally fetches sk_renderer via CPM from a pinned
release tag. To test local `sk_renderer` source changes against the WSL
build, patch the CPM source cache directly rather than re-fetching:

```
/home/kurtis/StereoKit_repro/.deps_cache/sk_renderer/<hash>/sk_renderer/vk/
```

(the `<hash>` directory name is CPM's content hash for the pinned
version — find it with `ls /home/kurtis/StereoKit_repro/.deps_cache/sk_renderer/`).
This is the actual source CMake compiles from (confirmed identical in
layout to the Windows-side `sk_renderer` checkout). After editing files
there, rebuild with the existing native preset — this only recompiles the
changed `sk_renderer` objects and relinks, it does not reconfigure:

```bash
wsl -d Ubuntu-22.04 -- bash -lc 'cd /home/kurtis/StereoKit_repro && cmake --build --preset Linux_x64_Debug_Fast'
```

Alternative (untested) for a more permanent local-source workflow: set
`-DSK_LOCAL_SK_RENDERER=ON` in the StereoKit CMake config, which expects a
sibling `../sk_renderer` checkout instead of the CPM-fetched one (see
`CMakeLists.txt`'s `SK_LOCAL_SK_RENDERER` option) — not currently set up on
this machine (no `/home/kurtis/sk_renderer` exists).

## Running sk_renderer's own C example directly

This repo's own standalone example (`example/`, built from this checkout's
`build-linux/` tree, *not* via StereoKit's CPM cache) exercises a much wider
variety of shader content than StereoKit's headless harness — it's what
caught the Google-extension VUID above. Build just the `sk_renderer_test`
target (the `example_xr` target fails to configure here — missing
`xcb/glx.h` — but that's unrelated and out of scope to fix):

```bash
wsl -d Ubuntu-22.04 -- bash -lc '
cd /home/kurtis/sk_renderer_repro/build-linux
cmake --build . --target sk_renderer_test -j$(nproc)
'
```

Run it under validation the same way as the .NET harness, using
`-testall` to cycle every scene and `-frames N` to cap runtime. Use
`stdbuf -oL -eL` and a `timeout`, and redirect stdin from `/dev/null` (it's
still a real windowed app):

```bash
wsl -d Ubuntu-22.04 -- bash -lc '
cd /home/kurtis/sk_renderer_repro/build-linux
export VK_LAYER_PATH=/home/kurtis/vklayer_local/extracted/usr/share/vulkan/explicit_layer.d
export LD_LIBRARY_PATH=/home/kurtis/vklayer_local/extracted/usr/lib/x86_64-linux-gnu:$LD_LIBRARY_PATH
export VK_INSTANCE_LAYERS=VK_LAYER_KHRONOS_validation
timeout 40 stdbuf -oL -eL ./example/sk_renderer_test -testall -frames 15 > ~/example_test.log 2>&1 < /dev/null
echo "exit code: $?"
'
```

`stdbuf -oL -eL` matters here specifically because this example *does*
crash (see below): a fully-buffered redirect can silently drop the last
buffered chunk when the process dies, making the log's apparent last line
misleading about where the crash actually happened.

### Known limitation: llvmpipe JIT SIGSEGV blocks full scene coverage

Even with the Google-extension fix in place and zero validation errors,
`-testall` still crashes partway through the very first scene ("Meshes") —
right after the "Switched to scene" / "Render target: ..." log lines, on
first shader/pipeline creation. This is **not** a validation-catchable
issue: it's a plain `SIGSEGV` on one of llvmpipe's own JIT worker threads
(observed thread name `llvmpipe-1`), confirmed via:

```bash
wsl -d Ubuntu-22.04 -- bash -lc '
cd /home/kurtis/sk_renderer_repro/build-linux
gdb -q -batch -ex run -ex bt --args ./example/sk_renderer_test -testall -frames 15 > ~/example_gdb.log 2>&1
'
```

— the backtrace is unsymbolized/garbage (`0x0` frames), as expected for a
crash inside JIT-generated code with no debug info. This is most likely a
Mesa/lavapipe bug or a shader construct it can't JIT, not an `sk_renderer`
bug, and hasn't been root-caused further. Net effect: the standalone
example is useful for catching shader-content/extension issues in whatever
scene loads first, but **cannot** currently be used to reach later scenes
(e.g. `scene_lifetime_stress.c`, scene 12) — the .NET headless harness
remains the only way to validate those on this machine.

