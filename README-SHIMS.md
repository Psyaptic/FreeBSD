# freebsd-linuxulator-shims

Userland shims and stubs for running Linux applications under FreeBSD's Linuxulator — the pieces that live in `LD_PRELOAD` and in `$PATH`, rather than in the kernel.

Companion to [freebsd-linuxulator-patches](../freebsd-linuxulator-patches), which carries the kernel-side fixes. The split is deliberate:

- **Kernel patches** are for divergences from Linux semantics that userland cannot see or intercept — an errno produced inside a syscall, a privilege check in `prison_priv_check()`.
- **Shims** are for everything else: missing `/proc` nodes, Linux kernel features FreeBSD will never grow (user namespaces), and workarounds you want available *now*, on a stock kernel, without a rebuild.

Where both exist for the same problem, the kernel patch is the real fix and the shim is the fallback. `libschedfix.so` and kernel patch 0001 are exactly this pair.

## Contents

| Component | Type | Purpose |
|---|---|---|
| `libschedfix.so` | `LD_PRELOAD` | Neutralises `sched_setscheduler()` failures inside a jail |
| `libmapfilesfix.so` | `LD_PRELOAD` | Synthesises `/proc/self/map_files` from `/proc/self/maps` |
| `bwrap` | executable stub | Drop-in bubblewrap replacement that skips namespace setup |
| `lsu/` | patch series | Local delta on top of linuxulator-steam-utils |

---

## `libschedfix.so`

**Problem.** On a stock kernel, `sched_setscheduler(2)` returns `EPERM` inside a jail for *every* policy, including the ordinary time-sharing policy that Linux grants unprivileged processes freely. Callers that treat the failure as fatal — Unreal Engine's task graph, several Chromium-derived worker pools, some JVM thread factories — die during startup.

**Approach.** Intercepts `sched_setscheduler`, `sched_setparam` and `pthread_setschedparam`. Requests for non-realtime policies are reported as successful. Where the request carries a meaningful priority, it is translated to a `setpriority()` call so the intent is at least partially honoured. `sched_getscheduler` and `sched_getparam` are also intercepted and read back from a small per-thread table, so callers that verify their own writes see what they expect rather than a contradiction.

**Caveats.** Requests for the realtime policies become no-ops. Anything genuinely depending on `SCHED_FIFO` latency guarantees gets ordinary time-sharing instead and will behave accordingly — this matters for low-latency audio paths more than for compute worker pools. If you need real scheduling behaviour rather than a silenced error, apply kernel patch 0001 and drop this shim.

---

## `libmapfilesfix.so`

**Problem.** `linprocfs` does not implement `/proc/self/map_files/`, the directory of `<start>-<end>` symlinks pointing at the backing file of each file-backed mapping. Consumers that walk it to enumerate loaded modules get `ENOENT` and either abort or fall into a degraded path.

**Approach.** Intercepts `open`, `openat`, `readlink`, `readlinkat`, `opendir` and `readdir` for paths under `/proc/self/map_files`. Entries are synthesised on demand by parsing `/proc/self/maps`, which `linprocfs` *does* provide — every file-backed line there carries both the address range and the pathname, which is precisely the information the symlink encodes. Non-file-backed mappings are omitted, matching Linux.

**Caveats.** Only `/proc/self` and `/proc/<own pid>` are handled; cross-process introspection is not. The synthesised symlinks resolve by path, so a mapping whose backing file has since been unlinked or replaced resolves differently than it would on Linux, where the symlink holds a direct reference to the inode.

---

## `bwrap` (stub)

