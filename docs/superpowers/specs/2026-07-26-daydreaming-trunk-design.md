# Daydreaming Trunk — Design Spec

**Date:** 2026-07-26
**Status:** Design approved, ready for implementation planning
**Scope of this spec:** Phase 1 (studio prototype + open studio showing). Phase 2 is sketched but not specified.

---

## 1. The work

A soft robotic trunk, mounted on a wall, that learns to move by learning a model of its own
body — and then learns which of its discovered movements make people delight in it.

It is not programmed with behaviours. It discovers what its body can do, in public, over
weeks. What it becomes is determined by how the room treated it.

## 2. Thesis

Two ideas, braided:

**Audience as reward function.** Viewers are the training signal. Not dwell time or clicks —
*delight*, specifically the quality of attention given to a small child: wonder, laughter,
admiration. Nobody in the room chooses to shape the creature, and it is shaped anyway.

**Learning in public.** The arc is the content. Day one it cannot hold itself up. Weeks later
it carries its own weight and turns toward a voice. A visitor who returns sees a different
creature.

### The consequence we accept

Delight rewards *effortful near-success*, not mastery. We coo at the wobble; when a child
walks competently the cheering stops. A creature honestly optimising for delight may
therefore learn that the optimum is to remain a beginner — charming, arrested, adored.

We do not design against this and we do not engineer toward it. The reward is left honest
(see §7), which means the exhibition may produce something tender or something bleak, and
that outcome is not the artist's to choose. This is accepted deliberately.

## 3. Architecture

Three layers at three timescales. The organising metaphor is **brain and cerebellum**: a
deliberative, semantic mind that decides *what* to want, over a motor system that learns
*how*, and that has no idea anyone is watching.

| Layer | Rate | Role | Reward | Implementation |
|---|---|---|---|---|
| **Brain** | minutes | Reads the room; names and selects goals | Delight (sparse, semantic) | LLM |
| **Manager** | seconds | Picks discrete goal codes | Delight + novelty | Director |
| **Cerebellum** | 20–50 Hz | Learns to *achieve* goals with this body | Curiosity (dense, local) | DreamerV3 |

The LLM is never in the control loop. An LLM call is 0.5–3 s against a 20–50 Hz control loop;
it belongs strictly in the slow layer, which is also the only layer where the question it
answers ("what do these people want?") is even well-posed.

### Why this resolves the sparse-reward problem

Delight is sparse, delayed and non-stationary — close to the hardest setting in RL. It is
hopeless as a signal for raw motor control: a gasp arriving 1.5 s after a movement cannot
assign credit across forty timesteps of cable commands.

The layer split moves it to the right altitude:

- **Curiosity drives skill *discovery*.** Dense, always available, requires no audience.
- **Delight drives skill *selection*** — choosing among ~10–20 already-discovered named
  moves given who is in the room. That is a contextual bandit over a small discrete set,
  learnable from a few dozen events an hour.

Same signal, tractable problem.

### The brain/cerebellum interface

Director compresses world-model states into discrete goal codes via a goal autoencoder, and
those codes decode back into images. That gives a concrete grounding path:

1. Cerebellum explores its body under curiosity and discovers reachable configurations.
2. Those become discrete goal codes — latent, undesigned, a record of what *this* trunk
   turned out to be able to do.
