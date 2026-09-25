# freebsd-jail-gaming

Host-side configuration for a Debian Linuxulator jail on FreeBSD 15.1 that runs GPU-accelerated Linux applications — Steam and Proton titles, Minecraft, Discord, OBS Studio, and the Unreal Engine 5.8 editor — against an NVIDIA GPU.

No code here. This repository is the scaffolding: the jail definition, the mount table, the devfs ruleset, and the small amount of glue needed to make `linprocfs` and `linsysfs` look enough like Linux for the applications above to start.

## Where this sits

Three repositories, applied in this order:

1. **`freebsd-jail-gaming`** *(this one)* — configuration. Start here; most things work without touching the other two.
2. **[`freebsd-linuxulator-shims`](../freebsd-linuxulator-shims)** — `LD_PRELOAD` shims and the `bwrap` stub, for applications that still fail once the jail is up.
3. **[`freebsd-linuxulator-patches`](../freebsd-linuxulator-patches)** — kernel patches, for the two divergences that userland cannot reach.

## Files

| File | Installs to | Purpose |
|---|---|---|
| `debian.conf` | `/etc/jail.conf.d/debian.conf` | Jail definition |
| `fstab.debian` | `/etc/fstab.debian` | Jail mount table |
| `devfs.rules.d/gaming` | appended to `/etc/devfs.rules` | Ruleset 9 — GPU, audio and input device nodes |
| `lx-proc-sys-kernel` | `/usr/local/sbin/` | Generates and mounts a `/proc/sys/kernel` overlay |
| `loader.conf.d/linux` | `/boot/loader.conf.d/` | Module loading |
| `sysctl.d/linuxulator` | `/etc/sysctl.conf` fragment | Host tunables |

---

## `debian.conf`

A standard `jail.conf(5)` fragment, with the Linux-specific parts being:

**`linux.osrelease`** — sets the kernel version the jail reports. Debian Trixie's glibc refuses to run if this is below its minimum, and several applications gate features on it. This value is also what the `linprocfs` conformance work in the patches repository targets, so keep the two consistent.

**`allow.mount.linprocfs` / `allow.mount.linsysfs` / `allow.mount.fdescfs`** — required for the mount table below to be established, together with `enforce_statfs = 1` so the jail can see mounts beneath its root.

**`sysvmsg` / `sysvsem` / `sysvshm = new`** — Steam and X11 clients use SysV shared memory. Without these the client starts and then hangs with no useful diagnostic.

**`exec.start = "/bin/true"`** — the jail runs no Linux init. Applications are launched with `jexec` from the host.

**`exec.stop`** — deliberately *empty*. The obvious value here is a SysV runlevel call (`/etc/init.d/rc 0`), and it is wrong: with no init running there is nothing to shut down, and the call blocks until it times out. Jail stop goes from tens of seconds to immediate once it is removed.

**`exec.prestart`** — invokes `lx-proc-sys-kernel` (below) so the overlay is in place before anything runs.

---

## `fstab.debian`

Mounted in file order, which matters — `linprocfs` must exist before the overlay lands on top of it.

| Mount | Notes |
|---|---|
| `devfs` on `/dev` | Ruleset 9, see below |
| `tmpfs` on `/dev/shm` | Mode `1777`. Chromium and Steam both fail without a writable, world-writable `/dev/shm` |
| `fdescfs` on `/dev/fd` | **With the `linrdlnk` option.** This makes `readlink()` on `/proc/self/fd` entries return a path rather than a descriptor number, which is what Linux does and what Steam expects |
| `linprocfs` on `/proc` | |
| `linsysfs` on `/sys` | |
| `nullfs` for the user's home | Keeps game libraries and project files on the host filesystem rather than inside the jail image |
| `nullfs` for the Wayland/X11 socket directory | So jailed clients can reach the host compositor |

The `linrdlnk` option on `fdescfs` is the single easiest thing to omit and the hardest to diagnose from the resulting failure.

---

## devfs ruleset 9

Built by including the standard hide-all base and then unhiding, specifically:

- `nvidia*`, `nvidiactl`, `nvidia-modeset`, `nvidia-uvm`, `nvidia-uvm-tools` — the GPU
- `dri`, `dri/*`, `drm`, `drm/*` — DRM render nodes
- `dsp*`, `sndstat` — audio, for the OSS bridge
- `input/*`, `usb/*` — controllers and HID devices

> [!WARNING]
> Exposing GPU device nodes to a jail hands the jailed process a direct ioctl path into the graphics driver. A driver bug reachable that way is a host compromise, and the jail boundary does not help. This configuration is appropriate for a single-user workstation running software you have chosen to trust. It is not appropriate for a shared machine or for running untrusted binaries.

Ruleset number 9 is arbitrary but must match `devfs_ruleset` in `debian.conf` and the `ruleset=` in `fstab.debian`. Reload with `service devfs restart` after editing.

---

## `lx-proc-sys-kernel`

**Problem.** Chromium-derived applications — Discord, Electron apps, the Steam client's web views, CEF in general — read a set of files under `/proc/sys/kernel` during child process startup: pointer restriction and ASLR settings, PID and thread limits, capability bounds. `linprocfs` implements only part of this tree. A missing file causes the zygote to abort, and the parent reports a generic child-launch failure that says nothing about the cause.

