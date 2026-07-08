# fmultiplier — FP32 Multiplier (Handshake, Multi-Cycle, IEEE-754)

## Overview
`fmultiplier` is a **multi-cycle** single-precision floating-point multiplier that accepts one operation at a time using a **valid/out_valid** handshake. Internally it runs a staged pipeline controlled by a small FSM (`counter`) and produces a 32-bit IEEE-754 binary32 result.

This design currently targets:
- **Bit-accurate results for normal FP32 numbers** (typical IEEE-754 behavior with round-to-nearest-even),
- Deterministic latency (fixed number of cycles from `valid` to `out_valid`),
- The design behaves as: z = a*b 
- z, a and b are single precision 32-bit IEEE-754 numbers

---

## Interface

### Ports
| Port | Dir | Width | Description |
|------|-----|-------|-------------|
| `clk`   | in | 1 | Clock |
| `rst`   | in | 1 | Async reset (posedge) |
| `valid` | in | 1 | **1-cycle start pulse**; accepted only when not busy |
| `a`     |  in | 32 | Operand A (FP32 bits) |
| `b`     | in | 32 | Operand B (FP32 bits) |
| `z`         | out | 32 | Result (FP32 bits) |
| `out_valid` | out | 1 | **1-cycle pulse** when `z` is updated/valid |

### Handshake contract
- When `busy==0`, a high `valid` on a rising edge **starts** an operation:
  - `a` and `b` are **registered** into internal regs `a_r` and `b_r`.
  - The FSM begins at `counter = 1`.
- While `busy==1`, new `valid` pulses are **ignored**.
- When the operation completes:
  - `z` is updated,
  - `out_valid` pulses high for 1 clock cycle,
  - `busy` is cleared.

---

## Latency and Throughput

### Latency
- Fixed latency of **7 stages**.
- In this implementation the operation begins at stage `counter=1` and completes at `counter=7`.
- `out_valid` asserts on the cycle where stage 7 packing finishes.

A safe expectation for system-level timing is:
- **`out_valid` occurs 7 clock cycles after the start edge** (the clock edge where `valid` was sampled when idle).

### Throughput
- **Not pipelined** (single-issue).
- Max throughput is **1 result per 7 cycles** (assuming `valid` is asserted only when idle).

---

## Internal Data Model (IEEE-754 binary32)
For each operand:
- `sign` = bit 31
- `exp`  = bits 30:23 (biased exponent)
- `mant` = bits 22:0 (fraction)

Internal signals:
- `a_s, b_s, z_s`: sign bits
- `a_e, b_e, z_e`: signed exponent in *unbiased* domain (stored as 10-bit regs, used with `$signed`)
- `a_m, b_m, z_m`: mantissas extended to 24-bit with hidden 1 when applicable
- `product`: 50-bit product of mantissas
- `guard_bit`, `round_bit`, `sticky`: rounding support bits for RNE
- `special_case`, `special_z`: a latched flag and latched result value used to short-circuit Stage 7's packing when Stage 2 classifies an operand as NaN, infinity, or zero (see the special-case discussion under Stage 2)

> **Reset note:** on the asynchronous reset, every register in this list should be driven to a
> defined value (typically zero), not just `counter` and `busy`. Leaving any of the mantissa,
> exponent, sign, product, rounding-bit, or special-case registers unreset means their value
> after reset is whatever the simulator happens to initialize them to, which is a common source
> of intermittent, hard-to-reproduce mismatches on the very first operation after reset.

---

## FSM / Pipeline Stages

The FSM is controlled by:
- `busy` (operation in progress)
- `counter` (stage number 1..7)

All stage actions are performed inside a single sequential always block using `case(counter)`.

> **Implementation note (applies to every stage, but matters most in Stage 6):**
> Within one `case` branch, later computations sometimes depend on the *result* of earlier
> computations in that same branch (e.g. "shift the mantissa, then decide whether to round
> the shifted mantissa"). A chain of plain non-blocking assignments to the same signal
> (`sig <= ...` followed later in the same branch by another statement that reads `sig`)
> will **not** see the updated value — it will see the value `sig` held before this clock
> edge, because non-blocking assignments only take effect at the end of the time step.
> Whenever a stage's steps are described as sequential ("first do X, then based on the result
> of X do Y"), compute the intermediate results with local variables (e.g. `reg` declared in
> the block and updated with blocking `=` assignment) and only assign the final value to the
> real state register with `<=` at the end of the branch. This preserves the intended
> sequential/combinational relationship between steps that happen "in the same cycle."

