# freebsd-linuxulator-patches

Kernel patches for FreeBSD's Linux ABI layer (the Linuxulator), extracted from the work of getting the **Unreal Engine 5.8 editor** running inside a Debian Linux jail on FreeBSD 15.0-CURRENT.

Each patch fixes a place where the emulated syscall path diverges from Linux semantics in a way that **cannot** be worked around from userland with an `LD_PRELOAD` shim — the divergence happens inside the kernel, before any userland code can see it.

## Patches

| # | File | Problem | Status |
|---|------|---------|--------|
| 0001 | `sys/compat/linux/linux_misc.c` | `sched_setscheduler(2)` returns `EPERM` inside a jail for non-realtime policies | Submitted upstream |
| 0002 | `sys/compat/linux/linux_file.c` | `mkdir("/")` returns `EISDIR` instead of Linux's `EEXIST` | Submitted upstream |

Patches are numbered in the order they were developed and are intended to be applied in that order, though they touch disjoint files and have no interdependency.

---

## 0001 — `linux_misc.c`: `sched_setscheduler()` inside a jail

**Symptom**

Any Linux process running in a jail that calls `sched_setscheduler(2)` — even to request the ordinary time-sharing policy — gets `EPERM`. Unreal Engine's task graph sets thread policy during worker pool startup, so the editor dies before reaching the first frame. The same failure shows up in Chromium-derived processes and in several Java runtimes.

**Root cause**

`linux_sched_setscheduler()` translates the Linux policy to FreeBSD's `rtprio` machinery, which gates on `PRIV_SCHED_SETPOLICY`. `prison_priv_check()` denies that privilege to jailed processes unconditionally, so the call fails regardless of which policy was requested.

This is stricter than Linux. On Linux, only the realtime policies (`SCHED_FIFO`, `SCHED_RR`, and the deadline policy) require `CAP_SYS_NICE`. `SCHED_OTHER`, `SCHED_BATCH`, and `SCHED_IDLE` are available to unprivileged processes, and a container that doesn't grant `CAP_SYS_NICE` still permits them.

**Fix**

Only require `PRIV_SCHED_SETPOLICY` for the realtime policies. Non-realtime policy changes are permitted without it, matching Linux behaviour and leaving the realtime path exactly as strict as it was.

**Open question for reviewers**

An alternative shape would be a new jail parameter (`allow.sched` or similar) that re-enables the full range of policies inside a jail, leaving the current default untouched. That is more configurable but adds jail ABI surface. The patch as written takes the narrower view that the current behaviour is simply not Linux-conformant for the non-realtime cases, so no new knob is warranted.

---

## 0002 — `linux_file.c`: `mkdir("/")` errno

**Symptom**

Unreal Engine's Derived Data Cache fails to initialise. The editor logs a DDC path creation failure and falls back to a degraded mode or aborts, depending on the DDC graph configuration.

**Root cause**

UE's recursive directory creation walks a path from the root down, calling `mkdir()` on each component and treating `EEXIST` as benign. On the first component the path is `/`, which already exists.

Linux returns `EEXIST` for `mkdir("/")`. FreeBSD's `kern_mkdirat()` returns `EISDIR` for the root vnode. UE has no case for `EISDIR`, so it treats it as a hard failure and unwinds.

Because the errno is produced inside the kernel and returned directly by the syscall, no `LD_PRELOAD` interposition on `mkdir()` fixes it for statically-linked or direct-syscall callers.

**Fix**

Remap `EISDIR` to `EEXIST` in the Linuxulator's `mkdir`/`mkdirat` translation. Linux's `mkdir(2)` never returns `EISDIR` for any input, so this is a translation-layer correction rather than a behaviour change — the mapping is exactly what the compat layer exists to do.

---

## Repository layout

```
patches/
  0001-linux_misc-relax-sched_setscheduler-privilege.patch
  0002-linux_file-map-EISDIR-to-EEXIST-for-mkdir.patch
scripts/
  apply.sh          # applies every patch in order against $SRCDIR
tests/
  sched_policy.c    # reproducer for 0001
  mkdir_root.c      # reproducer for 0002
```

## Applying

Patches are unified diffs generated against the FreeBSD `src` tree root, so `-p1` from `/usr/src`.

```sh
git clone https://git.FreeBSD.org/src.git /usr/src   # if you don't have it
cd /usr/src

for p in /path/to/freebsd-linuxulator-patches/patches/*.patch; do
    patch -p1 < "$p"
done

make -j"$(sysctl -n hw.ncpu)" buildkernel KERNCONF=GENERIC
make installkernel KERNCONF=GENERIC
reboot
```

To back them out, re-run with `patch -R -p1`.

## Verifying

After rebooting into the patched kernel, from inside a Linux jail:

```sh
cc -o /tmp/sched_policy tests/sched_policy.c && /tmp/sched_policy
cc -o /tmp/mkdir_root   tests/mkdir_root.c   && /tmp/mkdir_root
```

`sched_policy` sets `SCHED_OTHER` on itself and expects success, then attempts `SCHED_FIFO` and expects `EPERM` (confirming the realtime path is still gated). `mkdir_root` calls `mkdir("/", 0755)` and expects `EEXIST`.

## Tested on

- FreeBSD 15.0-CURRENT, custom kernel config `NERVE`, ZFS root
- Intel i9-10980XE (36 threads), 32 GB RAM, ASUS ROG RAMPAGE VI APEX
- NVIDIA RTX 3090 Ti, driver 595.80
- Debian Trixie Linuxulator jail
- Unreal Engine 5.8, SM5 Vulkan, 18 shader compile workers

## Upstream status

Both patches are formatted for submission to FreeBSD's Phabricator instance at <https://reviews.freebsd.org>. Linuxulator changes are reviewed by the emulation team; the relevant mailing list is `freebsd-emulation@FreeBSD.org`.

<!-- Fill in once the reviews are open: -->
- 0001 — review: `Dxxxxx`
- 0002 — review: `Dxxxxx`

## Out of scope

The following are part of the same overall effort but live elsewhere, since they are userland or port-level rather than kernel patches:

- SDL3 initialisation fallback ladder (`LinuxPlatformApplicationMisc.cpp`) — an engine-side patch
- `libmapfilesfix.so`, `steamfix.so` and other `LD_PRELOAD` shims
- `linprocfs` field-conformance work for Linux 6.1 `/proc/self/status`
- jail configuration, devfs rulesets and `linsysfs` overlays

## Licence

These patches are derived from FreeBSD source and are offered under the same terms as the files they modify (BSD-2-Clause).

## Contributing

Reproducers for other Linuxulator divergences are welcome, particularly ones that block a real application and that userland cannot paper over. Please include the failing syscall, the errno or behaviour Linux produces, and what FreeBSD produces instead.

