# DSPi -> Professional DLMS: Phase-1 Repository Audit

*Status: READ-ONLY audit. No source files were modified.*
*Audit date: 2026-10-16*
*Repository: `WeebLabs/DSPi` fork, checked out at `ec26bf0` ("Merge pull request #73 from WeebLabs/release/v1.1.5").*
*Audit branch: `cline/ytpegn8m` (this document only).*

---

## 0. Scope, method, and honesty constraints

This document satisfies Stage 3 of the master prompt ("REQUIRED INITIAL REPOSITORY AUDIT").
Per the non-negotiable rules it:

- Was produced by reading the actual source under `firmware/DSPi/`, `Documentation/`,
  `tools/`, and the build files. Every architectural claim cites a real file/symbol.
- Uses this status vocabulary for every capability:
  - **IMPLEMENTED** - present and self-consistent in source.
  - **IMPLEMENTED (differs)** - present but not shaped the way the DLMS spec assumes.
  - **PARTIAL** - some of the required behaviour exists, rest does not.
  - **MISSING** - no implementation found in source.
  - **UNVERIFIED** - code path exists but was not executed; no build/hardware test was possible here.
  - **HW-DEPENDENT** - cannot be decided without the unspecified DAC/amplifier/PCB.
- Does **not** claim production-readiness. The project compiles (per its own docs/CI) but
  nothing here was built or run.

### 0.1 Verification limitations (explicit)

- **No toolchain in this environment.** `arm-none-eabi-gcc`, `cmake` and `ninja` are not
  installed (`which` returned only `/usr/bin/python3`). The firmware was therefore
  **not compiled** and **no test was executed** during this audit. All runtime claims are
  read-from-source, not measured. Rules 14/15: reported as unverified.
- **`firmware/pico-sdk` submodule is not checked out** in this checkout
  (`firmware/pico-sdk/` is empty; `.gitmodules` declares it). SDK version is therefore
  **UNVERIFIED** - the CI workflow (`.github/workflows/cmake.yml`) is the only build
  evidence, and it was not run here.
- **No DAC/PCB/hardware attached.** All output-path electrical behaviour is HW-DEPENDENT.
- **No numerical filter checks were run**; filter/Q behaviour below is code-reading only.

---

## 1. Repository, branch, and build-system facts

| Item | Evidence | Finding |
|------|----------|---------|
| Identity | `README.md` | DSPi = RP2040/RP2350 USB Audio Class 1 DSP; this fork targets RP2350. |
| Head commit | `git log` | `ec26bf0`, tag lineage `release/v1.1.5`. Working tree clean at audit start. |
| Firmware source | `firmware/DSPi/` (~37.8k lines across `.c/.h`) | Single target `DSPi`, C11/C++17. |
| Build system | `firmware/CMakeLists.txt`, `firmware/DSPi/CMakeLists.txt` | CMake + Pico SDK + local `pico-extras`. Two local audio libs: `pico_audio_spdif_multi`, `pico_audio_i2s_multi`. |
| Linked libs | `firmware/DSPi/CMakeLists.txt` | `pico_multicore`, `hardware_flash`, `hardware_i2c`, `pico_i2c_slave`, `tinyusb_device`, the two audio libs, `pico_spdif_rx`. |
| SDK version | `.gitmodules`, empty `firmware/pico-sdk` | **UNVERIFIED** (submodule absent). |
| Linker files | `memmap_dspi_rp2040_xip.ld`, `memmap_dspi_rp2350_xip.ld` | XIP builds with hot audio/USB code forced into RAM via `.time_critical` and `__not_in_flash_func`. |
| CI | `.github/workflows/cmake.yml` | Build automation exists; not executed here. |
| Host tooling in repo | `tools/dspi_test/`, `tools/dspi_poke/`, `tools/filter_tester/`, `tools/adat_rx_test/` | Python/CLI test + filter-design helpers. **No GUI/control application.** |
| Host control app (Windows) | `find` for `*.sln/*.csproj/package.json/*.html` -> none | **MISSING from this repository** (see Sec. 8). |

**Documentation set** (`Documentation/`): a large feature-spec corpus
(`Documentation/Features/*.md`), `commands.md` (full protocol reference), `codebase_map.md`,
and the living `current_architecture.md`. Several specs are explicitly marked
**"design - not yet implemented"** (e.g. `webusb_spec.md`). README claims are treated as
suspect until matched to source (Rule 2).

---

## 2. Verified audio signal-flow architecture map

This corrects the conceptual map in the prompt to the **real** implementation.

