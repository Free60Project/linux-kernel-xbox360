# Porting the Xenon patch set from Linux 6.18 to 7.0.x

Target: **linux-7.0.14** (the final 7.0.x point release).
Base:   `patch-6.18-xenon0.30.diff` @ repo commit `498bbe1`.

---

## Headline result

**The 6.18 patch applies to 7.0.14 with zero rejects, zero fuzz and zero
offsets.** All 66 files patch cleanly:

```
$ patch -p1 --dry-run --forward --fuzz=0 < patch-6.18-xenon0.30.diff
  hunks FAILED : 0
  fuzz/offset  : 0
  files patched: 66 / 66
```

No hunk required manual resolution. There is no source file in this port that
needs reimplementation against 7.0.

### Why it applied cleanly

35 of the patched files are pre-existing mainline files. Comparing each at
`v6.18` and `v7.0`, 19 are byte-identical and 16 changed:

```
UNCHANGED 6.18 -> 7.0  (the load-bearing arch hunks)
  arch/powerpc/include/asm/{cputable,mmu,cacheflush,udbg}.h
  arch/powerpc/kernel/{cpu_specs_book3s_64.h,misc_64.S,prom.c,udbg.c}
  arch/powerpc/platforms/{Kconfig,Makefile}
  arch/powerpc/Kconfig.debug
  drivers/gpu/drm/tiny/{Kconfig,Makefile}
  drivers/ata/Makefile   drivers/tty/serial/Makefile
  include/uapi/linux/serial_core.h
  sound/pci/{Kconfig,Makefile}

CHANGED 6.18 -> 7.0  (context still matched, so no rejects)
  arch/powerpc/boot/Makefile          arch/powerpc/platforms/Kconfig.cputype
  lib/Kconfig.debug                   drivers/Makefile
  drivers/{ata,char,hwmon,leds,rtc,tty/serial}/Kconfig|Makefile
  drivers/net/ethernet/sis/sis190.c   drivers/scsi/scsi_scan.c
```

The 16 changed files are almost all Kconfig/Makefile one-line insertions whose
surrounding context did not move.

---

## What DID need manual resolution

Not a single source hunk. **Both manual changes are in the defconfig**, and
neither is something `patch` can detect -- the patch applies fine and the
breakage only appears at `make xenon_defconfig` time.

### 1. `CONFIG_PREEMPT_VOLUNTARY` no longer exists on powerpc  (SILENT)

This is the one that needs real understanding, not a mechanical fix.

At 7.0 the preemption Kconfig gained two new dependencies:

```diff
  config PREEMPT_NONE
+ 	depends on ARCH_NO_PREEMPT

  config PREEMPT_VOLUNTARY
+ 	depends on !ARCH_HAS_PREEMPT_LAZY

  choice
- 	default PREEMPT_NONE
+ 	default PREEMPT_LAZY if ARCH_HAS_PREEMPT_LAZY
```

`arch/powerpc/Kconfig:150` selects `ARCH_HAS_PREEMPT_LAZY`, and powerpc does
**not** set `ARCH_NO_PREEMPT` (0 occurrences). So on powerpc at 7.0:

| model | selectable? | why |
|---|---|---|
| `PREEMPT_NONE`      | NO  | needs `ARCH_NO_PREEMPT`, powerpc lacks it |
| `PREEMPT_VOLUNTARY` | NO  | needs `!ARCH_HAS_PREEMPT_LAZY`, powerpc has it |
| `PREEMPT_LAZY`      | yes | the new default |
| `PREEMPT`           | yes | full preemption |

**There is no non-preemptible option left.** `PREEMPT_DYNAMIC` is not an escape
hatch either -- it `select PREEMPT_BUILD` unconditionally.

Verified empirically. Feeding the *unmodified* 6.18 defconfig to 7.0.14:

```
defconfig asked for : CONFIG_PREEMPT_VOLUNTARY=y
resolved .config has: CONFIG_PREEMPT_LAZY=y
                      CONFIG_PREEMPT_BUILD=y
                      CONFIG_PREEMPT_COUNT=y
                      CONFIG_PREEMPTION=y
```