### Stage 1 — Unpack
- Extract mantissas into 24-bit regs (initially `{1'b0, frac}`).
- Convert biased exponent into unbiased form: `exp - 127`.
- Capture signs.

### Stage 2 — Special classification + denormal setup
- Checks operand classes using `a_is_nan`, `a_is_inf`, `a_is_zero`, etc. (derived from `a_r/b_r` fields).
- If any special condition is detected, latch `special_case` and set `special_z` to the
  appropriate result, checked in priority order (first match wins): first, if either operand is
  NaN, or if one operand is infinity and the other is zero, the result is the canonical quiet
  NaN, `32'h7FC0_0000`; next, if either operand (but not both, and not paired with a zero
  operand) is infinity, the result is a signed infinity, sign equal to the XOR of the operand
  signs, `{a_s ^ b_s, 8'hFF, 23'd0}`; finally, if either remaining operand is zero, the result is
  a signed zero, `{a_s ^ b_s, 8'd0, 23'd0}`. NaN payload bits from the operands are never
  propagated — every NaN result collapses to the same canonical bit pattern regardless of which
  operand was NaN or what its payload bits were.
- For normal operation:
  - If exponent is nonzero => sets implicit leading 1: `a_m[23] = 1`.
  - If exponent is zero (subnormal) => forces exponent to -126 (subnormal exponent baseline).

> If you restrict inputs to **normal numbers only**, then:
> - `expA` and `expB` are always 1..254,
> - hidden-one insertion always happens,
> - special logic is bypassed in practice.

### Stage 3 — Input normalization (lightweight)
- If mantissa MSB is not set, shift left and decrement exponent.
- This is mainly relevant for denormal handling; for strictly normal inputs, this typically does nothing.

### Stage 4 — Multiply core
- Compute result sign: `z_s = a_s ^ b_s`
- Exponent add: `z_e = a_e + b_e + 1`
- Mantissa product: `product = a_m * b_m * 4`
  - The `*4` scaling aligns the product for extraction into `{z_m, G, R, S}`.

> **Why the `+1`?** This is worth deriving explicitly rather than treating as a fixed constant,
> since it is easy to "simplify away" if re-derived casually. Both `a_m` and `b_m` carry their
> implicit leading 1 at bit position 23, so each represents a value in `[2^23, 2^24)`. Their
> raw product therefore lands in `[2^46, 2^48)`, i.e. it occupies 47 or 48 bits before any
> scaling. The `*4` (a left-shift by 2) moves this into a 50-bit-wide `product` register, now
> occupying the range `[2^48, 2^50)`. Stage 5 then extracts `z_m` from `product[49:26]` — a
> right-shift of 26 bits — which is calibrated so that `z_m` again lands with its own implicit
> leading 1 at bit 23, in the same `[2^23, 2^24)` convention used for `a_m`/`b_m`. Tracking the
> bit position of the leading 1 all the way through — from bit 23 in each operand, to roughly
> bit 46/47 in the raw product, to bit 48/49 after the `*4` scaling, to bit 23 again after the
> Stage 5 extraction — shows that one extra power of two survives the round trip beyond what a
> naive `a_e + b_e` would capture; that surviving factor of two is exactly the `+1` unbiased
> exponent correction applied here. Omitting it silently shifts every result by a factor of 2x.

### Stage 5 — Extract mantissa + rounding bits
- `z_m = product[49:26]`
- `guard_bit = product[25]`
- `round_bit = product[24]`
- `sticky = OR(product[23:0])`

### Stage 6 — Normalize + Round-to-Nearest-Even (RNE)