**Problem.** [bubblewrap](https://github.com/containers/bubblewrap) builds its sandbox out of Linux user namespaces, mount namespaces and `pivot_root`. The Linuxulator implements none of these and is unlikely to. Anything that shells out to `bwrap` — Steam's pressure-vessel runtime, Flatpak, several game launchers — fails at the first `unshare()`.

**Approach.** A stub binary installed as `bwrap` ahead of the real one in `$PATH`. It parses the bubblewrap command line, discards the sandboxing flags, honours the handful that affect the child's observable environment (`--setenv`, `--unsetenv`, `--chdir`, `--args`), locates the target command after the argument list, and `execvp()`s it directly in the current process.

> [!WARNING]
> **This provides no isolation of any kind.** Every filesystem bind, namespace and privilege-dropping request in the bwrap command line is silently ignored, and the target runs with the full ambient authority of the calling process. The jail is your security boundary; the stub assumes you have already decided that the jail is sufficient. Do not install this on a system where you are relying on bubblewrap for containment.

**Caveats.** `--bind` and friends are accepted and ignored rather than emulated, so a program that genuinely depends on a path appearing at a different location inside the sandbox will not find it. In practice this is worked around by arranging the equivalent mounts in the jail's `fstab` — nullfs handles most cases.

---

## `lsu/` — linuxulator-steam-utils integration

[linuxulator-steam-utils](https://github.com/shkhln/linuxulator-steam-utils) (LSU) is the established toolkit for running Steam under the Linuxulator, and provides `steamfix.so`, `pathfix.so`, `webfix.so` and `fakeudev.so`. This directory carries a local patch series on top of it rather than a fork.

The delta covers:

- **`pipe2()` and `setsockopt()` quirks** — flag and option combinations that Steam's IPC layer issues and that the Linuxulator rejects, papered over so the client's socket setup completes.
- **pressure-vessel bypass** — replacing the `_v2-entry-point` script so Steam's container runtime is skipped entirely and games run against the host jail's libraries. Necessary because pressure-vessel is itself a bubblewrap consumer; combined with the `bwrap` stub above it can be made to limp, but bypassing it is far more reliable.
- **`/proc/self/fd` magic links** — requires `fdescfs` mounted with the `linrdlnk` option inside the jail, so that `readlink()` on those entries returns a path rather than a bare descriptor number. This is a mount option, not a shim, but Steam breaks without it so it is documented here.

Apply against a checked-out LSU tree:

```sh
git clone https://github.com/shkhln/linuxulator-steam-utils.git
cd linuxulator-steam-utils
for p in /path/to/freebsd-linuxulator-shims/lsu/patches/*.patch; do
    patch -p1 < "$p"
done
```

---

## Building

The shims are Linux ELF objects. **They must be built inside the Linux jail with the Linux toolchain** — building them with FreeBSD's `cc` produces FreeBSD ELF objects that the Linux dynamic linker will refuse, and the resulting failure mode (`LD_PRELOAD cannot be preloaded ... ignored`) is quiet enough to waste an afternoon.

From inside the jail:

```sh
apt install build-essential gcc-multilib
make            # builds amd64
make ABI=i386   # builds the 32-bit variants
make install    # installs to /usr/local/lib/linuxshims
```

Both ABIs are needed if you are running Steam: the client itself is 32-bit and spawns 64-bit children, and a single `LD_PRELOAD` list has to satisfy both.

## Loading

Preload by bare soname, not absolute path, and let `LD_LIBRARY_PATH` resolve the right ABI per process:

```sh
export LD_LIBRARY_PATH=/usr/local/lib/linuxshims/\$LIB
export LD_PRELOAD="libschedfix.so libmapfilesfix.so"
```

The literal `$LIB` is expanded by the dynamic linker to `lib` or `lib64` depending on the ABI of the process being started — escape it so the shell leaves it alone. A shim that is missing for one ABI is skipped with a warning rather than being fatal, so a partial install degrades quietly.

Set `SHIM_DEBUG=1` to log every interception to stderr, with the arguments and the value returned.

## Compatibility

| Application | Shims required | Notes |
|---|---|---|
| Unreal Engine 5.8 editor | `schedfix` | Or kernel patch 0001 instead |
| Steam client | LSU series, `bwrap` stub | Both ABIs needed |
| Chromium / Electron | `mapfilesfix` | Also needs `lx-proc-sys-kernel` nullfs overlay |
| Flatpak-packaged apps | `bwrap` stub | Isolation is lost; see warning above |

## Tested on

- FreeBSD 15.1-CURRENT, custom kernel config `NERVE`, ZFS root
- Intel i9-10980XE (36 threads), 32 GB RAM
- NVIDIA RTX 3090 Ti, driver 595.80
- Debian Trixie Linuxulator jail

## Licence

<!-- Confirm before publishing: LSU is GPL-licensed, and the lsu/ patch series is a derived work. -->
Shims and the `bwrap` stub: BSD-2-Clause. The `lsu/` patch series follows the licence of linuxulator-steam-utils.

## Contributing

New shims are welcome, with a preference for ones that carry a clear statement of what breaks without them and why the fix belongs in userland rather than in the kernel. If a divergence *can* be fixed properly in `sys/compat/linux`, it probably should be — send it to the patches repository instead.