Kconfig emits **no warning at all** for this. The line is simply dropped.

**Consequence:** the Xenon platform now runs a preemptible kernel for the first
time. `xenon-7.0.defconfig` states `CONFIG_PREEMPT_LAZY=y` explicitly so the
model is visible to anyone reading the file, rather than being inherited
silently.

**Risk assessment (audited, not assumed).** The usual `PREEMPTION=y` failure
class is per-CPU state accessed without `preempt_disable()`. Across all Xenon
platform code and drivers:

```
smp_processor_id()  : 0 occurrences
get_cpu()/per_cpu   : 0 occurrences
__this_cpu          : 0 occurrences
```

`drivers/xenon/smc-core.c` is the only driver with both locks and busy-waits,
and it is already correct: the FIFO spin at `:64` runs inside
`spin_lock_irqsave` with `cpu_relax()`, and the sleeping path
(`wait_event_interruptible` at `:84`) is called outside any lock. No
sleep-under-lock.

The code looks ready for a preemptible kernel. **This has not been confirmed on
hardware.** Use `xenon-7.0-debug.defconfig` for first boot.

### 2. `BOOTPARAM_HUNG_TASK_PANIC` changed type bool -> int  (WARNS)

```diff
  config BOOTPARAM_HUNG_TASK_PANIC
- 	bool "Panic (Reboot) On Hung Tasks"                        # 6.18
+ 	int "Number of hung tasks to trigger kernel panic"         # 7.0
+ 	default 0
```

`CONFIG_BOOTPARAM_HUNG_TASK_PANIC=y` is now invalid:

```
arch/powerpc/configs/xenon_defconfig:250:warning:
  symbol value 'y' invalid for BOOTPARAM_HUNG_TASK_PANIC
```

It falls back to `0`, which **disables** hung-task panic -- the opposite of the
6.18 intent. Changed to `=1` (panic after one hung task) to preserve behaviour.

### 3. `xenos.c` lost a transitive `drm_print.h` include  (BUILD BREAK)

The only compile error in the entire kernel, and it is in the display driver:

```
drivers/gpu/drm/tiny/xenos.c:100:17: error: implicit declaration of function
  'drm_info'; did you mean 'pr_info'? [-Werror=implicit-function-declaration]
  100 |   drm_info(&xenos->dev, "Using %dx%d (%04lx) fb\n", width, height,
```

`drm_info()` is declared in `include/drm/drm_print.h`. `xenos.c` never included
that header directly -- through 6.18 it arrived transitively via another DRM
header, and the 7.0 header cleanup broke that chain. `drm_print.h` itself is
unchanged; only the include graph moved.

Note this is NOT suppressed by `CONFIG_PPC_DISABLE_WERROR=y`. The defconfig sets
that, but `-Werror=implicit-function-declaration` is enabled globally by the
kernel build, so the port fails regardless of the powerpc Werror setting.

**Fix:** include it explicitly, which is correct practice anyway:

```diff
  #include <drm/drm_module.h>
+ #include <drm/drm_print.h>
  #include <drm/drm_probe_helper.h>
```

This is the single hunk in the whole port that required real investigation
rather than a mechanical edit. It is invisible to `patch` -- the patch applies
perfectly and the failure only appears at compile time. It is exactly the class
of breakage the CI build gate exists to catch.

### 4. `zImage.xenon` had two competing make recipes  (ORDER-DEPENDENT)

Building the boot target emitted:

```
arch/powerpc/boot/Makefile:403: warning: overriding recipe for target 'zImage.xenon'
arch/powerpc/boot/Makefile:388: warning: ignoring old recipe for target 'zImage.xenon'
```

The patch does two things that collide:

```make
image-$(CONFIG_PPC_XENON)  += zImage.xenon            # feeds the pattern rule
...
$(addprefix $(obj)/, $(sort $(filter zImage.%, $(image-y)))): ...   # generic wrapper rule
...
$(obj)/zImage.xenon: $(obj)/vmlinux.strip             # explicit flat-elf32 rule
```