This stage has three sequential sub-steps. Each sub-step's output feeds the next one **in the
same cycle**, so compute them with local variables as described in the implementation note
above, then write the final values to `z_m, z_e, guard_bit, round_bit, sticky` at the
end of the branch.

> **This is the single highest-frequency bug location in the whole design.** Sub-step B (the
> normalize shift) and Sub-step C (the rounding decision) are described here as if they happen
> "in the same cycle," but if each is written as a direct non-blocking assignment to the real
> state registers (`z_m <= ...`, then later in the same branch reading `z_m` to decide whether
> to round), the rounding decision will read the **pre-shift** value of `z_m`, not the
> just-normalized one — because non-blocking assignments do not update their target until the
> end of the time step. The result is a rounding decision made against stale data: the guard/
> round/sticky bits and the mantissa parity bit (`zm[0]`) used in Sub-step C must reflect the
> *already-normalized* mantissa from Sub-step B, not the mantissa as it stood before
> normalization. The only reliable way to guarantee this chaining is to keep `zm, ze, g, r, s`
> as local variables updated with blocking assignment throughout Sub-steps A, B, and C, and only
> commit the final values to `z_m, z_e, guard_bit, round_bit, sticky` with non-blocking
> assignment once, at the very end of the branch.

Work on local copies `zm, ze, g, r, s` initialized from `z_m, z_e, guard_bit, round_bit, sticky`.

**Sub-step A — Underflow alignment (only if `$signed(ze) < -126`):**
- Let `sh = -126 - $signed(ze)` (number of extra right-shifts needed to reach the subnormal exponent floor).
- If `sh >= 24`: the entire mantissa shifts out. Set `zm = 0`; if the original `zm` was nonzero, remember that a bit was lost.
- Else: any of `zm`'s low `sh` bits that are `1` represent bits shifted out — remember that a bit was lost if any of them are `1`. Then `zm = zm >> sh`.
- The previous `g` and `r` also get shifted past the rounding position, so they also count as lost bits if either was `1`.
- Fold every "lost bit" from this sub-step into `s` (i.e. `s = s | lost_any`).
- Set `g = 0`, `r = 0`, and `ze = -126`.

**Sub-step B — Normalize (only if sub-step A did *not* fire, and `zm[23] == 0`):**
- `ze = ze - 1`
- `zm = {zm[22:0], g}` (shift mantissa left by 1, pulling in the old guard bit)
- `g = r`
- `r = 0`

**Sub-step C — Round-to-Nearest-Even (always, using the `zm/g/r/s` produced by A/B above):**
- Round up if `g && (r || s || zm[0])` (i.e. guard set, and either round/sticky set or the result is currently odd — the "round to even" tie-break).
- If rounding up: `zm = zm + 1`. If that addition carries out of bit 23 (i.e. `zm` was `24'hFFFFFF`), the result becomes `zm = 24'h800000` and `ze = ze + 1` (mantissa overflow bumps the exponent).

Write the final `zm, ze, g, r, s` back to `z_m, z_e, guard_bit, round_bit, sticky`.

### Stage 7 — Pack
- Default: pack as a normal number: `z = {z_s, z_e[7:0] + 127, z_m[22:0]}`.
- **Overflow:** if `$signed(z_e) > 127`, force `z = {z_s, 8'hFF, 23'd0}` (infinity) instead.
- **Subnormal/zero boundary:** if `$signed(z_e) == -126` and `z_m[23] == 0`, force the packed
  exponent field to `8'd0` (leave the mantissa field as computed by Stage 6 — do not add any
  further general denormal-repacking logic beyond this single boundary check; it is not needed
  under the normal-inputs-only assumption below).
- **Special case override:** if `special_case` was latched in Stage 2, `z` is driven from the
  latched `special_z` instead of the computed packing above.
- Asserts `out_valid` for one cycle and clears `busy`.

---

## Assumptions & Constraints
- Inputs: `exp ∈ [1..254]` (no zeros/subnormals, no inf/nan)

---

## Verification Notes
Recommended testbench behavior for this handshake design:
- Drive `a/b` and pulse `valid` **synchronously** on clock edges.
- Wait for `out_valid` before sampling `z`.
- Generate only normal operands,

---
--
