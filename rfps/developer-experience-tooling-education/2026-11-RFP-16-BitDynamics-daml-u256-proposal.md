# RFP #16: U256 for Daml on MainNet Now

## Development Fund Proposal

- Organization: BitDynamics
- Author / Primary Contact: Srikanth (srikanth@bitdynamics.me)
- Created: 2026-11-16
- Proposal Type: RFP-aligned
- RFP / Roadmap Area: RFP #16, Daml / Developer Usability Gaps (Daml U256 Support)
- Champion: Needs Champion
- Total Funding Request: 1,699,000 CC
- Project Duration: 22 weeks
- Label: daml-tooling

## Abstract
Daml cannot store or compute with unsigned 256-bit integers. Numeric holds 38 digits, so one EVM price × quantity product (2 × 10^39) overflows. Obsidian Systems will build daml-u256, a pure-Daml Apache 2.0 library with a U256 type, checked arithmetic, mulDiv, square root, scaling, and TypeScript and Java packages. It runs on today's stable Daml-LF, so teams can use it on MainNet at week 9, with no participant upgrade. 

## Specification

### 1. Objective

BitDynamics will give Canton applications exact 256-bit arithmetic on MainNet within this grant, using only the existing language. 

Out of scope: language changes, native bytes, signed and bitwise operations.

### 2. Implementation Mechanics

#### Representation

A U256 is a Daml record of nine Int fields, limb0 (least significant) to limb8, in base B = 2^30. Limbs 0 through 17 hold 30 bits each and limb 8 holds the top 16 bits, for exactly 256. Daml's Int is signed 64-bit and aborts on overflow, so four 64-bit limbs are impossible, and even 32-bit limbs overflow on one product. 30-bit limbs keep every intermediate below 2^61, leaving headroom under 2^63 (see the bound table).

- Canonical form. Each of limb0 through limb7 is in [0, 2^30) and limb8 is in [0, 2^16). Every value has exactly one encoding, so derived Eq is correct and equal amounts always serialize identically.
- Untrusted input. Any party can build the record directly, in a choice argument or through the Ledger API. Every public function checks canonical form first and aborts on a malformed value. isCanonical is exported for template ensure clauses, so a contract can never be created holding a malformed value.
- Ordering. The library defines Ord, so <, max and sorted maps are able to order values numerically.
- Size. A value is nine small integers in the contract payload.

#### Public API

| Module | Contents | Failure behaviour |
| --- | --- | --- |
| U256 | Type, zero, one, maxU256, add, sub, mul, div, remU256, pow, Eq/Ord | Aborts with a named error on overflow, underflow, division by zero or malformed input |
| U256.Checked | addChecked … powChecked | Returns Optional; never aborts on any input |
| U256.Math | mulDiv, mulDivRoundingUp, sqrt, rescale, pow10 | mulDiv aborts only if the quotient exceeds 2^256 − 1 or the divisor is zero |
| U256.Convert | Int, Numeric n, decimal Text, 0x hex (fixed 32-byte and minimal) | Conversions out of range abort; parse… functions return Optional |
| U256.Internal | Limb kernels | Assumes canonical operands; not a stable API |

Named functions are the stable interface. Additive, Multiplicative and Divisible instances let callers write a + b, with the same semantics. Large constants are built with fromText or pow10, since Daml integer literals are Int. No operation wraps modulo 2^256.

#### Algorithms

- Add, subtract, compare. Straight-line code over the nine fields, with no recursion or list traversal. A carry out of limb8 (sum ≥ 2^16) is overflow; a final borrow is underflow.
- Multiply. Row-wise schoolbook multiplication with the carry propagated at every step, producing the full 18-limb product. Column accumulation is not used, because nine 60-bit products summed in one column would pass 2^63. mul aborts if any limb above l8 is non-zero or l8 ≥ 2^16. mulDiv keeps the full product.
- Divide. One-limb divisors use short division. Larger divisors use Knuth's Algorithm D. The divisor is normalized so its top limb is at least 2^29. Each quotient digit is estimated from the top two limbs and corrected at most twice. Then comes a multiply-subtract, with an add-back when the estimate is one too high. Daml's / truncates toward zero, so each partial product is split into non-negative high and low parts before subtraction. No step divides a negative number.
- mulDiv(a, b, d). The 18-limb product is divided by d with the same Algorithm D. The result is exact when floor(a × b / d) fits in 256 bits, even when a × b does not. mulDivRoundingUp adds 1 when the remainder is non-zero, with an overflow check.
- Power. Square-and-multiply with a checked multiply at every step. For a base of at least 2, any exponent above 255 overflows and is rejected before computing, so no exponent causes long evaluation.
- Square root. Newton iteration from an upper-bound starting value, stopping when the estimate stops decreasing, with a fixed iteration cap. The result r satisfies r² ≤ n < (r + 1)², and tests check this on every vector.
- Decimal scaling. rescale amount fromDecimals toDecimals rounding multiplies or divides by a precomputed table of 10^0 to 10^77. When scaling down, the caller picks Down, Up or HalfEven. rescaleWithRemainder also returns the dropped units, so moving an 18-decimal token amount into a 10-decimal Decimal loses nothing silently.
- Text, hex, Numeric. Decimal parsing is multiply-by-10-and-add on limbs. It rejects empty strings, signs, whitespace and values over 2^256 − 1. Hex accepts an optional 0x, either letter case and up to 64 digits, and always outputs lowercase. Numeric n conversions go through exact decimal text, never through Int, so no step overflows for values up to Numeric's 38 digits.
- Daml-LF constraints. Daml-LF allows recursion only at top level. Every loop is therefore a top-level function with a fixed maximum depth: 18 limbs for division, 78 digits for decimal parsing, 64 for hex, and the cap for square root. Every operation has a known worst-case cost on a validator.