```
INPUT SOURCES
  USB UAC1 (16/24-bit PCM, 44.1/48/96 kHz)  --usb_audio.c: process_audio_packet()
  S/PDIF RX                                 --spdif_input.c
  I2S RX (master or slave)                  --i2s_input.c   (PIO)
  ADAT RX (RP2350)                          --adat_input.c  (PIO)
        |
        v
INPUT BUFFER / FORMAT DECODE
  stereo: buf_l / buf_r ; multichannel: buf_in_ext[6]  (audio_pipeline.h)
  Per-channel preamp applied at decode (PASS 1)  --usb_audio.c (preamp_l/r, Q28 path)
        |
        v
process_input_block(sample_count)   <-- audio_pipeline.c:265  [runs on CORE 0 main loop]
  PASS 2   per-input PEQ + per-input peak/clip meter
  PASS 2.5 Volume Leveller (input-linked RMS upward compressor + -3 dBFS gain-cap "safety limiter")
  PASS 3   stereo upmixer (optional; C/Ls/Rs into idle input rows)
  PASS 4   MATRIX MIXER  crosspoints[in][out] {enabled, phase_invert, gain_linear}
  PASS 4.5 per-output-PAIR crossfeed (pre-EQ)
  PASS 5-7 (per output, split across cores):
      psychoacoustic bass -> crossover cascade (XoverFilter sections[4])
      -> per-output PEQ -> per-output gain(ramped) -> loudness -> master volume
      -> mute -> per-output DELAY (circular delay_lines[][MAX_DELAY_SAMPLES])
      -> output encode
        |
        +--> CORE 1 (EQ_WORKER) handles outputs 2..N-1 via shared Core1EqWork struct
        |
        v
OUTPUT BUFFER
  S/PDIF/I2S: pico-audio producer pools producer_pool_1..4 (192 samples/buffer)
  PDM sub:    ring buffer -> pdm_generator (delta-sigma)
        |
        v
AUDIO OUTPUT DRIVER (PIO + DMA)
  Slot 0..3 : one PIO0 state machine each, S/PDIF (BMC) OR I2S (left-justified 24-in-32)
  PDM sub   : PIO1 SM0 delta-sigma
  MCK       : hardware CLK_GPOUTn (NOT PIO)  --config.h / master_clock_spec.md
        |
        v
EXTERNAL I2S DAC (model UNSPECIFIED)  -> optional GPIO hardware MUTE line (dac_hw_mute.c)
        |
        v
ANALOG OUTPUT STAGE (UNSPECIFIED / not in this repository)
```

Key correction vs. the prompt's conceptual map: **there is no separate "master"/input
processing block distinct from the outputs** - as of wire format V16 the unified channel
model (`config.h` "Channel Model" comment) makes inputs `0..N-1` and outputs `N..` in one
`NUM_CHANNELS` index space. Inputs carry PEQ + metering only; outputs carry PEQ + crossover
+ gain/delay/mute. `NUM_CHANNELS` = 17 on RP2350 (8 in + 9 out) and 7 on RP2040 (2 in + 5 out).

---

## 3. Verified control-path map

```
HOST (Windows PC)                       <-- NO first-party app exists in this repo (Sec. 8)
   |  USB EP0 control transfers, vendor interface #2, VID:PID 0x2E8B:0xFEAA
   |  bmRequestType 0xC1 (IN/read) or 0x41 (OUT/write)
   v
USB IRQ  (TinyUSB UAC1 class driver)  --usb_audio.c
   |  ISR ONLY moves bytes (audio into RAM ring; control SETUP staged)
   v
MAIN LOOP dispatch  --main.c:1948 while(1) { tud_task(); ... }
   v
vendor_control_xfer_cb()  --vendor_commands.c   (170 REQ_* opcodes in config.h; ~188 case handlers)
   v
VALIDATION + APPLY
   |  "SET" handlers validate ranges, write the DSP global, then emit notify_param_write(offset,...)
   |  Pipeline-affecting ops (preset load/save/delete, factory reset, output-type switch, pin move,
   |  input-source switch, bulk apply, flash write) are DEFERRED to the main loop by *_pending flags
   v
DSP PARAMETER UPDATE
   |  Direct global writes (matrix_mixer, channel_delay_samples, filters[], etc.)
   |  Coefficients guarded by a Core-1 "wait for block boundary" spin (main.c ~2533)
   v
PERSISTENCE  --flash_storage.c  (10 preset slots, SLOT_DATA_VERSION 35, CRC32, wear-aware)
             --bulk_params.c   (WIRE_FORMAT_VERSION 28 whole-state wire image)
```

Additional control transports share the same vendor surface: **UART (`uart_control.c`)** and
**I2C target (`i2c_control.c`)**, polled from the main loop. **WebUSB** is designed but
**not implemented** (`webusb_spec.md` header: "design - not yet implemented").

### 3.1 Where control can touch the real-time audio path

- Control is dispatched from the **main loop**, and the DSP also runs in the **main loop**
  (`usb_audio_drain_ring()` is the first loop step). They are **co-located in one thread**,
  which eliminates mutex contention on the hot path but means a slow control handler can
  **delay the next audio block**. Mitigations: `*_pending` deferral plus the RAM ring buffer
  (the USB ISR fills the ring independently, so the USB stream survives flash stalls).
- **Flash writes** stall XIP for tens of ms; bracketed by mute + RX suspend
  (`prepare_flash_write_operation()` / `complete_flash_write_operation_*()`). During the
  deferred window a control transfer can STALL/time out - the "busy window" of
  `commands.md` Sec. 1.3 (~45 ms per sector worst case).
- **Coefficient updates** to a channel Core 1 is processing are guarded by spinning until
  `core1_eq_work.work_done` (main.c ~2533-2541): a bounded busy-wait on Core 0 with `__dmb()`.
  This is the project's atomic-update mechanism.
- `main.c:480` and `main.c:988` contain `while(...) tight_loop_contents();` waits (DAC-mute
  hold and a core-0/core-1 sync). These are main-loop busy-waits, not ISR waits; they do
  block other main-loop work for their bounded duration.


---

## 4. Real-time execution model (RP2350)