**Approach.** The script builds a real directory on the host populated with the needed files and plausible Linux values, then `nullfs`-mounts it over `<jailroot>/proc/sys/kernel`. Because it is a real filesystem, everything reads correctly and nothing has to be implemented in the kernel.

It is idempotent — safe to re-run, and it will not stack mounts. Invoked from `exec.prestart` in `debian.conf`, but can be run by hand against a running jail while debugging.

**Note.** This overlaps with the `linprocfs` conformance work in the patches repository. Where a field is properly implemented in `linprocfs`, the overlay for it becomes redundant and can be dropped. The overlay exists because it works today on a stock kernel.

---

## Bootstrapping

```sh
# Host: modules and tunables
sysrc linux_enable="YES"
service linux start
sysctl -f /etc/sysctl.conf

# Create the jail root
zfs create -o mountpoint=/jails/debian zroot/jails/debian
pkg install debootstrap
debootstrap trixie /jails/debian http://deb.debian.org/debian

# Install configuration
install -m 644 debian.conf   /etc/jail.conf.d/debian.conf
install -m 644 fstab.debian  /etc/fstab.debian
install -m 755 lx-proc-sys-kernel /usr/local/sbin/
cat devfs.rules.d/gaming >> /etc/devfs.rules
service devfs restart

service jail start debian
jexec debian /bin/bash
```

---

## GPU setup inside the jail

This is the part that consumes the most time, and the failure modes are the least informative.

**The driver userland must be the Linux build, at the exact version of the host FreeBSD kernel driver.** Copying FreeBSD's NVIDIA libraries into the jail produces FreeBSD ELF objects that the Linux loader rejects; installing Debian's packaged NVIDIA driver produces a version mismatch against the host kernel module. Obtain the Linux `.run` installer for the matching version, extract it with `--extract-only`, and install the libraries by hand.

Beyond the libraries, four pieces are needed:

- **GLVND vendor JSON** at `/usr/share/glvnd/egl_vendor.d/` so EGL dispatches to NVIDIA
- **Vulkan ICD JSON** at `/usr/share/vulkan/icd.d/`
- **GBM backend symlink** — `gbm/nvidia-drm_gbm.so` pointing at `libnvidia-allocator.so.<version>`. Without it, GBM finds no backend and GLX falls back to single-buffered visuals, which most applications treat as no acceleration at all
- **`/sys/module/nvidia/initstate` overlay** — `linsysfs` does not provide it, and NVML reads it to confirm the driver is live. A single file containing `live`, nullfs-mounted into place

Environment for jailed clients:

```sh
export GBM_BACKEND=nvidia-drm
export __GLX_VENDOR_LIBRARY_NAME=nvidia
```

Verify with `vulkaninfo | head` and `glxinfo -B` before trying to launch anything real. If Vulkan reports the GPU and GLX reports a double-buffered visual, the hard part is done.

## Audio

Sound is bridged out of the jail to the host's OSS device rather than running a sound server inside it: a PulseAudio sink inside the jail is captured with `pacat`, piped through `ffmpeg` for format conversion, and written to `/dev/dsp`. Latency is acceptable for games and unremarkable for media playback; it is not suitable for low-latency music production.

## D-Bus

Applications expecting a session bus should be launched under `dbus-run-session` rather than relying on one being present. Discord, OBS and anything using portals will otherwise start with subtly broken behaviour — missing notifications, non-functional screen capture — rather than failing outright.

## Host tunables

`kern.elf64.pie_base` must be raised for `wine-preloader` to work, which means every Proton title depends on it. The default base collides with the address range the preloader reserves.

## Application status

| Application | Works | Requires |
|---|---|---|
| Steam client | Yes | LSU series, `bwrap` stub |
| Proton titles | Yes | `pie_base` sysctl |
| Minecraft (PrismLauncher) | Yes | — |
| Discord | Yes | `lx-proc-sys-kernel` |
| OBS Studio | Partial | Screen capture via portal; webcam needs an ffmpeg path |
| Unreal Engine 5.8 editor | Yes | `libschedfix.so` or kernel patch 0001 |
| Roblox (Sober) | Partial | `linprocfs` status field conformance |

## Tested on

- FreeBSD 15.1-CURRENT, custom kernel config `NERVE`, ZFS root
- Intel i9-10980XE (36 threads), 32 GB RAM, ASUS ROG RAMPAGE VI APEX
- NVIDIA RTX 3090 Ti, driver 595.99.02, Vulkan 1.4.360
- Debian Trixie jail
- Hyprland 0.56.2 on the host, with `no_hardware_cursors = true`

That last setting is not optional on NVIDIA — hardware cursors panic the host kernel.

## Troubleshooting

| Symptom | Likely cause |
|---|---|
| Jail takes ~30s to stop | `exec.stop` is set; empty it |
| Steam hangs after splash | `sysvshm` not set to `new`, or `linrdlnk` missing on `fdescfs` |
| Electron app child process fails | `lx-proc-sys-kernel` overlay not mounted |
| `glxinfo` shows no double-buffered visual | GBM backend symlink missing |
| NVML reports no driver | `/sys/module/nvidia/initstate` overlay missing |
| `LD_PRELOAD ... ignored` | Shim built with FreeBSD `cc` instead of the jail toolchain |

## Licence

BSD-2-Clause.
