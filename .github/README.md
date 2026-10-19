# PanVK kbase

Mesa **Panfrost/PanVK** running on Arm's proprietary Mali **kbase** kernel driver.

This is a fork of [Mesa](https://gitlab.freedesktop.org/mesa/mesa). It adds a **kbase backend** to
Panfrost/PanVK so Mesa's Mali drivers run on top of the vendor `kbase` driver
(`/dev/mali*`, r32p0–r44p0) — the driver shipped on most Android phones — instead of the upstream
`panfrost`/`panthor` DRM drivers.

Vulkan (PanVK) is the focus. OpenGL on kbase is gated off.

> **Status: experimental.** Only the devices listed below are confirmed. This is not a claim of
> support for every Mali GPU or kbase kernel release.

## Confirmed on

- **Google Pixel 7** — Mali-G710 MC7 (Google Tensor G2), Valhall **CSF**, arch 10
- **Mali-G925** (reported as G725 by the tested platform), 5th-gen **CSF**
- **Mali-G57**, Valhall **Job Manager** (from the original base)

Both kbase flavours are handled: **CSF** (arch ≥ 10) and **Job Manager** (arch ≤ 9).

## What it does

- **kbase KMD backend** in the shared `pan_kmod` layer
  (`src/panfrost/lib/kmod/kbase_kmod.c`), selected with `-Dpanfrost-kmds=kbase`. Every
  kbase-specific change is behind `HAVE_PAN_KMOD_KBASE`, so `-Dpanfrost-kmds=panfrost,panthor`
  still builds cleanly.
- **No DRM fd.** kbase uses a VERSION_CHECK → SET_FLAGS → tracking-page mmap handshake, and all
  other ioctls return `-EPERM` until it completes. The CSF and JM flavours use different
  VERSION_CHECK ioctl numbers and are probed in turn. The UAPI lives in
  `include/drm-uapi/mali_kbase_ioctl.h`.
- **SAME_VA memory model.** For 64-bit clients the kernel forces SAME_VA on non-executable
  allocations; `MEM_ALLOC` returns a cookie and mmap-ing the fd at that cookie gives CPU VA == GPU
  VA. The backend maps every BO at allocation time and keeps the mapping for the BO's lifetime.
- **Command submission (CSF).** panvk drives userspace-owned CS rings
  (`csf/panvk_vX_gpu_queue.c`, bound with `CS_QUEUE_REGISTER`/`CS_QUEUE_BIND`) and submits the
  same ring sequence the panthor kernel uses, then kicks the firmware. Synchronization is
  currently synchronous: submissions are CPU-waited on per-subqueue seqno cells.
- **kbase tiler heap.** Each Vulkan queue owns a kbase tiler heap (1 MiB chunks) that is renewed
  periodically; `PANVK_KBASE_HEAP_RENEW_INTERVAL` overrides the interval.
- **Device discovery.** DRM devices are probed first, then `/dev/mali0`..`/dev/mali7`
  (`PAN_KBASE_MAX_NODES`). Reachable from PanVK, from the Gallium pipe-loader, and as an fd-based
  fallback when `drmGetVersion()` fails on a kbase fd.
- **Android.** Runs under the Android system Vulkan loader using `VK_ANDROID_native_buffer`
  (gralloc) rather than implementing its own surface extension. This is the path consumed by the
  Copper Launcher PanVK driver manager.
- **GPU model table** covering Bifrost, Valhall (v9/v10) and 5th-gen Mali/Immortalis.
- **KRAID** (Mesa's Rust shader compiler) is available for Valhall and newer; opt in per stage
  with `PAN_USE_KRAID`.
- OpenGL on kbase is disabled until submission support lands; bypass with
  `PAN_EXPERIMENTAL_KBASE_GL=1`. Vulkan enumeration works without the gate.

## Limitations

- Synchronous submission only — no async fences / KCPU queues yet.
- Sparse binding (the `BIND` queue family) is a stub.
- `PAN_KMOD_VM_OP_MODE_ASYNC` and explicit-VA `vm_bind` are unsupported.
- Running kbase from a chroot/proot/DroidSpaces container can expose kernel-driver bugs and, on
  some devices, panic the kernel. See `docs/panvk-kbase.md`.

## Building

### Android (bionic, Copper driver)

Cross file `android-cross.txt` (`--target=aarch64-linux-android35`) plus the options in
`tools/fristonerr/build-options.txt`:

```sh
meson setup build-android-final --cross-file android-cross.txt $(cat tools/fristonerr/build-options.txt)
ninja -C build-android-final
# output: build-android-final/src/panfrost/vulkan/libvulkan_panfrost.so
```

It expects a host build of `mesa_clc` at the path set in `android-cross.txt`
(`build-host/src/util/mesa_clc`).

### Linux / DroidSpaces (packaged)

The release packages are built with the options in `.github/workflows/kbase-release.yml`
(`-Dplatforms=x11 -Dpanfrost-kmds=kbase,panthor -Dvulkan-drivers=panfrost -Dopengl=false`).
`kbase-debug.yml` is the aarch64 debug cross-check; CI artifact
`mesa-panvk-kbase-debug-aarch64.zip`.

## Documentation

- [`docs/panvk-kbase.md`](docs/panvk-kbase.md) — install, X11 WSI design, performance, troubleshooting
- [`docs/kbase-minecraft-fault-analysis.md`](docs/kbase-minecraft-fault-analysis.md) — tiler-heap fault analysis
- [`docs/kbase-triangle-hang-analysis.md`](docs/kbase-triangle-hang-analysis.md)
- [`docs/kbase-dvfs.md`](docs/kbase-dvfs.md)
- [`CLAUDE.md`](CLAUDE.md) — architecture notes for the kbase backend

## Base and history

| | |
|---|---|
| Upstream | Mesa (PanVK), MIT license |
| Direct base | [funnymdzz/mesa](https://github.com/funnymdzz/mesa) `main` @ `6598829019c0746aa8e473b4ae1c980cbfa6ea4b` |
| Our work | Everything that differs from that commit |

The full git history is included so every change can be checked against the base.

## Credits

| Who | Contribution |
|---|---|
| **Mesa / PanVK developers** (Collabora, Arm and contributors) | PanVK, the panfrost/panthor kernel interfaces, and the pan_kmod layer this backend plugs into |
| **funnymdzz** & **leegao** | The Mesa fork this started from; [bionic-vulkan-wrapper](https://github.com/leegao/bionic-vulkan-wrapper); advice |
| **Noysz** / [panvk-g99-jm](https://github.com/Noysz/panvk-g99-jm) | Valhall v9 / Job Manager groundwork |
| **Vtgamer998** (MESA-KMOD) | kbase kernel-interface work |
| **mexicanbr0auth** / [mesa-panvk-g57](https://github.com/mexicanbr0auth/mesa-panvk-g57) | PanVK/kbase work for Mali-G57 |
| **wonderkast02** / [panvk-g720-kbase-csf](https://github.com/wonderkast02/panvk-g720-kbase-csf) | Community PanVK-over-kbase work (Mali-G720, CSF) |
| **BossDrk** / [0x8055/panvk-g52-oppo-a38](https://github.com/0x8055/panvk-g52-oppo-a38) | Mali-G52 research and testing (stride fix) |
| **LukeValen** | Mali-G52 research |
| **Claude** (AI assistant by Anthropic) | Development help, debugging and code review |
| **Community testers** | Device reports and logs via Issues |

## License

MIT (same as Mesa). See the license headers in each file.