| Aspect | Finding | Evidence |
|--------|---------|----------|
| System clock | 307.2 MHz fixed, core voltage 1.15 V. | `README.md`, `firmware/CMakeLists.txt` (`PICO_FLASH_SPI_CLKDIV=6` override). |
| Audio processing context | **Main loop**, not an ISR. `usb_audio_drain_ring()` -> `process_audio_packet()` -> `process_input_block()`. | `usb_audio.c:748`, `usb_audio.c:737`, `main.c:1948` loop. |
| USB audio RX | ISR pushes packets into a RAM ring (`audio_ring`); main loop drains. Feedback via SOF-measured 10.14 controller. | `usb_audio.c:122,1491`; `usb_audio_ring.h`; `usb_feedback_controller.c`. |
| Core allocation | Core 0 = control + pipeline driver + outputs 0-1 (pairs 0) + PDM pacing. Core 1 = **one of**: PDM generator OR EQ worker (outputs 2..N-1). Mode is exclusive. | `main.c:1816-1828 derive_core1_mode()`, `multicore_launch_core1(pdm_core1_entry)`. |
| Inter-core sync | Shared `Core1EqWork` struct + `work_ready`/`work_done` flags + `__sev()`/`__dmb()`. Single-view snapshots (loudness/crossfeed/psybass) taken once per block. | `config.h` Core1EqWork; `audio_pipeline.c:511-531`. |
| Block size | `AUDIO_BUFFER_SAMPLES = 192`, `AUDIO_BUFFER_COUNT = 8` producer buffers; consumer depth 16. | `config.h:78-80`. |
| Sample rates | 44.1 / 48 / 96 kHz. | `README.md`; rate handling in `usb_audio.c`. |
| Sample representation | RP2350 = **single-precision float** internal; RP2040 = **Q28 fixed point** (1.0 = `1<<28`). Final encode clamps to signed 24-bit. | `config.h` Filter struct; `output_s24.h`. |
| IRQ priorities | `DMA_IRQ_0` for SPDIF/I2S TX set to `PICO_HIGHEST_IRQ_PRIORITY`. | `usb_audio.c:1781-1782`. |
| DMA/PIO | S/PDIF TX + I2S TX on `DMA_IRQ_0`/`DMA_IRQ_1`; S/PDIF RX on `DMA_IRQ_1`; PDM on PIO1. | `firmware/DSPi/CMakeLists.txt` IRQ defines; `config.h:64-76`. |
| CPU metering | Budget-based EMA: `inst_q8 = busy_us*25600/budget_us`, cap 100%. | `audio_pipeline.c:1240-1250`. |
| Memory placement | Hot functions `__not_in_flash_func` / `.time_critical`; delay lines & buffers in BSS; a `scripts/check_ram_placement.py` + `ram_baseline.json` guard RAM residency. | `config.h` `DSP_TIME_CRITICAL`; `scripts/`. |

**Assessment.** The "no allocation / no flash / no blocking locks on the audio path"
discipline is genuinely present: audio buffers are static, the ISR is decoupled via a RAM
ring, and flash work is bracketed. The two busy-waits (`main.c:480`, `main.c:988`) are
bounded but are the only places where a lifecycle event can stall the main loop. Whether the
budget math holds at 96 kHz with all 9 outputs + all DSP features enabled is **UNVERIFIED**
(requires on-target measurement; Rule 16 forbids inventing the number).


---

## 5. Subsystem audit

Format per subsystem: Implementation / Source / Strengths / Missing / Correctness risks /
HW dependencies / Recommended change / Tests needed.

### 5.1 Boot & orchestration
- **Current implementation:** `core0_init()` brings up clocks, flash divider, USB, outputs,
  control surfaces; then `main()` runs the loop. Deferred pipeline ops serviced in-loop.
- **Source:** `main.c` (`core0_init`, `main` 1934+), `flash_clkdiv.c`.
- **Strengths:** explicit deferred-work model; synchronized output restarts; watchdog (8 s).
- **Missing:** no formal RT budget/enforcement; no fault capture on loop overrun.
- **Correctness risks:** a slow deferred op (flash) can starve the loop; busy-wait syncs.
- **HW dependencies:** none.
- **Recommended change:** add a loop-time overrun counter + log (diagnostic only) before any
  architectural change. **Tests:** host-side loop-timing instrumentation; on-target soak.

### 5.2 Audio inputs (USB / S/PDIF / I2S / ADAT)
- **Current implementation:** USB UAC1 custom class driver; S/PDIF RX (channel-status + LG
  Sound Sync decode); I2S RX master/slave (PIO); ADAT RX (PIO, RP2350). Runtime source switch.
- **Source:** `usb_audio.c`, `spdif_input.c`, `i2s_input.c`+`i2s_input.pio`, `adat_input.c`,
  `lg_sound_sync.c`, `input_servo.c`.
- **Strengths:** multi-source with runtime switch; 2/4/6/8-channel USB alt modes.
- **Missing:** sample-rate conversion (not expected); explicit input-availability fault state.
- **Correctness risks:** input-switch transients (mitigated by mute brackets).
- **HW dependencies:** S/PDIF/I2S/ADAT electricals.
- **Recommended change:** none for Phase 1 - preserve. **Tests:** `tools/dspi_test/tests/inputs.py`.

### 5.3 DSP pipeline core (`audio_pipeline.c`)
- **Current implementation:** pass-ordered pipeline; float (RP2350) and Q28 (RP2040) variants;
  per-output work split across cores via `Core1EqWork`.