Both target the same file. Make resolves it by taking whichever appears **last**
-- currently the explicit OBJCOPY rule, which is the correct one for Xenon. The
generic rule would produce a standard wrapped zImage that XeLL cannot boot.

So the build is correct today **by file ordering alone**. Any reordering of
`boot/Makefile` silently yields an unbootable image with no error.

Mainline avoids this for PS3 by naming its target `dtbImage.ps3`, which matches a
different pattern rule. Xenon cannot rename without breaking every existing guide.

**Fix:** exclude it from the generic rule so the explicit one is authoritative:

```diff
-$(addprefix $(obj)/, $(sort $(filter zImage.%, $(image-y)))): vmlinux $(wrapperbits) FORCE
+$(addprefix $(obj)/, $(sort $(filter-out zImage.xenon, $(filter zImage.%, $(image-y))))): vmlinux $(wrapperbits) FORCE
```

The rule itself was also converted to standard kernel idiom
(`quiet_cmd_`/`if_changed` + `FORCE`) so rebuilds track properly.
`targets += $(image-y)` already covers it.

### 5. A developer's personal path shipped in the patch

The upstream `zImage.xenon` rule contained:

```make
@test -e /mnt/e/Misc/tftpd64 && \
   cp -f $@ /mnt/e/Misc/tftpd64/vmlinux || true
```

Someone's WSL TFTP directory. Harmless (guarded by `test -e`) but it is
accidental debris. Removed.

---

## T7: `xenos.c` migrated off `drm_simple_display_pipe`

Requested during review as forward-looking cleanup. To be clear about the
justification: **`drm_simple_display_pipe` is NOT deprecated at 7.0.** It is
still exported (`EXPORT_SYMBOL(drm_simple_display_pipe_init)`), carries no
deprecation notice, and 9 of the 12 real `drm/tiny` drivers still use it. The
direction of travel is real -- the three newest tiny drivers (appletbdrm, bochs,
pixpaper) use the split API -- but nothing forced this change.

| Before | After |
|---|---|
| `struct drm_simple_display_pipe pipe` | `drm_plane primary_plane` + `drm_crtc crtc` + `drm_encoder encoder` |
| `xenos_pipe_funcs{.enable,.disable,.update}` | `drm_plane_helper_funcs{.atomic_check,.atomic_update}` + `drm_crtc_helper_funcs{.atomic_enable,.atomic_disable}` |
| `drm_simple_display_pipe_init()` | `drm_universal_plane_init()` + `drm_crtc_init_with_planes()` + `drm_encoder_init()` + `drm_connector_attach_encoder()` |

One real behavioural improvement fell out of it. The old `.enable` callback
allocated the tiled shadow framebuffer, and `.update` had to defend against
being called first:

```c
// I don't understand the DRM lifecycle well enough, but this gets
// called *before* enable in some cases.
if (IS_ERR_OR_NULL(xenos->real_framebuffer.vaddr))
        return;
```

With the split API the plane owns the shadow buffer, so allocation moved into
`.atomic_update` where the framebuffer is guaranteed to exist. The ordering race
the comment describes is gone rather than worked around.

**Status: compiles clean, zero warnings. NOT verified on hardware.** This is the
highest-risk change in the port and it is the one with the least justification.
If display misbehaves on real hardware, revert this commit first -- the rest of
the port is independent of it.

---

## Additional hardening included in this port

These are NOT 7.0 fixes. They are pre-existing defects in the 6.18 patch that
were fixed while the files were being touched anyway. Listed separately so a
reviewer can tell port work from cleanup.

### Dead NULL check in the interrupt controller  (real bug)

`arch/powerpc/platforms/xenon/interrupt.c` had the NULL test *after* the
dereference it was meant to guard, so it could never fire:

