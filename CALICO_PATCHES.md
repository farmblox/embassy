# Calico Patches to Embassy

This file documents the patches Farmblox/Calico applies on top of the upstream
`embassy-stm32-v0.6.0` tag on the `calico-v0.6.0-base` branch of our fork.

It is a record of what we carry, why, and whether each patch could be
upstreamed. Use it whenever we rebase onto a future upstream release.

## Snapshot

- Fork: `https://github.com/farmblox/embassy.git`
- Branch: `calico-v0.6.0-base`
- Upstream base: `embassy-stm32-v0.6.0` (commit `84444a19e`, 2026-03-20)
- Divergence: 8 commits ahead of `embassy-stm32-v0.6.0`
- All patches live in `embassy-stm32`. No other crate is modified.

## Chips Calico ships against

`stm32l051c8`, `stm32l053r8`, `stm32l431cc`, `stm32l476rg`. **No STM32WL chip
ships.** Patches that only affect WL are not carried.

## Carried patches

Listed in topological order (oldest → newest). Each entry names the commit, the
purpose, the firmware call sites that depend on it, and a verdict on whether
the patch is upstreamable as-is.

### 1. `b78f3ff9b` — rcc: add `msi_pllmode` feature for PLL-locked MSI frequency reporting

When the LSE is 32.768 kHz and MSI is locked to it via the PLL
(`set_msipllen(true)`), the actual MSI frequencies are integer multiples of
`32_768`, not the nominal `MSIRange` values. With `msi_pllmode` enabled,
`msirange_to_hertz()` returns the PLL-locked values (e.g. `RANGE16M` →
`15_991_808 Hz`), which keeps baud rates and timer math correct.

Used by `pnp-firmware`'s `clock_tree.rs:47` (`MSIRange::RANGE16M`) plus
`clock_tree.rs:101` (LSE 32.768 kHz oscillator). Embassy's `init` calls
`set_msipllen(true)` for that LSE setup. Without this patch, `frequency()`
reports the nominal 16 MHz when the MCU actually runs at ~15.99 MHz.

**Upstream verdict:** mergeable as a config option.

### 2. `20d249170` — usart: add non-breaking `read_until_idle_with_rto`

Adds a new `UartRx::read_until_idle_with_rto(&mut self, buf, rto: u32)` method
on `usart_v3` / `usart_v4` that programs the `RTOR` register so the IDLE
interrupt only fires after `rto` bit-times of silence. Coexists with the
existing `read_until_idle(buf)` to avoid a breaking API change.

Used by:
- `interfaces/bloxbus_v0.rs:207`, `interfaces/bloxbus_v1.rs:150`,
  `interfaces/modbus.rs:480` (rto = 200)
- The remaining sensor and SDI-12 callers still use the old
  `read_until_idle(buf, rto)` two-argument shape from the prior fork; those
  files are part of the in-progress migration and will be updated to either
  `read_until_idle_with_rto` or to the upstream single-arg form when their
  builds are revived.

**Upstream verdict:** mergeable. The two-arg variant is a clean addition.

### 3. `99d9206e0` — usart, rcc: add `enable`/`disable` on `Uart`, `UartTx`, `UartRx`

Toggles the USART RE/TE bits and clock-gates the peripheral via
`rcc::enable_with_cs::<T>(cs)` / `rcc::disable::<T>()` (both already
upstream as of v0.6.0). Configured baud, parity etc. survive a sleep.

Used by:
- `sensors/dyp_a22_uart_ctrl.rs:38,44` (`uart_rx.disable()`/`enable()`)
- `sensors/motion_triggered_distance.rs:53,56` (`uart.enable()`/`disable()`)
- `main.rs:242,467` (`bloxbus_serial.disable()`/`enable()`)

**Upstream verdict:** small, self-contained API addition; mergeable.

### 4. `5046ae619` — adc/v3: extend `Drop` with STOP2 power-down sequence

Upstream's `Drop` calls `AdcRegs::stop` (clears `cont`/`dmaen`) and
`disable_without_stop()`. That is insufficient for the MCU to enter STOP2
because `aden`, `advregen`, and `deeppwd` remain set: the ADC keeps drawing
current and the chip refuses STOP2 or wakes immediately.

For `adc_v3` (stm32l4 family), the new `Drop` runs `power_down()` (set_addis,
spin until `aden=0`), clears `advregen`, sets `deeppwd=true`, and then calls
`disable_without_stop()`.

Used by:
- `sensors/alarm_kit.rs:303`, `sensors/builtin_adc.rs:30`,
  `sensors/coprlock_dual_trigger.rs:440`, `boards/vanguard.rs:78`

**Upstream verdict:** STOP2 correctness fix any low-power user would want.
Mergeable.

### 5. `c9786f74f` — adc/v3: route STOP2 power-down through trait method

Follow-up to patch 4. Drop's bound stays generic over `T: Instance` (so
non-`adc_v3` chips still compile); the v3-specific work goes through a new
`AdcRegs::full_power_down` trait method with an empty default and a
`#[cfg(adc_v3)]` override on `crate::pac::adc::Adc`. Other ADC families that
share `v3.rs` keep upstream's lighter Drop unchanged.

**Upstream verdict:** part of the same upstreamable change as patch 4.

### 6. `28aa86e72` — low_power: add `low-power-sleep-gate` feature for `SHOULD_SLEEP` atomic

Behind the opt-in `low-power-sleep-gate` Cargo feature, declares a public
`SHOULD_SLEEP: AtomicBool` (default `true`). When `false`, `configure_pwr`
short-circuits before calling `get_stop_mode`, so the chip executes plain
`WFI` instead of entering STOP. Useful for keeping the chip awake during
RTT/SWD debugging or for application code paths that must stay awake.

