# Open ends

Resumption notes as of 2026-07-26. Design spec:
`docs/superpowers/specs/2026-07-26-daydreaming-trunk-design.md`.

Decisions with identified triggers live in **§12** of the spec and are not duplicated here.
This file holds everything *else* that was left hanging.

---

## 1. Not yet done

- **The milestone 0 implementation plan has not been written.** Agreed scope: milestone 0 only,
  ending at the seven characterisation deliverables in §9. Everything past it depends on
  numbers that don't exist yet.
- **The spec has not been reviewed.** Written and committed, but not read through by Daan.
  Flagged as worth actual attention: §2's accepted consequence (not steering away from the
  toddler trap), §4's spring-load argument, §9's gate thresholds, §12's five open questions.

## 2. Documents to obtain

- **The SpiRobs supplementary material — highest value missing item.** The paper repeatedly
  defers to it, and it contains what milestone 0 most needs:
  - *"see supplemental experimental procedures for a step-by-step build guide"* for the 1 m
    three-cable robot — i.e. the exact build we intend
  - Note S1 and Figure S2: printing and assembly guidelines
  - Notes S2–S3: workspace envelope derivation, MuJoCo reachability validation
  - Table S1: curvature parameters
  - Videos S1–S20, including S19 (1 m arm) and S9 (tip guided to target pose)
- **MuJoCo model provenance is unverified.** The paper validates reachability in MuJoCo, but
  whether the authors *published* a usable model is unknown. Milestone 1 estimate (2–4 weeks)
  assumes building one from scratch; it shrinks considerably if theirs is available. Check
  before planning milestone 1.
- **Cable spec.** The paper notes choosing a wear-resistant, low-friction cable but the
  extracted text did not give a clear diameter or material. Needed for the through-body hole
  sizing. In the supplement.

## 3. Assumptions about Daan's setup that swing the budget

Unconfirmed, and together they move the total by roughly €3–5k:

- Does a Bambu Lab X1C (or equivalent TPU-capable printer) already exist? Budgeted €1,200–1,500.
- Is there a GPU workstation available? Budgeted €2,000–4,000.
- Is there a wall that can be drilled and gone behind, in a space that can host open studio
  hours? The whole morphology argument (§4) assumes yes.

## 4. Deferred by choice, not oversight

Recorded in spec §13, restated so they aren't lost: audience **disclosure**; **exhibited
interiority** (the printed/projected LLM monologue); **gallery-grade reliability**; the
**two-act run**; and the **phase 2 specification**, which is gated on milestone 4's number.

## 5. Actions with no owner yet

- **Contact Wang, Freris & Wei at USTC.** Their stated top-priority open problem is closely
  this project's architecture (spec §14). No approach drafted.
- **Answer the privacy/GDPR question** before any public venue. Not blocking for open studio
  with disclosure; blocking for a real institution. Current design intent: local processing
  only, no audio retention, thermal rather than RGB.
- **Ask the LeRobot community** about servo duty cycle, if the STS3215 fallback is ever taken.

## 6. Housekeeping

- **No git remote.** No `gh` CLI, no SSH keys, no credential helper on this machine, so nothing
  can be pushed yet. Repo identity is set per-repo to `DaanArt <125995@gmail.com>`; global
  config (`DaanBio <daan@vandervorm.com>`) deliberately untouched.
- `.gitignore` excludes replay buffers, checkpoints and recordings. The replay buffer is
  supposed to persist indefinitely (§6) — **it needs a backup route that isn't git.**
- Calibration output (`calibration/spring_curve_*.csv`) is deliberately *not* ignored.

## 7. Threads raised and closed, for the record

So they don't get re-opened from scratch:

- Actuator selection converged after several revisions: Dynamixel XM430 → stepper+leadscrew →
  closed-loop stepper → STS3215 servo → **NEMA 17 + TMC2209 direct-drive on a drum.** The
  argument that settled it: the elastic spine makes the load a spring, so position control is
  force control, and the current limit is a safety ceiling. Series elasticity is expensive in
  rigid robotics and free in this morphology.
- Load cell mounting: inline-on-carriage and floating-motor-mount both rejected in favour of
  **idler pulley on a beam cell** (T√2 at 90° wrap), for vibration decoupling.
- FOC was investigated in depth and is an upgrade path, not a requirement. If taken, the right
  motor is a NEMA 17 (with TMC4671) rather than a gimbal BLDC — better torque, lower cost.
  Note that FOC without current sensing on the board is *not* torque control.
- Visual companion for brainstorming: declined, text only.