- **Strengths:** static buffers, snapshot discipline, RAM-resident hot path.
- **Missing:** no per-output limiter stage; no oversampled peak detection.
- **Correctness risks:** mid-block coefficient change avoided via guard; ramp for volume/gain.
- **Recommended change:** add per-output protection as a *new* stage (see 6.6) rather than
  altering existing stages. **Tests:** deterministic vector tests via `tools/dspi_test`.

### 5.4 Matrix mixer & channel model
- **Current implementation:** `MatrixMixer` of `MatrixCrosspoint[in][out]` with enabled /
  phase_invert / gain_db / gain_linear; per-output `OutputChannel` with enabled/mute/gain/delay.
- **Source:** `config.h` (`MatrixCrosspoint`, `OutputChannel`, channel-model comment);
  handlers `REQ_SET_MATRIX_ROUTE 0x70` etc. in `vendor_commands.c`.
- **Strengths:** single authoritative channel model after V16; per-crosspoint gain + polarity.
- **Missing:** no explicit "input unavailable" safe-state beyond zeroing; no anti-feedback
  routing guard; channel linking only implicit via identical params.
- **Recommended change:** add routing validation + a capabilities query (see 6.1).
  **Tests:** `tools/dspi_test/tests/crosscut.py`, `outputs.py`.

### 5.5 PEQ
- **Current implementation:** up to 10 bands/channel; 14 `FilterType` PEQ values (peaking,
  low/high shelf, LP, HP, notch, 2nd-order and 1st-order all-pass, 1st-order shelves,
  Linkwitz Transform, 1st-order LP/HP); hybrid SVF/TDF2 on RP2350, Q28 TDF2 on RP2040.
- **Source:** `config.h` (`enum FilterType`, `Filter`), `dsp_pipeline.c`, `dsp_svf.h`,
  `dsp_biquad.h`, `Documentation/Features/peq_filters.md`.
- **Strengths:** rate-aware coefficients; per-band bypass; stable SVF at low freq on RP2350.
- **Missing:** none material; type space 14..31 reserved.
- **Recommended change:** none - preserve. **Tests:** `tests/eq.py`, `tools/filter_tester/`.


### 5.6 Crossovers
- **Current implementation:** per-output cascades of up to 4 sections. Families/orders:
  Linkwitz-Riley 2/4/6/8, Butterworth 1-8, Bessel 2/4/6/8. Wire bands `20..23`.
- **Source:** `crossover.c`/`.h` (`xover_design_filter`, `xover_process_channel_block`,
  `design_butterworth/linkwitz_riley/bessel`), `Features/crossover_filters_spec.md`.
- **Strengths:** analytic pole tables; LR = squared Butterworth; in-place between matrix and
  PEQ; whole-band bypass.
- **Missing:** odd-order LR (by design); range/order validation appears to be upstream in the
  vendor handler - **UNVERIFIED** here.
- **Correctness risks:** state re-init on type change; section ordering (documented stable).
- **Recommended change:** confirm/centralize frequency+order validation (see 6.2).
  **Tests:** `tools/filter_tester/test_crossover.py`, `compare_filter.py`.

### 5.7 Gain / polarity / master volume
- **Current implementation:** per-crosspoint gain (-inf..+12 dB) + polarity; per-output gain;
  input preamp; master volume (attenuation-only ceiling, -128..0 dB, true-mute sentinel).
- **Source:** `config.h` (`REQ_SET_CHANNEL_GAIN 0x54`, `REQ_SET_OUTPUT_GAIN 0x74`,
  `MASTER_VOL_*`), `audio_pipeline.c` (ramped `vol_mul_master_*`).
- **Strengths:** gains pre-computed to linear; master volume applied last; per-block ramp
  avoids zipper noise.
- **Missing:** no consolidated dB-vs-linear contract; no internal-headroom documentation
  (float vs. DAC full-scale vs. analog voltage) (see 6.4).
- **Recommended change:** documentation + headroom table. **Tests:** `tests/volume.py`.

### 5.8 Delay & alignment
- **Current implementation:** per-output circular delay lines
  `delay_lines[NUM_DELAY_CHANNELS][MAX_DELAY_SAMPLES]`; `channel_delay_samples[]` derived from
  ms + sample rate via `dsp_update_delay_samples()`.
- **Source:** `config.h:91-99`, `dsp_pipeline.h:18-25`, `audio_pipeline.c:617-672`,
  handler `REQ_SET_OUTPUT_DELAY 0x78`.
- **Strengths:** integer-sample delay, rate-aware, index-masked (branchless), automatic
  S/PDIF-vs-PDM latency compensation (`SUB_ALIGN_SAMPLES`).
- **Missing:** fractional-sample delay (not implemented - fine).
- **Correctness risks (flag):** README claims "up to 85 ms" but code caps at
  `MAX_DELAY_SAMPLES = 2048` = ~42.7 ms @48 kHz (46.4 ms @44.1 k, 21.3 ms @96 k); the
  `REQ_SET_OUTPUT_DELAY` handler does **not** clamp `delay_ms` to the ring length, so an
  over-large setting wraps the tap through `MAX_DELAY_MASK` (aliasing).
- **Recommended change:** clamp `delay_ms` in the handler to `MAX_DELAY_SAMPLES / Fs`; correct
  the README/spec numbers. **Tests:** impulse-position delay test.