Used by `pnp-firmware/src/main.rs:264, 350, 444, 554` (gating around the
sensor-specific work that benefits from RTT visibility).

**Upstream verdict:** small, opt-in, no behavioral change when feature is off;
likely mergeable.

### 7. `a448c789a` — low_power: don't enter STOP when `pause_time` failed

Upstream's `configure_pwr` warned when `pause_time` returned `Err` (meaning
the next embassy-time alarm is within `min_stop_pause`) but then *fell
through* into `enter_stop` and `set_sleepdeep`. The chip would enter STOP
with TIM still running, the TIM compare event would never fire from STOP
(APB gated), and the chip wedged until either an EXTI event or the watchdog
~15 s later.

This patch adds an early `return` after the failed `pause_time`, so the
caller's `WFI` does plain idle for the imminent alarm window. Net behavior:
short waits below `min_stop_pause` no longer attempt STOP entry, matching
the documented intent of `min_stop_pause`. The original upstream `warn!`
log is preserved; if it becomes too noisy in a given workload, the
intended fix is at the application layer (raise the workload's longest
short-await above `min_stop_pause`, or accept the log as informational).

**Upstream verdict:** non-controversial bug fix; should land upstream.

### 8. `0ccaf8d33` — stm32/rtc: fix `stop_wakeup_alarm` clearing unrelated EXTI pending bits

Cherry-pick of upstream commit `0ccaf8d33` (PR #5919, merged to `main` after
the v0.6.0 tag). `stop_wakeup_alarm` was using `.modify()` on `EXTI.PR(0)`,
which is a `rc_w1` (write-1-to-clear) register. The read-modify-write cycle
reads every currently-pending EXTI bit and writes them all back, clearing
**every** pending EXTI line, not just RTC's `EXTI_WAKEUP_LINE` (line 20 on
stm32l4). Fixed by switching to `.write()`, which only writes the RTC bit
and leaves the other lines' pending state untouched.

The bug bites every STM32L4 build that uses `low_power`: when the wake pin
(EXTI3 on coprlock/alarm-kit) fires while the chip is in STOP, WFI returns,
`on_wakeup_irq_or_event` calls `resume_time` → `stop_wakeup_alarm`, the
buggy `.modify()` clobbers `EXTI.PR[3]`, the EXTI3 ISR then sees `PR(0) = 0`
and never wakes its waker. The task awaiting `wait_for_high()` stays Pending,
`SHOULD_SLEEP` stays `true`, and the chip re-enters STOP. With the wake pin
now low again, no further rising edge ever wakes it. Previously diagnosed
on `calico-main` and cherry-picked there; the fix was lost during the
rebase to `calico-v0.6.0-base` and is restored here.

The cfg gate `#[cfg(any(exti_v1, stm32h7, stm32wb))]` matches stm32l4
(`exti_v1` is true; verified via `cargo:rustc-cfg=exti_v1` in the build
script output).

Used by every `pnp-firmware` target that enables `low_power` — the bug
manifests as the alarm-kit/coprlock Stop wedge documented in
`pnp-firmware/CRASH_NOTES.md`.

**Upstream verdict:** already upstream; will arrive naturally with the next
embassy-stm32 release tag.

## Patches deliberately not carried

These were on the previous `calico-main` fork branch and were dropped or
subsumed during the rebase to `calico-v0.6.0-base`:

- **RTC overflow fix** (`2af156e2d`): subsumed by upstream's RTC rewrite — the
  `RtcInstant::Sub` impl no longer exists; new code does u64 math throughout.
- **RCC enable without reset** (`9e9d99c22`): upstreamed as
  `RccInfo::enable_with_cs` / `enable_with_cs<T: RccPeripheral>(cs)` in
  `rcc/mod.rs`. Patch 3 above uses this upstream helper directly.
- **USART eh-02 Write shim** (`c3d98a71d`): no caller in pnp-firmware.
- **RNG bracket fix** (`694c66378`): the surrounding code that the bracket
  was misplaced inside has been refactored away; pnp-firmware does not use
  `embassy_stm32::rng`.
- **WL low-power initial support, WL stop2-only flag, WL revert, WL lptim**
  (`47ec62e9b`, `4e56f839b`, `e679e216e`, `d60f132ed`): we ship no STM32WL
  chip in any current target.
- **Always restart time driver on EXTI wake** (`1469f2195`): subsumed —
  embassy's EXTI master ISR already calls `on_wakeup_irq` from EXTI handlers,
  which calls `resume_time` (verified in `embassy-stm32/src/exti/...` for
  v0.6.0).
- **Atomic-disable-sleep-for-debugging** (`d17bfccb9`): re-implemented
  cleanly as patch 6 (`SHOULD_SLEEP`) on top of the v0.6.0 `low_power.rs`.
- **low-power-rcc-reinit feature** (planned but not carried): the previous
  audit recommended porting an opt-in `rcc::reinit_saved` on STOP wake to
  recover stm32l4 from "MSI-not-PLL-locked-after-wake". The current
  observation is that with patch 1 (`msi_pllmode`) plus the upstream RCC
  flow, MSI re-locks to LSE on wake without an explicit reinit on the
  shipping targets. Revisit if a STOP-wake clock anomaly is observed on
  stm32l4.

## Side notes

- **monitor-firmware uses a different fork branch** (`msipll`). Whatever
  rebase path we choose for the next embassy bump on pnp-firmware will need
  to be applied independently to monitor-firmware.
- **Three of seven patches are upstreamable as-is** (1, 2, 7). Two more (3,
  4+5) are upstreamable with light cleanup. Patch 6 is opt-in and could be
  upstreamed but mostly serves debugging. Open upstream PRs after the next
  rebase to shrink the long-term carry.