```c
host = irq_domain_add_linear(NULL, XENON_NR_IRQS, &xenon_irq_host_ops, NULL);
host->host_data = of_node_get(dn);   /* dereference */
BUG_ON(host == NULL);                /* dead -- the line above already crashed */
```

Reordered to check before use, with proper unwind.

### Unchecked `ioremap()` returns  (11 sites, 4 files)

Each of these is dereferenced immediately after mapping, several before any
console exists -- so a mapping failure was a silent dead box.

| File | Sites | Note |
|---|---|---|
| `arch/powerpc/platforms/xenon/interrupt.c` | 4 | `iic_base`, `bridge_base`, `biu`, `graphics` -- all written to immediately |
| `drivers/ata/sata_xenon.c` | 2 | `cmd_addr` feeds pointer arithmetic (`+ 0xa`), so NULL becomes `0xa` |
| `drivers/tty/serial/xenon_uart.c` | 2 | probe + console init |
| `sound/pci/snd-xenon.c` | 3 | incl. one feeding a `memset()` |

`drivers/char/xenon_probe.c` and `arch/powerpc/platforms/xenon/xe_udbg.c`
already checked their mappings correctly and were left alone.

### Unbounded spin on a possibly-NULL mapping

`sound/pci/snd-xenon.c` had:

```c
void *base = ioremap(0x200ea001000, 0x1000);
while (!(readl(base + 0x84) & 4));   /* NULL deref, empty body, never exits */
```

It also leaked the mapping on every call. Now NULL-checked, bounded by an
iteration cap with `cpu_relax()`, and `iounmap()`d on the timeout path.

### Still outstanding (see TODOS.md)

`drivers/xenon/smc-core.c:64` has the same unbounded-spin shape, but inside
`spin_lock_irqsave` with interrupts off -- a wedged SMC FIFO hangs the machine
with no console. Left alone here because bounding it needs a hardware-informed
timeout value. Tracked as T-E.

---

## Deliverables

| File | Purpose |
|---|---|
| `patch-7.0-xenon.diff` | The port. 6.18 patch + defconfig fixes + error hardening. |
| `xenon-7.0.defconfig` | Base config. Two forced fixes vs 6.18, nothing else. |
| `xenon-7.0-debug.defconfig` | Tier 1 bringup: `DEBUG_PREEMPT`, `DEBUG_ATOMIC_SLEEP`, `SCHED_DEBUG`, `PANIC_ON_OOPS`. Cheap; should boot on 512MB. |
| `xenon-7.0-lockdep.defconfig` | Tier 2: adds `PROVE_LOCKING`, `LOCK_STAT`. May OOM on 512MB unified RAM -- fall back to tier 1 if so. |
| `.github/workflows/build-7.0.yml` | CI gate: clean-apply check, then build both configs and assert `zImage.xenon` links. |

All three defconfigs resolve against 7.0.14 with **zero Kconfig warnings**.

---

## Toolchain

Linux 6.18 and 7.0 have identical minimum toolchain requirements
(GCC 8.1, binutils 2.30), so the port introduces no new toolchain constraint.

CI builds with Debian's stock `powerpc64-linux-gnu-gcc`. That verifies the
source **compiles and links**. It is NOT a hardware-certified build: Xenon's
AltiVec unit is incomplete (VMX128 substitutes for some instructions) and the
defconfig sets `CONFIG_ALTIVEC=y`. Build hardware images with the patched Free60
toolchain (`Free60Project/buildroot`, GCC ~12.4.0 + VMX128 patches).

Note `free60/toolchain:latest` (used by libxenon CI) builds a **bare-metal
newlib `xenon-` toolchain**, not a Linux-targeting one.

---

# Hardware bring-up findings (real Xbox 360, JTAG + XeLL)

Everything above was verified by building. This section is what only showed up
on real silicon. Boot chain: XeLL -> `uda0:/vmlinux` (USB, FAT32) -> kernel.

## Result

**Linux 7.0.14 boots on Xenon hardware**, all the way to:

```
VFS: Unable to mount root fs on unknown-block(8,1)
```

