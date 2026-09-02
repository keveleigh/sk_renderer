# Running Vulkan validation against sk_renderer

`example/main.c` always creates a real window/swapchain (no headless mode),
and on this dev machine's WSL install there is no functional real-GPU Vulkan
ICD (`vulkaninfo` fails with `ERROR_INCOMPATIBLE_DRIVER` for both the default
loader search and an explicit `virtio_icd` override). Falling back to
lavapipe (`lvp_icd`, software rasterizer) for the windowed example crashes
inside the llvmpipe JIT before any validation output is produced.

The reliable way to exercise `sk_renderer`'s Vulkan validation is through
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