### 5.9 Output limiter / protection  (see also 6.6)
- **Current implementation:** **There is no per-output limiter.** The only dynamics stage is
  the **Volume Leveller**, an *input-side, channel-linked RMS upward compressor* (threshold
  fixed -20 dBFS, soft knee 6 dB) with a **gain-cap "safety limiter" at
  `LEVELLER_LIMITER_CEIL = 0.70795` (-3 dBFS)** and an optional 5 ms lookahead. Sample-peak
  clip *detection* exists (`CLIP_THRESH_F=1.001`, sticky `clip_flags`).
- **Source:** `leveller.c`/`.h`, `config.h:60-61`, `Features/volume_leveller_spec.md`.
- **Strengths:** deterministic gain-cap semantics; linked detector/apply masks preserve image.
- **Missing:** per-output limiting; per-output threshold/attack/release; true-peak (oversampled)
  detection; explicit invalid-param behaviour; a guard so a preset/control error can never
  silently defeat protection.
- **Correctness risks:** the leveller is **not** a loudspeaker protection limiter and must not
  be presented as one. Its `enabled` flag can be cleared by a `REQ_SET_*`.
- **HW dependencies:** driver power handling (thermal/mechanical) - **unknown**.
- **Recommended change:** add a distinct, always-on per-output sample-peak limiter with
  testable semantics (6.6); leave the leveller untouched.
- **Tests:** deterministic impulse/step/sine/burst/over-threshold vector tests.


### 5.10 Metering & diagnostics
- **Current implementation:** `SystemStatusPacket{peaks[NUM_CHANNELS] (Q15), cpu0_load,
  cpu1_load, clip_flags (sticky)}`; per-slot buffer stats; S/PDIF starvation counters; PDM
  ring/DMA over/underrun counters; USB PHY error counters.
- **Source:** `config.h` (`SystemStatusPacket`, `BufferStatsPacket`), `audio_pipeline.c`
  (peaks in PASS 2 and PASS 5-7), `REQ_GET_STATUS 0x50`, `REQ_CLEAR_CLIPS 0x83`,
  `Features/peak_clip_metering_spec.md`, `buffer_statistics_spec.md`.
- **Strengths:** decimated/cheap metering; no unbounded work in the callback; per-core CPU.
- **Missing:** no gain-reduction meter; no limiter-activity indication (no per-output limiter).
- **Correctness risks:** `peaks` are Q15 clamped to 1.0 -> a clip is only knowable via
  `clip_flags`; block-peak, not true-peak.
- **Recommended change:** add gain-reduction fields when the limiter lands. **Tests:**
  `tests/diagnostics.py`.

### 5.11 Outputs (S/PDIF / I2S / PDM / DAC mute)
- **Current implementation:** 4 slots (RP2350) switchable S/PDIF<->I2S at runtime on PIO0;
  shared BCK/LRCLK (LRCLK = BCK+1, side-set constraint); optional MCK via CLK_GPOUTn; PDM sub
  on PIO1; optional GPIO hardware-mute line.
- **Source:** `main.c` (`rebuild_i2s_output_clocking`, output-type switch), `output_s24.h`,
  `pdm_generator.c`, `dac_hw_mute.c`, `Features/i2s_output_spec.md`,
  `dac_hardware_mute_spec.md`; local libs `pico_audio_spdif_multi`, `pico_audio_i2s_multi`.
- **Strengths:** synchronized multi-slot restart; glitch-free blackout; S24 clamp once/sample;
  hardware mute bracket around clock stop.
- **Missing:** **no DAC-chip-specific configuration** (no register init, no I2C/SPI config
  path); channel order / bit depth is whatever the external DAC expects.
- **HW dependencies:** all of it (see Sec. 7).
- **Recommended change:** none until the DAC is chosen. **Tests:** DAC acceptance checklist.

### 5.12 Persistence & protocol
- **Current implementation:** 10 preset slots (`SLOT_DATA_VERSION 35`, CRC32, wear-aware flash
  driver), whole-state wire image (`WIRE_FORMAT_VERSION 28`, `bulk_params.c`), notifications
  (v2 protocol), UART/I2C transports.
- **Source:** `flash_storage.c`/`.h`, `bulk_params.c`/`.h`, `notify.c`, `uart_control.c`,
  `i2c_control.c`, `Documentation/commands.md`, `notification_protocol_v2_spec.md`.
- **Strengths:** versioned wire/slot formats with documented migrations; chunked bulk transfer;
  CRC validation; a full protocol reference already exists.
- **Missing:** no application-level protocol version negotiation beyond `format_version`
  equality (a bump = hard reject, not negotiate).
- **Recommended change:** preserve; additive fields only (Rule 8). **Tests:**
  `tests/presets.py`, `tests/audio_loopback.py`.


---

## 6. DLMS feature inventory (status first, then gaps)