That is the expected endpoint with no root filesystem attached: major 8 =
SCSI disk layer, minor 1 = first partition, i.e. XeLL passed `root=/dev/sda1`
and no such block device exists. Every stage before it completed:

```
XeLL: 'uda0:/vmlinux' found, loading 18836520 / Launching ELF / Executing
kernel: real-mode init, early console
        device tree parsed, 3 CPU nodes / 6 threads
        hash MMU (64K pages, orders from DT)
        early_setup() returned
        Top of RAM 0x1e000000, 1 zone, 7680 pages
        percpu [0] 0 1 2 3 4 5
        SLUB CPUs=6 Nodes=1, RCU, NR_IRQS 384
        xenon IIC: init on cpu 0
        timebase + decrementer clocksources
        VT layer, tty0
```

## H1. `boot_cpuid_phys` is `0xfeedbeef` -- kernel BUG() before any output

The first real panic:

```
Oops: Exception in kernel mode, sig: 5 [#1]   TRAP: 0700
NIP  early_init_devtree+0x3d8/0x3fc
Kernel panic - not syncing: Fatal exception
```

Disassembling that address gives `bl _printk` followed by `twui r0,0` -- a
`printk()` then `BUG()`. In `prom.c` that is:

```c
printk("Failed to identify boot CPU !\n");
BUG();
```

`early_init_dt_scan_cpus()` matches CPU nodes against the `boot_cpuid_phys`
field of the FDT *header*:

```c
if (be32_to_cpu(intserv[i]) == fdt_boot_cpuid_phys(initial_boot_params))
```

Instrumenting the scan showed why it never matches:

```
XENON cpuscan: node="Xenon,PPE@0" nthreads=2 fdt_v17 boot_cpuid_phys=0xfeedbeef
XENON cpuscan:   intserv[0]=0x0
XENON cpuscan:   intserv[1]=0x1
XENON cpuscan: node="Xenon,PPE@1" ... intserv=0x2,0x3
XENON cpuscan: node="Xenon,PPE@2" ... intserv=0x4,0x5
```

The CPUs enumerate correctly as 0..5. `boot_cpuid_phys` is **0xfeedbeef** -- a
placeholder sentinel that is never filled in. The FDT is v17, so the header
field exists; it just contains garbage. XeLL patches `/chosen` bootargs,
initrd start/end and per-CPU `clock-frequency`, but never sets this field.

**Workaround applied:** fall back to logical CPU 0 (thread 0 of core 0, hwid 0)
with a loud message instead of `BUG()`.

**Proper fix, not yet done:** either have XeLL set `boot_cpuid_phys`, or derive
the boot CPU from the PIR register rather than trusting the FDT header. The
fallback is correct for a cold boot on core 0 but is an assumption, not a
measurement.

Note this is NOT a 7.0 regression -- `prom.c` is byte-identical between v6.18
and v7.0. It is a latent mismatch between the Xenon patch set and XeLL. How the
6.18 kernel ever got past it is an open question.

## H2. The early console was never enabled (why every failure was silent)

`udbg_init_xenon()` runs in real mode, sets up the console context, clears the
screen -- and leaves output disabled:

```c
console_clrscr(&console_ctx);
// udbg_putc = console_putch;      <- commented out upstream
```

`udbg_putc` is only set later by `udbg_init_xenon_virtual()`, called from
`xenon_probe()` (the machine `.probe` hook) inside `setup_arch()`. So **any**
failure before `probe_machine()` produced a blank/partially-cleared screen and
nothing else -- which is exactly what H1 looked like from the outside.

Enabling it in the real-mode path is what made H1 diagnosable at all.

**But upstream's comment was right, for a reason worth writing down.** The
framebuffer is at `0x1E000000`, immediately above the end of RAM (the device
tree declares memory as `0..0x1e000000`), so it is **outside the kernel linear
map**. The physical pointer works in real mode and becomes unusable the instant
the MMU is enabled at the end of `early_setup()`. Naively enabling early output
therefore trades a silent early failure for a silent fault at MMU-on.