#### Bound table

The M1 design note lists every intermediate expression with its maximum value. The main ones:

| Step | Expression | Maximum | Limit |
| --- | --- | --- | --- |
| Add | a_i + b_i + carry | 2^31 − 1 | 2^63 − 1 |
| Subtract | a_i − b_i − borrow | ≥ −2^30 | −2^63 |
| Multiply row step | r + a_i × b_j + carry | 2^60 − 1 | 2^63 − 1 |
| Division estimate | u_top × B + u_next | 2^60 − 1 | 2^63 − 1 |
| Division multiply-subtract | q̂ × v_i + borrow | < 2^60 + 2^30 | 2^63 − 1 |
| Decimal parse | l_i × 10 + carry | < 2^34 | 2^63 − 1 |

#### Client packages

- Wire form. On the gRPC Ledger API, a U256 field is a record of nine int64 values. On the JSON Ledger API and in PQS, those values are strings.
- TypeScript (npm). toBigInt and fromBigInt convert between that JSON form and bigint. They accept the shapes produced by the JSON Ledger API, PQS and the Daml TypeScript codegen.
- Java (Maven Central). The same conversions between the Java codegen class and BigInteger.
- Validation. Both packages reject non-canonical input with the same rules as the Daml library, so a bad value cannot enter from either side.
- Releases. Packages are versioned in lockstep with the DAR and published with npm provenance and signed Maven artifacts.

#### Verification

- Differential testing. Python generates reference vectors with arbitrary-precision integers from pinned seeds. Daml Script runs every public operation over them, and CI fails on any mismatch or on drift when the vectors are regenerated. The corpus is split across scripts to stay under the script service's evaluation time limit.
- Edge corpus. 0, 1, 2^255 and 2^256 − 1. Every limb boundary 2^(30k) ± 1. Maximum carry and borrow chains. Products just above and below 2^256. Divisors of every limb length.
- Properties. q × d + r = a with r < d. mulDiv matches the reference wherever the result fits. Checked and aborting variants agree on every input. Text, hex and Numeric conversions round-trip. Malformed records are rejected by every public function.
- Seeded faults. CI also builds deliberately broken variants, such as a dropped carry, an off-by-one estimate or a skipped canonical check, and the suite must fail on each one. This shows the tests can detect real errors.
- End-to-end. On LocalNet, contracts holding U256 fields are created and read back through the JSON Ledger API and PQS, then decoded in TypeScript and Java and checked against the reference.
- Benchmarks. Per-operation timing with warm-up, raw samples and median/p95. Serialized size per value. A representative swap step compared with the same workflow in Numeric.

#### Release and upgrade