### 6.1 Channel and routing architecture
| Required capability | Status | Evidence / note |
|---------------------|--------|-----------------|
| Explicit input/output channel identification | IMPLEMENTED | `config.h` `CH_OUT_*`, `NUM_CHANNELS`, `NUM_INPUT_CHANNELS`. |
| Per-output source selection | IMPLEMENTED | `MatrixCrosspoint.enabled` per (in,out). |
| Per-output mute | IMPLEMENTED | `OutputChannel.mute` (`REQ_SET_OUTPUT_MUTE 0x76`). |
| Per-output gain | IMPLEMENTED | `OutputChannel.gain_db/gain_linear` (`0x74`). |
| Polarity inversion | IMPLEMENTED | `MatrixCrosspoint.phase_invert`. |
| Channel linking | PARTIAL | No link object; linking is writing identical params. |
| Mono summing with gain management | PARTIAL | Route both inputs to one output; no summing-gain compensation. |
| Safe behaviour when input/output unavailable | PARTIAL | Inactive inputs zeroed (PASS 2 tail); disabled outputs skipped. No fault state. |
| Feedback-routing prevention | MISSING | No guard in `REQ_SET_MATRIX_ROUTE`. |
| Routing validation | PARTIAL | Index bounds checked; no semantic validation. |
| Consistent GUI/firmware/physical mapping | N/A | No GUI exists (Sec. 8). Firmware mapping is authoritative. |
| Single authoritative channel model | IMPLEMENTED | Unified model since V16 (`config.h` comment). |

**Capability-honesty gap:** nothing exposes a machine-readable "how many outputs are physically
wired" query. A controller cannot distinguish logical channels from populated DAC channels.
Recommend an additive `REQ_GET_CAPS`-style query (new opcode, never redefining existing ones).

### 6.2 Crossover filters
| Required capability | Status | Note |
|---------------------|--------|------|
| Butterworth | IMPLEMENTED | `design_butterworth`, orders 1-8. |
| Linkwitz-Riley | IMPLEMENTED | `design_linkwitz_riley`, orders 2/4/6/8 (even only - correct). |
| Bessel | IMPLEMENTED | `design_bessel`, orders 2/4/6/8 (table-driven). |
| Configurable order | IMPLEMENTED | Encoded in `FilterType` 32-63. |
| Frequency validation vs Nyquist/hardware | PARTIAL / UNVERIFIED | Handler-level validation not confirmed this pass. |
| Stable coefficient generation | IMPLEMENTED (code) / UNVERIFIED (numeric) | Analytic poles; no numeric sweep run here. |
| Sample-rate-aware | IMPLEMENTED | `xover_recalculate_all(float sample_rate)`. |
| Safe coefficient updates while running | IMPLEMENTED (differs) | Core-1 block-boundary spin (`main.c ~2533`). |
| Bypass handling | IMPLEMENTED | `XoverFilter.bypass`, `xover_update_channel_bypass`. |

### 6.3 Parametric EQ
| Required capability | Status | Note |
|---------------------|--------|------|
| Per-output PEQ | IMPLEMENTED | 10 bands/channel, `filter_recipes[ch][band]`. |
| Freq/gain/Q controls | IMPLEMENTED | `EqParamPacket{freq,Q,gain_db}`. |
| Peak / low-shelf / high-shelf | IMPLEMENTED | `FILTER_PEAKING/LOWSHELF/HIGHSHELF` (+1st-order variants). |
| Validation + safe ranges | PARTIAL / UNVERIFIED | Range checks in handlers; not fully enumerated here. |
| Stable coefficients | IMPLEMENTED (code) | RBJ biquads + SVF hybrid. |
| Bypass | IMPLEMENTED | per-band `bypass`; `channel_bypassed[]`. |
| Sample-rate aware | IMPLEMENTED | recomputed on rate change. |
| Safe parameter updates | IMPLEMENTED (differs) | Block-boundary guard. |
| GUI accuracy | N/A | No GUI. |

### 6.4 Gain and polarity
| Required capability | Status | Note |
|---------------------|--------|------|
| Units/ranges defined | PARTIAL | Mixed dB (controls) and linear (`gain_linear`, `vol_mul` Q15, master vol linear). No single contract doc. |
| Conversion around silence | PARTIAL | `-inf` handled implicitly by very small linear gain; undocumented. |
| Numeric headroom | MISSING (documentation) | No stated headroom for float vs. S24 vs. analog. |
| Clipping behaviour | PARTIAL | S24 clamp in `output_s24.h`; sticky clip flags. |
| Gain staging through summing/routing | IMPLEMENTED | Matrix sums per-output. |
| Polarity semantics | IMPLEMENTED | `phase_invert` sign flip. |
| Smoothed gain changes | IMPLEMENTED | per-block `vol_mul_master_start/step` ramp. |

**Note:** internal float has no explicit headroom ceiling; clipping is only detected at the
output clamp. Document the mapping (internal float 1.0 -> S24 full scale -> DAC full-scale ->
unknown analog voltage).


### 6.5 Delay and alignment
See 5.8. Summary: IMPLEMENTED (integer, per-output, rate-aware) with the **unclamped max-delay
correctness risk** and a README/code number mismatch. No fractional-sample delay (acceptable).
Acoustic alignment is explicitly **not** claimed by the firmware.

### 6.6 Output limiter and loudspeaker protection  (major gap)
| Required capability | Status | Note |
|---------------------|--------|------|
| True/sample-peak limiter per output | **MISSING** | Only the input-side leveller exists. |
| Configurable threshold / attack / release | **MISSING** | Leveller is fixed-threshold, speed-preset only. |
| Peak detection | PARTIAL | Detection only (clip flags), no gain action. |
| Gain-reduction behaviour | PARTIAL | Leveller has gain smoothing but is not a limiter. |
| Defined behaviour on input > full scale | PARTIAL | S24 clamp (hard clip) at output. |
| Defined behaviour on invalid params | UNVERIFIED | Handler-level. |
| Protection vs coefficient/state corruption | MISSING | No such guard. |
| Predictable startup/recovery | PARTIAL | Leveller resets on active-set change. |
| Soft/hard limiting | MISSING | n/a. |
| True-peak estimate (oversampling) | **MISSING** | No oversampling in the dynamics path. |
| Thermal/mechanical modelling | **MISSING / HW-DEPENDENT** | No driver params, sensing, or model exist. |