Resolved by tracking which kind of pointer the context holds and skipping
output across the window:

```c
if (!console_ctx.fb_is_virtual && (mfmsr() & MSR_DR)) return;
```

Real-mode output works, goes quiet from MMU-on until `xenon_probe()` ioremaps
the framebuffer, then resumes. The Linux banner is lost in that gap.

## H3. Bootconsole handoff blanks the display

```
Console: colour dummy device 80x25
printk: legacy console [tty0] enabled
printk: legacy bootconsole [udbg0] disabled
```

Normal Linux behaviour, but on this platform it means the display goes dead:
`tty0` binds to the dummy console because DRM is a PCI driver and has not
probed yet. Shortly after, `xenos_enable()` allocates its scanout with
`dma_alloc_coherent()` and points the display at it **without clearing it**, so
the screen fills with uninitialised RAM (full-colour noise).

`keep_bootcon` alone does not fix this -- udbg writes to `0x1E000000` while DRM
has moved the scanout elsewhere.

For bring-up, `xenon-7.0-diag.defconfig` disables `DRM_XENOS` and sets
`CONFIG_CMDLINE="keep_bootcon ignore_loglevel"` with `CMDLINE_EXTEND=y`, giving
one uninterrupted log for the whole boot. That is a diagnostic config, not the
shipping one.

**Open:** `xenos_enable()` should clear the buffer it allocates. And whether
fbcon renders correctly through the tiled blit path is still unverified --
if it does not, commit `ffa7bc8` (the drm_simple_display_pipe migration) is the
first thing to revert, since it rewrote the plane update path.

## H4. Processor frequency is wrong by ~3.2x

```
WARNING: Estimating processor frequency (not found)
time_init: processor frequency = 1000.000000 MHz
```

The Xenon runs at 3.2GHz. XeLL sets `clock-frequency` on
`/cpus/Xenon,PPE@{0,1,2}` (confirmed in libxenon `elf.c`) but the kernel does
not find it where it looks, and falls back to an estimate of 1GHz.

Everything `udelay()`-based is therefore roughly a third of its intended
duration. Not fatal for boot, but SATA resets, SMC transactions and any other
timing-sensitive driver should not be trusted until this is fixed.

## Still open

| # | Issue | Impact |
|---|---|---|
| H1 | `boot_cpuid_phys=0xfeedbeef`, fallback is an assumption | boot CPU selection |
| H3 | fbcon over the tiled blit path unverified; DRM buffer not cleared | display / WM goal |
| H4 | CPU frequency 1GHz vs 3.2GHz | all `udelay()` timing |
| -- | no root filesystem (ppc64 **big-endian** userland needed) | SSH / WM goals |

---

# Framebuffer console (working)

`Console: switching to colour frame buffer device 160x45`

fbcon binds to the Xenos DRM driver and owns tty0. Login prompts run on
tty1/tty2 (Alt+F1/F2). Three bugs had to be fixed to get there, all of them
introduced or exposed by the T7 `drm_simple_display_pipe` migration.

## D1. dma_alloc_coherent() from atomic context

```
WARNING: kernel/dma/pool.c:291 at dma_alloc_from_pool
Workqueue: events drm_fb_helper_damage_work
  dma_alloc_from_pool <- dma_direct_alloc <- xenos_plane_atomic_update
```

T7 moved the scanout allocation out of `.enable` (sleepable) into
`.atomic_update`, which runs in atomic commit context. `dma_alloc_coherent()`
there falls back to the small atomic DMA pool, which cannot satisfy a
multi-megabyte request. It failed on every damage update, so the buffer was
never valid and nothing was ever blitted.

Symptom on screen: full-colour noise (the never-painted buffer being scanned
out) plus a warning storm.

Fix: allocate once in `xenos_load()` at probe time, in process context. The
mode is fixed (read from the Xenos registers) so it never needs to change.

## D2. Scanout buffer sized linear instead of tiled  (memory corruption)