3. Each code is rendered as **real recorded video of the worker actually reaching it**
   (not the world model's blurry prediction — see §11).
4. The LLM captions it. That caption is a name.
5. The LLM learns which names earn delight in which contexts.

The vocabulary grows as the body discovers more. The brain does not invent it — it reads it
off a body it cannot directly command.

**Consequence worth stating:** the brain's words become *more true* over the run. On day one
"the tall reach" requests a flail; by week three the same request produces a reach. The
vocabulary didn't change — the body grew into it.

## 4. The body

**SpiRobs** (Wang, Freris & Wei, *Device* 2025) — 3D-printed logarithmic-spiral soft robots,
cable-driven, `r = ae^(bθ)` discretised in the angular domain so units taper base-to-tip.
Curvature radius decreases toward the tip, giving high tip flexibility (unlike
constant-curvature soft arms). An elastic layer on the central axis provides the restoring
force.

| Parameter | Value | Note |
|---|---|---|
| Variant | **3-cable** (spatial) | 2-cable is planar — insufficient for turning toward a voice |
| Length | ~1 m target; build 25 cm first | See §9 milestone 0 |
| Reference geometry | 42 units, 12 cm base, 5 mm tip | Paper's 1 m 3-cable build |
| Material | TPU 95A | Bambu Lab X1C |
| Construction | 4 segments, dovetail connectors | Exceeds printer bed |
| Elastic axis | 10% thickness (3-cable) | 5% for 2-cable |
| Mount | Wall, ~chest height | Motors behind wall |
| Action space | **3 continuous dimensions** | Cable tensions |

### Why wall-mounted

A cantilever puts gravity permanently against the trunk, which supplies the competence arc
for free: day one it hangs limp because it genuinely cannot hold itself up. It also puts the
motors behind the wall and out of the microphones' acoustic field (§5), and leaves no floor
footprint — the building appears to have grown a limb.

### Free reset

Release tension and the elastic spine returns the trunk to a known rest pose. No handler, no
cage, no intervention — the "no resets" condition DayDreamer had to engineer for is a
property of the morphology here. Also gives a free re-zero point for load sensing.

### Actuation

**3 × NEMA 17 stepper + TMC2209 driver, direct-drive on a cable drum**, at 24 V.

| Item | Spec | ~EUR/axis |
|---|---|---|
| NEMA 17 | ~0.45 Nm holding | 15 |
| TMC2209 | StealthChop, CoolStep, StallGuard4 | 8 |
| Load cell + HX711 | ~20 kg, via idler pulley (below) | 15 |
| AS5600 (optional) | Step-loss detection | 3 |

**~€38/axis, ~€115 for three**, plus a 24 V PSU and MCU.

**Spool sizing.** No 360° limit — a stepper counts turns freely, so force and travel are
decoupled. Small spool for force, multiple turns for reach:

| Spool radius | Travel/turn | Force @ 0.45 Nm | Speed @ 300 rpm |
|---|---|---|---|
| 8 mm | 50 mm | 56 N | 250 mm/s |
| **10 mm** | **63 mm** | **45 N** | **314 mm/s** |

Estimated cable travel is 20–40% of body length, so ~30 cm on a 1 m trunk — about 5 turns
at 10 mm radius. **That 20–40% is an assumption; measure real travel on the 25 cm test print
before committing.**

Higher bus voltage (24 V, not 12 V) matters: stepper torque collapses at high step rates
because winding inductance limits current rise, and voltage is the main lever against it.

### Why open-loop current limiting is sufficient here

SpiRobs is specified in cable *forces*, and a TMC2209 is a step/dir driver with a current
chopper — it has no torque command. That would normally be a problem, because stepper torque
is `τ ≈ τ_max(I) × sin(θ_commanded − θ_actual)`: current sets the *ceiling*, and the load
picks a point on the sine curve by however far the rotor lags.

**It is not a problem here, because the load is a spring.** The elastic spine is a monotonic
restoring force, so commanding position X yields tension F(X) deterministically and
repeatably. For a spring-loaded cable drive, **position control *is* force control** — the
two are the same variable, related by the spring constant. The current limit then serves as
a pure safety ceiling: a hand in the way means required torque exceeds the limit and the
motor stalls rather than escalating. Force capped by physics, not by software.

This is why the cheap option works here and would not work on a rigid arm. Series-elastic
actuation is the expensive trick in rigid robotics; SpiRobs has it built into the morphology
for free. **Measuring F(x) empirically is therefore a milestone 0 deliverable** — the whole
approach rests on that curve being well-behaved.

Consequences of open loop, and why each is acceptable:

- **Step loss is recoverable and honest.** Proprioception is camera-primary, so a slip shows
  up as prediction error — which is what the world model consumes. The logged action is what
  the policy commanded, so the mapping stays self-consistent; the model learns "commanding X
  usually gives Y," which is aleatoric noise that ensemble disagreement (§7) is built to
  handle. The elastic spine also gives a free re-home whenever wanted.
- **Detent notchiness is masked.** A NEMA 17 has meaningful detent torque, but the spine's
  compliance is *in series* with it — a visitor deforms the trunk long before they are
  fighting 20 mNm of cogging.
- **Noise is manageable.** StealthChop is the TMC2209's headline feature, and the motors are
  behind the wall.
- **Heat is manageable.** CoolStep scales current with load; rest tension is near zero
  because the spine does the returning.

**Alternatives considered and rejected as primary:** Feetech STS3215 serial bus servos
(~€25/axis, 3 Nm, position + load feedback, and **LeRobot supports them natively** — retained
as the lower-integration-effort fallback, but geared, so not backdrivable, and the 360° range
forces a ~96 mm spool). Dynamixel XM430 (~€280/axis, no advantage for 10× the cost). BLDC +
FOC and NEMA 17 + TMC4671 (true torque control and smooth yielding, ~€60–150/axis plus 2–4
weekends of commissioning per axis) — a refinement, not an enabler; see §12 Q1.

**Requirement: swappable actuator plate.** Design the mount and spool interface so a stepper,
a servo, or a BLDC bolts to the same plate with the same cable exit geometry. Costs nothing
now and makes the §12 Q1 decision an afternoon rather than a redesign.

### Cable tension sensing

**Idler pulley on a beam load cell**, behind the wall. The cable turns 90° over a small
pulley mounted on the load cell's free end; the resultant force on the pulley is
`2T·sin(φ/2)`, so exactly **T√2** at 90°, directed along the bisector.

Chosen over the two alternatives:

- *Inline on a linear carriage* — the most direct, and what the paper's slider rig would
  allow, but a slider gives up the multi-turn travel that makes the drum attractive.
- *Floating motor mount* — mount motor and drum on the load cell's free end so it becomes
  the flexure, measuring reaction force. No cable-path additions, but motor vibration couples
  straight into the sensor, which is a persistent noise floor at 80 SPS. Retained as fallback.

The pulley costs one wear point and one direction change of friction, and buys mechanical
decoupling from motor vibration — the right trade when the job is resolving small tension
changes for touch detection.

**Sizing:** at 60 N max cable tension the pulley sees ~85 N, so specify a **~20 kg cell** to
keep peak load near 50% of rating.

**Build requirements:**

- Hard mechanical overload stops either side of the flexure, so a yank cannot destroy the cell
- Sensing axis horizontal, or accept the assembly mass as a tared constant offset
- Purely axial loading — side load or bending moment corrupts the reading
- HX711 within a few cm of the cell, shielded twisted pair, routed away from stepper wiring.
  A 5 mV bridge signal next to stepper PWM is the classic week-long debugging trap
- Scheduled tare at rest (free, per the elastic spine) — cheap cells creep

**HX711 samples at 10 or 80 SPS.** Adequate for the 10–20 Hz control loop and for touch
detection; *not* a kHz force loop. If high-bandwidth force control is ever needed, that means
a different ADC (ADS1232 class), not a different load cell.

**Fit provisions, not parts, in milestone 0.** Put the pulley boss and cell bolt-holes in the
bracket and leave them empty. StallGuard4 plus the camera may prove sufficient; finding that
out is what milestone 0 is for, and retrofitting is then twenty minutes.

## 5. Sensors

### Proprioception

| Sensor | Role |
|---|---|
| **Camera watching the trunk** | **Primary.** Dreamer and Director are pixel-native |
| IMUs (2–3 along length) | Orientation, acceleration. Magnetometer unreliable near motors |
| Commanded step count | Cable position — open loop; optional AS5600 for slip detection |
| Load cells (§4) | Cable tension — distinguishes free motion from being held |

The camera is primary by design, not compromise: hand-building a clean proprioceptive state
vector for a soft body fights the tool. DayDreamer learned from images on several of its
robots.

**The camera must be framed so the audience is physically out of shot** — the trunk against
blank wall. This is a lens and geometry decision, not a software mask. See §7 (noisy TV).

Rejected: stretch/flex sensors (drifty, hysteretic, fatigue); fibre-optic and magnetic
tracking (research-grade cost).

### Touch

**Cable tension via load cells** (idler pulley, §4), thresholded. The paper validates the
principle on this exact mechanism — contact detected from motor load, sensitive enough to
register a **feather**, with a single threshold valid across the whole workspace — but it
inferred load from motor current. A load cell measures tension directly, which is both more
sensitive and the quantity SpiRobs is specified in.

Interim, before cells are fitted: **StallGuard4 plus the camera.** StallGuard is coarse and
speed-dependent, so it will not resolve a feather; the point of milestone 0 is establishing
what is actually insufficient.

Capacitive sensing was considered and dropped — the tension channel already exists in the
drivetrain.

Touch is a reward channel as well as a sensor. A stroke is delight expressed physically, and
it should reach the **brain**, not only the cerebellum.

### The room

| Sensor | Gives | Identifying? |
|---|---|---|
| **Mic array (4–6 ch)** | Laughter, gasps, speech, direction of arrival | Needs care |
| **Thermal array (MLX90640-class)** | Warm bodies and positions | **No** |

**Deliberately poor sensorium.** No faces, no facial-expression recognition, no RGB of the
audience. A creature that perceives the room as warmth and sound is a different artwork from
one running expression recognition — the first is alien and dignified, the second is a
surveillance apparatus that also happens to be sculpture. Thermal plus audio supplies
everything the delight signal needs, and rules out a category of problem before it exists.

### Which layer senses what

| Layer | Its senses |
|---|---|
| Cerebellum | Trunk camera, IMUs, encoders, load, touch |
| Manager | World-model latent state + coarse room state |
| Brain | Laughter/gasp events, transcript, warm-body positions, touch history, goal→reaction log |

**The cerebellum does not know we are there.** Only the brain does. Correct engineering,
correct biology, and it gives the piece an interior structure: a body absorbed in itself,
under a mind entirely preoccupied with us.

## 6. Learning stack

| Phase | Component | Source |
|---|---|---|
| 1 | DreamerV3, intrinsic reward only | `danijar/dreamerv3` |
| 1 | Latent-disagreement ensemble | Plan2Explore |
| 2 | Manager/worker, goal codebook | `danijar/director` |
| 2 | LLM brain: names codes, selects goals | new |

Phase 1 is not a throwaway warm-up. Plan2Explore's central claim is that pure exploration
yields a **task-agnostic world model** that adapts to downstream tasks few-shot. Phase 1
produces the substrate phase 2 stands on.

### Real-world training requirements

- **Asynchronous acting and learning.** Separate processes; the trunk never stalls waiting on
  a gradient step. One of DayDreamer's central engineering points.
- **Persistent replay buffer, retained indefinitely.** Simultaneously training set, life
  history, and the corpus the LLM reads when naming.
- **Safety limits in firmware, outside the learner.** Hard caps on current, temperature and
  excursion. The policy must not be *capable* of destroying the body — an exploring agent
  will find every way to hurt itself and it has weeks to look.
- **Action space: 3 cable-tension targets at 10–20 Hz**, with smoothing beneath. No raw
  high-rate commands; that shreds hardware in an afternoon.
- **Sim is a head start, not a dependency.** The paper validated workspace reachability in
  MuJoCo. Pre-train there, fine-tune on hardware. DayDreamer's whole point is that learning
  on hardware directly works.

## 7. Reward design

Two terms.

**Curiosity (intrinsic, dense) — cerebellum.** Latent disagreement, per Plan2Explore: an
ensemble of one-step latent predictors, reward = their disagreement.

**Not raw prediction error.** Raw error suffers the noisy-TV problem — it rewards things that
are unpredictable *and uncontrollable*. In this piece the noisy TV is literally the audience,
and a creature chasing prediction error would sit transfixed by the crowd and learn nothing
about itself. Ensemble disagreement distinguishes *unlearnable* from *not-yet-learned*:
members converge on "this is random" and agree, so reward decays.

Second mitigation: the trunk camera is framed to exclude the audience (§5).

**Delight (extrinsic, sparse) — brain.** Laughter, gasps, vocal prosody of wonder, people
calling others over, and touch. Selects among discovered named skills. Not shaped toward
either growth or the toddler trap (§2).

Curiosity is also not optional as an engineering matter: it is the exploration bonus that
makes early learning possible at all. Pure sparse delight cannot bootstrap.

## 8. Scope and staging

**Phase 1, studio prototype, no deadline**, shown publicly via **announced open studio
hours**.

Open studio was chosen over a short venue show, a long institutional run, or gallery-grade
from day one. It keeps the cheap build while still producing the thing phase 2 needs — real
delight data from real strangers — with the artist present and nothing required to survive
unattended.

**Destination (not this spec):** a two-act gallery run. Weeks 1–4 the creature is absorbed in
itself and ignores you; mid-run the brain comes online and it begins to notice the room.
Returning visitors watch it discover them. This needs both phases finished plus gallery-grade
reliability, and should be earned, not attempted first.

## 9. Milestones

Build the body small first: print the 25 cm two-cable version purely to validate the
print-and-assembly pipeline before committing to a 1 m four-segment build.

| # | Milestone | Deliverable | Est. |
|---|---|---|---|
| 0 | Body: 2-cable 25 cm → 3-cable ~50 cm → 1 m. Wall mount, motors behind wall, firmware limits | Teleoperable trunk + characterisation data — criteria below | 4–8 wk |
| 1 | Sim: MuJoCo model, 3-cable spiral, elastic spine, cable constraints | Sim runnable against DreamerV3 | 2–4 wk |
| 2 | Cerebellum in sim: DreamerV3 + Plan2Explore | Agent that explores its body in sim | 3–4 wk |
| 3 | **Cerebellum on hardware** ⚠️ **gate** | **Phase 1 complete** — criteria below | 4–8 wk |
| 4 | Delight sanity check — run *during* 3, not after | **A number: usable delight events/hour** | 1 wk |
| 5 | Open studio, announced hours | Real delight corpus from strangers | ongoing |
| 6 | Brain + Manager (phase 2) | Director codebook, LLM naming, bandit selection | later |

### Milestone 0 deliverables

The point of milestone 0 is not "a trunk that moves" — it is the measurements everything
downstream is sized against. Specifically:

1. **The spring curve F(x)** for each cable: commanded position → resulting tension, swept
   across the full range, in both directions so hysteresis is visible. The entire
   position-as-force-control argument (§4) rests on this being monotonic and repeatable. If
   it isn't, the actuator choice must be revisited.
2. **Real cable travel** from straight to full spiral, measured, not estimated. Decides spool
   radius. Replaces the 20–40% assumption.
3. **Real peak cable tension** required. Confirms or refutes the 45 N figure in §4.
4. **Fabrication pipeline validated** — 25 cm two-cable print and assembly working before
   committing to a 1 m four-segment build.
5. **Swappable actuator plate** and load-cell provisions (bosses and bolt-holes, unfitted).
6. **Whether StallGuard4 + camera suffices for touch**, or the load cells are needed.
7. **Re-measure F(x) after ~10 hours of running** — first data point on TPU fatigue, which
   the paper gives no figures for (§11).

Deliverable 7 is easy to skip and worth not skipping: if F(x) drifts materially in ten hours,
that reshapes the whole reliability picture before any of it is expensive.

### Milestone 3 gate criteria

If the world model will not learn on this body, nothing downstream matters. So the gate is
stated in terms that can pass or fail without appeal to taste:

1. **Model quality.** Multi-step latent prediction error over a 1 s horizon is materially
   lower than two baselines on held-out episodes: (a) a random-action policy's model, and
   (b) a constant-action (do-nothing) predictor. "Materially" = the gap is stable across
   three independent training runs.
2. **Workspace coverage.** Under curiosity alone, the trunk visits ≥60% of the reachable
   workspace cells, on the same 5 × 5 cm grid discretisation the paper uses for its grasping
   tests, within a defined session length.
3. **Goal-reaching repeatability.** Given a target configuration sampled from previously
   visited states, the agent reaches it within tolerance in ≥70% of attempts, across ≥20
   distinct targets.
4. **Survival.** Completes a full session with no firmware limit trips and no mechanical
   failure.

Criterion 3 is the one that most directly answers the paper's open problem (§14). Thresholds
are first-pass and may be revised once milestone 2 shows what sim achieves — but they must be
fixed *before* hardware runs begin, not after seeing the results.

**Milestone 4 is the cheap test that could save a year.** Mic array, laughter/gasp detection,
real people in the room, logged against a moving trunk. One number. 200 usable events/hour
means phase 2 is the tractable bandit described in §3. Six means phase 2 as conceived does
not work — and that must be discovered in week 10, not month 14.

### Timeline, honestly

Milestones 0–3, one person part-time: **6–9 months.** Full-time with help: 3–4. Then add two
or three body revisions, because the first trunk will be wrong. **Realistically about a year
part-time to a public phase 1.**

## 10. Budget (excl. labour)

| Item | ~EUR |
|---|---|
| Bambu X1C, if not owned | 1,200–1,500 |
| TPU 95A filament + spares | 200–400 |
| 3 × NEMA 17 + TMC2209 + 24 V PSU + MCU | 110–150 |
| Mic array (4–6 ch) | 70–200 |
| Thermal array (MLX90640-class) | 50–80 |
| Cameras, mount, wiring, PSU, enclosure | 600 |
| 3 × load cell + HX711 + idler pulley (fit when needed) | 45 |
| Optional: 3 × AS5600 for slip detection | 10 |
| GPU workstation, if needed | 2,000–4,000 |
| **Total** | **~€4.3–7.0k** |

Excluding the printer and GPU workstation, if you already have both: **~€1.1–1.5k.**

## 11. Risks

| Risk | Severity | Mitigation |
|---|---|---|
| **Delight too sparse to learn from** | Kills phase 2 | Milestone 4 answers it early and cheaply. **Fallback: touch** — dense, unambiguous, free to sense, feather-sensitive, and still audience-as-reward |
| **Autonomous 3-cable control is unprecedented** | High | The paper's 3-cable 1 m build was operated *by hand*; no autonomous control demonstrated. Sim first (M1–M2). This is also the contribution, not only the hazard — see §13 |
| **TPU fatigue over hundreds of hours** | Medium | No fatigue data in the paper. Print spares, log cycles, segmented build means replacing one section |
| **F(x) turns out non-monotonic or badly hysteretic** | Undermines the actuator choice | Milestone 0 deliverable 1 measures it before anything depends on it. Fallback: STS3215 with torque limit, or true torque control (§12 Q1) |
| Motor noise reaches the reward mic | Low–Medium | StealthChop, motors behind wall, damping in housing, mic array pointed into the room |
| **Load cell signal swamped by stepper PWM** | Medium | Shielded twisted pair, HX711 at the cell, physical separation from motor wiring, software averaging. Budget debugging time for this specifically |
| Silent step loss | Low | Camera-primary proprioception makes slip observable; optional AS5600. See §4 |
| Noisy TV: curiosity fixates on the crowd | Medium | Latent disagreement, not raw error; camera framed to physically exclude audience |
| Blurry decoded goal images degrade LLM naming | Low | Caption from real recorded video of the worker reaching the code, not model predictions |
| Goal codes drift when world model retrains | Low | Periodic re-naming passes. Also a legitimate event in the work: the brain relearning its own words |
| Sim→real transfer fails | Medium | Sim is a head start, not a dependency |
| **Privacy / GDPR on audience audio** | Blocking before any venue | Local processing only, no audio retention, thermal not RGB. Must be answered before a public venue; open studio with disclosure is acceptable interim |

## 12. Open questions

Not TBDs — decisions with identified triggers.

1. **Is being physically movable by a visitor part of the work?** Decides whether true torque
   control is worth adding — NEMA 17 + TMC4671, or BLDC + FOC. Does *not* gate the build,
   given the swappable plate (§4). Trigger: after the first open-studio sessions, watch
   whether people try to move it, and whether the notchiness of detent is perceptible through
   the trunk.
2. **Does the piece need to be silent?** Related but separable. StealthChop behind a wall is
   quiet, not inaudible. A constantly-buzzing creature is a worse artwork and a marginally
   worse reward signal. Trigger: first wall-mounted assembly — listen to it in the room.
3. **Real cable travel, peak tension, and F(x).** Decide spool radius and confirm the §4
   figures. Trigger: milestone 0, deliverables 1–3.
4. **Whether load cells are needed at all**, or StallGuard4 + camera suffices for touch.
   Trigger: milestone 0, deliverable 6.
5. **Delight events per hour.** Decides whether phase 2 proceeds as designed or falls back to
   touch. Trigger: milestone 4.

## 13. Deliberately deferred

- **Disclosure** — whether visitors are told they are training it
- **Exhibited interiority** — the LLM's running theory of the room, printed or projected. Build
  the capability, decide about showing it much later; it is the element most likely to tip twee
- **Gallery-grade reliability** — self-recovery, telemetry, watchdog, invigilator procedures
- **The two-act run** (§8)
- **Phase 2 specification** — depends on milestone 4's number

## 14. Why this is also research

SpiRobs' stated main limitation, and its authors' declared top priority for future work, is:

> "the absence of an inverse kinematic and dynamical model that would serve to infer the
> optimal cable actuation to reach a target configuration while also incorporating feedback."

A learned dynamical model of the body that infers cable actuation to reach target
configurations, with feedback, **is a world model plus a goal-conditioned worker.** The
paper's number-one open problem is, closely, this architecture.

This is not a solved problem dressed as art. It is an open problem in soft robotics attacked
through an artwork — which matters for funding, for writing, and as grounds to contact
Wang, Freris and Wei at USTC.

## 15. References

- **DayDreamer: World Models for Physical Robot Learning** — Wu, Escontrela, Hafner, Goldberg,
  Abbeel (2022). [arXiv:2206.14176](https://arxiv.org/abs/2206.14176). Quadruped learns to roll
  off its back, stand and walk in 1 hour, no simulator, no resets; adapts to perturbations in
  ~10 min.
- **Deep Hierarchical Planning from Pixels (Director)** — Hafner, Lee, Davidson, Norouzi (2022).
  [arXiv:2206.04114](https://arxiv.org/abs/2206.04114). Manager selects discrete goal codes,
  worker achieves them; goals decode to images.
- **Planning to Explore via Self-Supervised World Models (Plan2Explore)** — Sekar, Rybkin,
  Pertsch, Zhang, Abbeel, Hafner (ICML 2020).
  [arXiv:2005.05960](https://arxiv.org/abs/2005.05960). Latent-disagreement intrinsic reward;
  task-agnostic world model adapts few-shot.
- **SpiRobs: Logarithmic spiral-shaped robots for versatile grasping across scales** — Wang,
  Freris, Wei. *Device* 3, 100646 (18 April 2025).
  [doi:10.1016/j.device.2024.100646](https://doi.org/10.1016/j.device.2024.100646). Open access,
  CC BY-NC-ND. Local copy: `docs/trunk_paper.pdf`.
- **LeRobot SO-101** — [huggingface.co/docs/lerobot/so101](https://huggingface.co/docs/lerobot/so101).
  Open-source arm on Feetech STS3215 servos — the maintained driver stack behind the fallback
  actuation option (§4).
- **SimpleFOC** — [simplefoc.com](https://simplefoc.com). Note: true torque control requires
  current sensing on the board — voltage mode is not torque control. Relevant only if §12 Q1
  resolves toward FOC.
- **TMC2209 / TMC4671** — Trinamic (ADI) datasheets. StealthChop, CoolStep and StallGuard4 on
  the 2209; the 4671 does hardware FOC on two-phase steppers with encoder feedback, and is the
  upgrade path if §12 Q1 resolves toward true torque control.