**Rule 7/16:** the existing engine must not be relabelled; a sample-peak limiter must not be
called a true-peak limiter, and no thermal/mechanical claim may be made without driver data.

### 6.7 Metering and diagnostics
Feasible and largely present (Sec. 5.10). Gaps: gain-reduction meter, limiter-activity flag,
single aggregated dropout counter, explicit audio-clock status field. All are **additive** to
`SystemStatusPacket`/buffer-stats and should ride the existing decimated status path.

---

## 7. I2S and external multichannel DAC integration

**The DAC is unspecified, so no model-specific driver may be written.** What exists today:

- The firmware is a **generic I2S TX** (PIO0, left-justified 24-bit-in-32 frame, shared
  BCK/LRCLK, optional MCK via CLK_GPOUTn). It does **not** configure any DAC register.
- An optional **GPIO hardware-mute** line exists (`dac_hw_mute.c`) for DACs with a dedicated
  MUTE pin (PCM5102A XSMT, WM8741 MUTEB, AK4493 SMUTE). Register-only-mute DACs are explicitly
  out of scope (module header).
- The 8 logical S/PDIF/I2S channels map to **4 stereo slots**, each on one data line. How many
  independent analog outputs result depends entirely on the DAC.

### 7.1 Hardware compatibility checklist (to be completed with the real DAC)
| Item | Value | Status |
|------|-------|--------|
| Exact chip model + revision | ? | **UNKNOWN** |
| Physical DAC channels | ? | **UNKNOWN** |
| Serial data lines supported | ? | **UNKNOWN** |
| Framing (I2S / LJ / RJ) | firmware emits I2S left-justified 24-in-32 | to confirm vs DAC |
| Slot width / valid sample width | 32-bit slot, 24-bit valid | to confirm |
| BCK/LRCLK relationship | LRCLK = BCK+1; BCK from PIO divider | to confirm ratios |
| Master/slave | firmware is clock **master** for I2S TX | to confirm DAC accepts |
| MCLK requirement + ratio | optional MCK (128x/256x) via CLK_GPOUTn | to confirm DAC needs |
| Supported sample rates | firmware 44.1/48/96 kHz | to confirm DAC range |
| Clock tolerance | ? | **UNKNOWN** |
| Reset/config sequence | none in firmware | **MISSING / TBD** |
| Register init | none | **MISSING / TBD** |
| GPIO assignments | configurable via `REQ_SET_OUTPUT_PIN`, BCK pin, MCK pin | IMPLEMENTED |
| Voltage/logic level | RP2350 3.3 V | to confirm DAC I/O |
| Clock sharing across DACs | shared BCK/LRCLK/MCK | to confirm |
| Output channel order | slot order = Out1..8 | to confirm vs PCB |
| Analog output topology | not in repo | **UNKNOWN** |
| External hardware mute available | optional GPIO mute module | to confirm DAC pin |

### 7.2 Audio-clock integrity
Clock source is the RP2350 PLL at 307.2 MHz with integer PIO dividers at every supported rate
(`README.md`, `i2s_output_spec.md`). Buffer under/overrun is counted (`spdif_underruns`,
`pdm_ring_*`). A controlled transition on reconfiguration exists via synchronized slot restart
+ hardware mute + fade (`glitchfree_blackout_spec.md`). Long-duration stability and
clock-domain synchronisation between multiple DACs are **UNVERIFIED** (no hardware).

### 7.3 DAC driver acceptance tests (to run when hardware exists)
Channel ordering; sample framing; channel polarity; sample rate; silence + known vectors;
no interchannel mixing; no underruns at spec load; no clock drift/discontinuity; safe
startup/shutdown/reset. Until then these are **UNVERIFIED**; only host-side/static checks are
possible and none were run here.


---

## 8. Control application reality

**There is no first-party control application in this repository.** Searches for
`*.sln`, `*.csproj`, `package.json`, or `*.html` returned nothing. What exists is:

- **Python host/test tooling** under `tools/dspi_test/` (framework, `device.py`, test suites)
  and `tools/dspi_poke/dspi_poke.py` - a reference implementation of the USB control protocol,
  explicitly cited by `Documentation/commands.md` as "the reference host implementation".
- **A complete protocol reference** in `Documentation/commands.md` (transport conventions,
  every command, units, ranges, deferred-op behaviour) and
  `notification_protocol_v2_spec.md` (device->host events).
- **`webusb_spec.md`**: a browser-control design, marked **"design - not yet implemented"**.

Implications for the master prompt's Section 7 ("extend the existing DSPi interface"):

- The "existing interface" to extend is the **firmware vendor protocol**, not a GUI. Any Windows
  application would be **new work**, built against `commands.md` and `tools/dspi_test/device.py`.
- The prompt's proposed workspace pages are therefore design goals against a protocol that is
  already well-documented. The protocol already carries the data a DLMS UI needs: routing,
  per-output gain/mute/delay, PEQ, crossover, leveller, metering, presets, diagnostics.