The scanout is a 32x32 tile grid. `xenos_blit()` indexes it as

```
(y >> 5) * 32 * width + (x >> 5) * 1024 + <up to 1023>
```

At 1280x720 the maximum index is 942,079 words = 3,768,320 bytes. Allocating
the linear size `1280 * 720 * 4 = 3,686,400` is **81,920 bytes short**, so the
bottom rows write past the end of the DMA buffer into whatever the allocator
handed out next.

Symptom: black screen (the memset from D1 worked) with colour artefacts in the
**bottom-right corner only** -- the tiled index grows with y, so only the final
row band overflows -- followed by the machine dying as kernel memory was
corrupted. It died before dropbear started, so there was no way in over the
network; the USB stick had to be moved to fix it.

Fix: `ALIGN(fb_width, 32) * ALIGN(fb_height, 32) * 4`, i.e. 1280x736. This is
the same rounding the udbg console already did (`((720 + 31) >> 5) << 5`).

## D3. Buffer freed on every CRTC disable

`atomic_disable` called `dma_free_coherent()` and NULLed the pointer. That was
a use-after-free hazard against in-flight plane updates, and it is *why* the
buffer had to be reallocated from atomic context in the first place. The
scanout is now owned for the lifetime of the device.

## D4. getty on tty0 never holds the terminal

fbcon worked but no login prompt appeared. `tty0` is the "current VT" alias,
not a real terminal -- busybox getty exits immediately when given it, and init
respawns it forever. Use concrete VTs:

```
tty1::respawn:/bin/getty 38400 tty1
tty2::respawn:/bin/getty 38400 tty2
```

## Also fixed

- `/etc/fstab` and the kernel-push tooling hardcoded `/dev/sdb*`. **USB
  enumeration order is not stable across boots on this machine** -- the stick
  appeared as `sdb` on one boot and `sda` on the next, with the internal HDD
  swapping the other way. Everything now uses `LABEL=`. The initramfs was
  already label-based, which is the only reason boots kept working.

## Debugging note

Every one of D1-D4 was found in about twenty minutes once SSH existed, by
reading `dmesg` directly. The preceding hours were spent photographing a TV and
theorising. On a platform with no serial console, **getting a network shell up
before touching the display driver is worth more than any amount of careful
reasoning about pixels.**

---

# Build verification

Built and booted with the **Free60 reference toolchain**, not just a distro
cross-compiler:

```
Linux version 7.0.14-xenon (xenon-gcc (GCC) 9.2.0, GNU ld (GNU Binutils) 2.32)
zImage.xenon  18,849,384 bytes  ELF 32-bit MSB PowerPC, statically linked
0 errors, 0 warnings
```

On hardware this reaches VFS with:

```
CPU: 1 UID: 0 PID: 1 Comm: swapper/0 Not tainted 7.0.14-xenon #1 PREEMPTLAZY
Hardware name: Xenon Game Console Xenon
```

`Not tainted` -- no WARN fires anywhere during boot.

## Gotcha: root device timing when booting from USB

The defconfigs deliberately do not set `CONFIG_CMDLINE`, matching
`xenon-6.18.defconfig` and the rest of this repo. If you boot from a USB stick,
be aware the kernel tries to mount root almost immediately:

```
[ 3.7] VFS: Cannot open root device "" or unknown-block(8,1): error -6
[ 3.8] Kernel panic - not syncing: VFS: Unable to mount root fs
       List of all bdev filesystems: ext3 ext2 squashfs vfat
       0b00    177600 sr0
       0800 117220824 sda        <- only the internal disk; the USB is absent
```

USB mass storage on this hardware does not enumerate until roughly **33
seconds** in, so at 3.8s the stick genuinely does not exist yet. Either:

* append `rootwait` to the kernel command line (XeLL builds it from its device
  tree `/chosen` node), or
* use an initramfs that waits for the device and then `switch_root`s.

This is not specific to 7.0 -- it applies equally to the 6.x defconfigs -- but
it is easy to mistake for a broken port.