- Package. The DAR depends only on daml-prim and daml-stdlib and targets the current stable Daml-LF version.
- Releases. Tags publish the DAR, its package ID, checksums and build instructions.
- Upgrades. The U256 record is frozen at v1.0. Every v1.x release is checked in CI with the Smart Contract Upgrade checker against v1.0, so contracts storing U256 keep working across library upgrades.
- Distribution. Releases also go to the Daml package registry by Obsidian (PR #533).

### 3. Architectural Alignment

RFP #16 asks for "unsigned 256-bit integer workflows commonly needed in token, DeFi, and digital asset applications." The library depends only on daml-stdlib and adds no work for the Daml language or runtime teams.

### 4. Backward Compatibility

No backward compatibility impact. Validators, participants and existing codegen are unchanged. The record shape freezes at v1.0.

## Milestones and Deliverables

### Schedule

| Weeks | Work |
| --- | --- |
| 1–2 | Design note and bound table. Record type, canonical check, comparison. Reference-vector generator and CI. Design partners confirmed. |
| 3–5 | Add, subtract, multiply, short division, Algorithm D, power. Int, Text and hex conversions. Edge corpus, seeded faults. v0.1 tag. |
| 6–7 | mulDiv, square root, rescale, Numeric conversions. Design-partner feedback folded in. |
| 8–9 | TypeScript and Java packages. LocalNet end-to-end tests. Benchmarks, docs, SCU baseline. v1.0 tag. |
| 10–22 | Adoption support at about 20 engineer-hours a week. v1.x fixes. Adoption report and handover. |

### Milestone 1: Core U256 (v0.1)

- Estimated Delivery: Week 5
- Focus: A correct, storable U256 that design partners can build against.
- Deliverables:
  - Design note: representation, canonical-form rules, error semantics, and the full bound table for every intermediate value. Posted for comment in the Daml Language & Developer Tooling SIG.
  - daml-u256 v0.1 DAR with the U256, U256.Checked, U256.Convert (Int, Text, hex) and U256.Internal modules.
  - Public CI running the differential corpus, the edge corpus, forced Algorithm D correction paths with the binary-division oracle, property tests and seeded-fault builds.
  - API reference for the v0.1 modules and a quick-start example.
- Value Metrics:
  - 2 design partners outside Obsidian build a prototype contract against v0.1.
  - Each design partner's written feedback is published with their permission, along with Obsidian's response.
- Acceptance evidence: v0.1 tag and commit SHA, DAR package ID, link to a green CI run, the design note, and the design-partner feedback.

### Milestone 2: v1.0 Release

- Estimated Delivery: Week 9
- Focus: Everything an adopter needs to ship on MainNet.
- Deliverables:
  - U256.Math: mulDiv, mulDivRoundingUp, sqrt, rescale, rescaleWithRemainder, pow10.
  - Numeric n conversions in U256.Convert.
  - TypeScript package on npm and Java package on Maven Central, both with validation matching the DAR.
  - LocalNet end-to-end tests through the JSON Ledger API and PQS, decoded in both client packages.
  - Benchmark report: per-operation timing, serialized size, and a representative swap step against Numeric.
  - Documentation: API reference, U256 vs Numeric guide, migration guide for 18-decimal token amounts, and two worked examples (a token-amount settlement and a constant-product swap step).
  - v1.0 tag with frozen record shape, published package ID and checksums. SCU check baseline in CI. Release also submitted to the PR #533 registry if it is live.
- Value Metrics:
  - 1 outside team depends on v1.0 in a public or shared repository.
  - Every issue raised by M1 design partners is closed or documented with a disposition.
- Acceptance evidence: v1.0 tag, DAR package ID, npm and Maven package versions, link to a green CI run including end-to-end tests, the benchmark report, and the documentation.

### Milestone 3: Adoption and Handover

- Estimated Delivery: Week 22
- Focus: Outside teams run daml-u256 in deployed applications, and the project can be maintained without the grant team.
- Deliverables:
  - Integration support for adopting teams: office hours, code review of their U256 usage, and fixes.
  - v1.x releases for adoption-blocking bugs, each passing the SCU check against v1.0.
  - Adoption report: each adopter, its network, the operations it uses and the issues it raised.
  - Maintainer handover: maintainer guide, release runbook, SECURITY.md with a private reporting contact, and the post-grant maintenance plan.
- Value Metrics and payment:
  - USD 25,000 in CC per accepted adopter, up to 3.
  - The third tranche requires at least one accepted adopter to be on MainNet.
- Adopter evidence:
  - The adopter's package ID showing a dependency on a released daml-u256 DAR.
  - A transaction or contract ID on TestNet or MainNet that exercises a daml-u256 function in the adopter's workflow.
  - Written confirmation from the adopting organization to Tech & Ops.
- Acceptance evidence: adoption report, handover documents in the repository, and the evidence above for each adopter.

## Acceptance Criteria

- Each milestone’s deliverables are tagged and build from the public Apache 2.0 repository.
- Every public operation matches the reference corpus; every abort case aborts.
- Public functions reject non-canonical input; seeded-fault builds fail CI.
- Values round-trip losslessly through the JSON Ledger API, PQS, TypeScript and Java.
- Every v1.x release passes the SCU check against v1.0.
- Adoption: an outside organization whose TestNet or MainNet application runs a released DAR in a deployed workflow, confirmed in writing to Tech & Ops.

## Funding

Total Funding Request: 1,699,000 CC.

| Milestone | Deliverable | Week | Effort (hrs) | CC |
| --- | --- | --- | --- | --- |
| M1 | Core U256 v0.1: representation, arithmetic, division, Int/Text/hex conversions, bound table, verification suite. 2 design partners prototype. | 5 | 234 | 409,500 |
| M2 | v1.0: math layer, Numeric conversions, TypeScript and Java packages, LocalNet end-to-end tests, benchmarks, docs. 1 outside team depends on it. | 9 | 194 | 339,500 |
| M3 | Adoption: integration support, v1.x fixes, adoption report, maintainer handover. 250,000 CC paid per adoptor (up to 3). | to 22 | 292 | up to 750,000 |
| Total |  | 22 | 720 | 1,499,000 |

## Volatility Stipulation

The project duration is under 6 months. Should the project timeline extend beyond 6 months due to Committee-requested scope changes, any remaining milestones must be renegotiated to account for significant USD/CC price volatility.

## Co-Marketing

At v1.0, BitDynamics will announce with the Foundation and publish an EVM porting guide.