- **Missing protocol surface for DLMS:** capability/availability query (Sec. 6.1), per-output
  limiter parameters + gain-reduction telemetry (Sec. 6.6/6.7), and any audio-clock status
  field. These should be added **additively** with a `WIRE_FORMAT_VERSION` bump and migration,
  never by redefining existing commands (Rules 8, 12).

---

## 9. Risk register (prioritized)

| # | Risk | Severity | Evidence | Mitigation |
|---|------|----------|----------|-----------|
| R1 | No per-output limiter; leveller can be disabled -> no loudspeaker protection | High (safety) | Sec. 5.9/6.6 | Add always-on per-output limiter with testable semantics |
| R2 | `REQ_SET_OUTPUT_DELAY` not clamped to `MAX_DELAY_SAMPLES` -> tap aliasing | High (correctness) | Sec. 5.8 | Clamp in handler; add test |
| R3 | README "85 ms delay" contradicts code (~42.7 ms @48 k) | Medium (docs) | Sec. 5.8 | Fix docs or raise limit deliberately |
| R4 | No true-peak/oversampled detection; only block sample-peak | Medium | Sec. 5.10/6.6 | Document limitation; add only if justified |
| R5 | Control + DSP share the main loop; a slow handler can delay audio | Medium (RT) | Sec. 3.1 | Keep `*_pending` discipline; add overrun counter |
| R6 | Coefficient updates use a Core-0 busy-wait spin | Low/Medium | Sec. 3.1 | Bounded; consider snapshots only with justification |
| R7 | No thermal/mechanical protection and no driver data | High if claimed | Sec. 6.6 | Do not claim; require driver params/sensing first |
| R8 | No DAC register init / chip detection | Medium (HW) | Sec. 7 | Resolve only when DAC chosen |
| R9 | No capability query; UI could show unsupported channels | Low/Medium | Sec. 6.1 | Add additive caps query |
| R10 | No protocol version negotiation (hard reject on mismatch) | Low | Sec. 5.12 | Preserve; document migration path |

---

## 10. Recommended staged plan (proposal, not executed)

Each step is small, reviewable, reversible, and touches one subsystem (Rules 9-10). None is
started without approval (Rule 13).

1. **Baseline reproducibility.** Get the Pico SDK submodule + toolchain, build `DSPi` for
   RP2350, run the CI build, record the artifact. (Prereq for every later claim.)
2. **Test-harness enablement.** Run `tools/dspi_test` against real hardware; capture a green
   baseline of `tests/*.py` so regressions are detectable.
3. **Fix R2 (delay clamp)** - smallest correctness fix; add an impulse-position test.
4. **Add per-output limiter (R1)** as a *new* pipeline stage with documented sample-peak
   semantics, per-output threshold/attack/release, and deterministic vector tests. No changes
   to existing stages. Bump wire format + preset version with migration.
5. **Metering extension (R4/R6)** - gain-reduction + limiter-activity fields, additive.
6. **Capability query (R9)** - additive `REQ_GET_CAPS`.
7. **Control application** - only after the protocol additions settle; build against
   `commands.md` + `tools/dspi_test/device.py`. Decide WebUSB vs. native WinUSB deliberately.

**Acceptance criteria per step:** builds clean; existing tests still pass; new deterministic
tests pass; no unrelated subsystem touched; docs updated (`current_architecture.md`,
`codebase_map.md`).


---

## 11. Explicitly unverified / hardware-dependent items

- Firmware build and all runtime behaviour (no toolchain here).
- Pico SDK version (submodule absent).
- Crossover/PEQ numeric stability and frequency/order validation boundaries.
- CPU headroom at 96 kHz with all outputs + all features enabled.
- All DAC electrical behaviour, channel order, clock ratios, long-term stability.
- Input-switch/preset-load audible transients.
- External DAC/amplifier clipping vs. the software clip flag.

---

## 12. Evidence index (primary sources read)

- `README.md`; `Documentation/commands.md`; `Documentation/codebase_map.md`;
  `Documentation/current_architecture.md`.
- `Documentation/Features/`: `i2s_output_spec.md`, `master_clock_spec.md`,
  `crossover_filters_spec.md`, `peq_filters.md`, `volume_leveller_spec.md`,
  `peak_clip_metering_spec.md`, `dac_hardware_mute_spec.md`, `buffer_statistics_spec.md`,
  `webusb_spec.md`, `notification_protocol_v2_spec.md`, `control_interfaces_spec.md`.
- Firmware: `config.h`, `audio_pipeline.c`, `dsp_pipeline.c`, `dsp_svf.h`, `dsp_biquad.h`,
  `crossover.c/.h`, `leveller.c/.h`, `output_s24.h`, `pdm_generator.c`, `dac_hw_mute.c/.h`,
  `usb_audio.c`, `main.c`, `vendor_commands.c`, `flash_storage.c/.h`, `bulk_params.c/.h`,
  `notify.c`, `uart_control.c`, `i2c_control.c`, `i2s_input.c/.pio`, `spdif_input.c`,
  `adat_input.c`.
- Build: `firmware/CMakeLists.txt`, `firmware/DSPi/CMakeLists.txt`, `memmap_dspi_rp2350_xip.ld`,
  `.github/workflows/cmake.yml`, `.gitmodules`.
- Tooling: `tools/dspi_test/` (framework/device/tests), `tools/dspi_poke/`,
  `tools/filter_tester/`, `scripts/check_ram_placement.py`.

---

*End of Phase-1 audit. Awaiting approval before any code modification (staged mode, Rule 13).*

