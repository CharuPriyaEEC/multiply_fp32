# fmultiplier — FP32 Multiplier (Handshake, Multi-Cycle, IEEE-754)

## Overview
`fmultiplier` is a **multi-cycle** single-precision floating-point multiplier that accepts one operation at a time using a **valid/out_valid** handshake. Internally it runs a staged pipeline controlled by a small FSM (`counter`) and produces a 32-bit IEEE-754 binary32 result.

This design currently targets:
- Bit-accurate IEEE-754 binary32 multiplication for normal finite FP32 input operands,
- IEEE-754 round-to-nearest, ties-to-even (RNE) rounding,
- Correct handling of normal finite results, overflow to signed infinity, and underflow to signed zero for the supported evaluation domain,
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
- `a_e, b_e, z_e`: signed exponents represented internally in *unbiased* exponent domain (stored as 10-bit regs, used with `$signed`)
- `a_m, b_m, z_m`: mantissas extended to 24-bit with hidden 1 when applicable
- `product`: 50-bit product of mantissas
- `guard_bit`, `round_bit`, `sticky`: rounding support bits for RNE
-  All exponent arithmetic must be performed using signed values
---

## FSM / Pipeline Stages

The FSM is controlled by:
- `busy` (operation in progress)
- `counter` (stage number 1..7)

All stage actions are performed inside a single sequential always block using `case(counter)`.

### Stage 1 — Unpack
- Extract mantissas into 24-bit regs (initially `{1'b0, frac}`).
- Capture the operand signs.
- Convert the biased exponent fields to unbiased signed exponents:

    unbiased_exp = biased_exp - 127

- Reconstruct the 24-bit significands for normal inputs:

    significand = {1'b1, fraction}

### Stage 2 — Special classification + denormal setup
- Checks operand classes using `a_is_nan`, `a_is_inf`, `a_is_zero`, etc. (derived from `a_r/b_r` fields).
- For normal operation:
  - If exponent is nonzero => sets implicit leading 1: `a_m[23] = 1`.
  - If exponent is zero (subnormal) => forces exponent to -126 (subnormal exponent baseline).

> If you restrict inputs to **normal numbers only**, then:
> - `expA` and `expB` are always 1..254,
> - hidden-one insertion always happens,
> - special logic is bypassed in practice.


### Stage 3 — Prepare for Multiplication

For the supported normal-input domain, both significands are already normalized:

    1.xxxxxxxxxxxxxxxxxxxxxxx

No input normalization is required.

This stage may be used to register or prepare intermediate values for multiplication.

### Stage 4 — Multiply core
- Compute result sign: `z_s = a_s ^ b_s`
- Exponent add: `z_e = a_e + b_e`
- Mantissa product: `product = a_m * b_m * 4`
  - The `*4` scaling aligns the product for extraction into `{z_m, G, R, S}`.

The significand product represents a value in the range:

    [1.0, 4.0)

The product must subsequently be normalized.

If the product is greater than or equal to 2.0, the exponent must be incremented by one during normalization.

### Stage 5 — Extract mantissa + rounding bits
- `z_m = product[49:26]`
- `guard_bit = product[25]`
- `round_bit = product[24]`
- `sticky = OR(product[23:0])`

### Stage 6 — Normalize + Round-to-Nearest-Even (RNE)
This stage performs:
1. **Underflow alignment** toward exponent -126:
   - Computes shift amount `sh = (-126 - z_e)` when `z_e < -126`.
   - Shifts mantissa right and accumulates shifted-out bits into sticky.
2. **Normalize** if MSB missing:
   - Left-shifts mantissa while adjusting exponent, carrying guard into LSB.
3. **RNE rounding**:
   - If `G == 1` and `(R || S || LSB)` then increment mantissa.
   - Handles carry-out from rounding:
     - If rounding overflows mantissa, set mantissa to 0x800000 and increment exponent.


### Stage 7 — Pack
- For a normal path

  - Before packing:

    - detect exponent overflow and output signed infinity,
    - detect underflow-to-zero and output signed zero when applicable,
    - ensure rounding has already been completed.


  - Pack sign, biased exponent, fraction.
  - If exponent indicates overflow -> output INF.
  - If exponent indicates exact denorm boundary -> force exponent field to 0 (denormal/zero representation).
- Asserts `out_valid` for one cycle and clears `busy`.

---

## Assumptions & Constraints
- Inputs: `exp ∈ [1..254]` (no zeros/subnormals, no inf/nan)

However, the multiplication result may:

- be a normal finite FP32 value,
- overflow to signed infinity,
- underflow to signed zero.

The DUT must handle these output cases correctly.

---

## Verification Notes
1. Drive a and b.
2. Assert valid for one clock cycle while the DUT is idle.
3. Deassert valid.
4. Wait for out_valid.
5. Sample z when out_valid is asserted.
6. Compare z bit-for-bit against the expected IEEE-754 binary32 result.

The implementation must prioritize bit-accurate functional correctness for:

- sign calculation,
- exponent arithmetic,
- significand multiplication,
- normalization,
- IEEE-754 round-to-nearest, ties-to-even,
- overflow to signed infinity,
- underflow to signed zero.


---
