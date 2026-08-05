# Halloween Candy Dispenser — SO-ARM100 Project Plan
**Phase 1 (Aug 4 → Oct 31, 2026): Safety + Imitation Learning | Phase 2 (post-Halloween): VLA + RL**

Two things are true at once: this runs near children, so it needs a guaranteed-safe fallback that ships early and never depends on ML working — and you're a beginner with ~1–2 hrs/evening, so this phase focuses entirely on **imitation learning**, done well, rather than spreading thin across IL/VLA/RL before Halloween. **Tier 1 (safety + fixed fallback) is your insurance policy, built in Week 1–2 no matter what.** VLA and RL become Phase 2, once you actually know the platform — see the bottom of this doc for how that phase will build on what you do now.

---

## 0. Safety first — non-negotiable, before anything else

- **Force/speed limiting:** cap servo torque and joint velocity so it can't injure someone even at full extension into an obstruction. Test by putting your own hand in the arm's path at max settings.
- **Physical workspace limiting:** mount, orient, and joint-limit the arm so it is *physically impossible* for the end-effector to reach face/eye height of a kneeling child — a hard mechanical/software constraint, never something you rely on a policy to respect.
- **Hard e-stop:** a physical, always-reachable kill switch that cuts motor power, not a keyboard Ctrl+C.
- **Barrier/enclosure:** plexiglass, table edge, or rope barrier keeping kids' hands out of the motion envelope except at a defined candy pickup point.
- **Supervision:** an adult present with a hand near the e-stop the entire time it runs on Halloween night. Hard requirement.
- **Fallback mode:** a non-autonomous mode (manual teleop, or a fixed pre-recorded motion on a button press) you can drop into instantly if the learned policy misbehaves.

Build the e-stop and workspace limits into the control stack this week — every later stage inherits them for free.

---

## 1. Imitation learning, in one paragraph

You teleoperate the follower arm through the task by hand (using the leader arm), the software records (camera images, joint positions) → (next action) pairs, and a neural net learns to copy you — "behavior cloning." You'll use **ACT** (Action Chunking Transformer, a good default for short manipulation tasks, works with modest dataset sizes) and, if needed, **Diffusion Policy** (more robust when your demos have multiple valid ways to do the task, but pricier to train). Both are ready-made recipes in LeRobot — you're not building an architecture from scratch.

Stack: `lerobot` (Hugging Face — teleop, dataset recording/format, ACT/Diffusion Policy training, real-robot drivers) · optionally Isaac Sim for URDF/USD import and synthetic lighting/domain variety later · your GPU.

---

## 2. Task tiers — pick the floor you ship no matter what

1. **Tier 1 — guaranteed fallback, no ML:** button/mat press → arm runs a fixed pre-recorded motion, drops candy into a bucket at a marked spot. Built in Week 1–2. This is what runs if anything else fails on the night.
2. **Tier 2 — IL (this phase's target):** arm picks a candy from a bin and places it into a bucket at a roughly consistent position, using a *simple* vision cue for the trigger — a marker or reflective tape on the bucket beats general detection, especially at night. Don't reach for a sophisticated model if tape solves it.
3. **Tier 3 — VLA + RL (Phase 2, next project):** general hand/bucket detection, adaptive grip for candy variety, RL-sharpened handoff timing. Deferred until you're comfortable with the platform.

---

## Week 1 (Aug 4–9) — Setup + safety design, then hardware arrives

**Day 1 (Tue, today):** Install `lerobot` + PyTorch, verify GPU. Skim the LeRobot docs structure (teleoperators, datasets, policies).

**Day 2 (Wed):** Read through LeRobot's SO-101 getting-started guide end to end before touching hardware — know the calibration and teleop steps before you're standing in front of the arm.

**Day 3 (Thu):** On paper: design e-stop wiring, mounting orientation, and joint limits so the end-effector physically cannot reach face height. Order/gather e-stop hardware and barrier materials.

**Day 4 (Fri):** (Optional) Install Isaac Sim, import the SO-ARM URDF, convert to USD — useful later for generating varied-lighting synthetic data, not required for core IL. Skip this day entirely if Week 1 is already full.

**Day 5 (Sat) — hardware arrives:** Unbox, assemble, mount the follower arm at its fixed station (bolted down, cabling out of reach). Wire and test the physical e-stop. Calibrate both arms.

**Day 6 (Sun):** First teleoperation test. Mount camera(s) — wrist + static overhead/front view is the common setup. Record one throwaway episode to confirm the capture pipeline works. Test force/speed limits by hand.

**Milestone:** hardware mounted, e-stopped, and calibrated; capture pipeline confirmed.

---

## Week 2 (Aug 10–16) — Tier 1 fallback (insurance policy) + safety hardening
- Finish the physical barrier/enclosure and workspace joint limits.
- Teleoperate and record **one clean fixed motion**: reach into bin → grasp → move to handoff point → release. Wire this to a button/mat trigger, no ML involved. Test it end to end, repeatedly.
- This is your guaranteed demo. Halloween is never at risk once this works, regardless of what happens with IL below.

**Milestone:** Tier 1 fixed-motion fallback fully working and button-triggered.

---

## Week 3 (Aug 17–23) — IL data collection, pass 1
- Practice teleoperating smoothly — jerky demos make worse training data than expected.
- Collect **50–100 demonstration episodes** with deliberate variation: different candy positions, different shapes/sizes if mixed candy, slightly different bucket positions. Identical repeated demos won't generalize to a real kid's real bucket.

**Milestone:** a real LeRobot dataset with genuine variation.

---

## Week 4 (Aug 24–30) — Train the first IL policy
- Train ACT on a small subset first, to confirm the full data → train → deploy loop works before investing more collection time.
- Then train on the full dataset, evaluate on the real robot. Note *how* it fails, not just whether it fails (wrong grasp point? drops early? misses bucket by a consistent offset?).

**Milestone:** first trained ACT policy, with a concrete list of failure modes.

---

## Week 5 (Aug 31–Sep 6) — Data collection, pass 2 (targeted)
- Collect more episodes specifically covering what broke in Week 4 — this targeted pass matters more than raw volume.
- If ACT is struggling because your demos have multiple valid ways to do the task (e.g. grasping candy from different sides), try **Diffusion Policy** instead.
- Retrain, re-evaluate.

**Milestone:** measurable improvement over the Week 4 baseline.

---

## Week 6 (Sep 7–13) — Generalization
- Collect real data under varied lighting (including some evening/night sessions — porch/yard lighting at night is exactly the kind of shift that breaks vision policies trained in daytime).
- If you set up Isaac Sim in Week 1: generate a few domain-randomized (lighting/texture) episodes as extra variety.
- Retrain, test on conditions not in the original data.

**Milestone:** policy holds up across lighting conditions, not just your desk-lamp setup.

---

## Week 7 (Sep 14–20) — Build the visual trigger
- Simplest version first: a marker or reflective tape on the bucket. Get this reliable before considering anything fancier — it will outperform general detection, especially at night.
- Test standalone, log false positive/negative rate under your actual lighting.

**Milestone:** trigger detector with measured accuracy at deployment lighting.

---

## Week 8 (Sep 21–27) — Integrate trigger + policy
- State machine: `idle → detect trigger → run policy → dispense → cooldown/reset → idle`.
- Full dry run with a friend acting as a trick-or-treater. Watch where it breaks (timing, false triggers, occlusion).
- Time it — a real handoff needs to take a few seconds, not a leisurely minute, or you'll bottleneck a line of kids.

**Milestone:** end-to-end IL demo working in controlled conditions.

---

## Week 9 (Sep 28–Oct 4) — Robustness + real-location testing
- Test **at the actual deployment location, at night**, under real porch/yard lighting.
- Stress-test: different candy, different bucket angles/heights, kids in costumes waving hands. Collect + retrain wherever it breaks.

**Milestone:** policy + trigger survive worst-realistic-case testing on-site, at night.

---

## Week 10 (Oct 5–11) — Failure-mode planning + Halloween theming
- Explicitly test and decide the fallback for each: candy bin runs low/empty, bucket held at unexpected angle, kid grabs at the arm, camera partially blocked, power blip/restart. Decide now (stop-and-wait vs. revert to Tier 1) — not during the event.
- Build/decorate the enclosure without obstructing the camera's view.

**Milestone:** every failure mode has a decided, tested fallback.

---

## Week 11 (Oct 12–18) — Full integration + reliability
- Long-run testing: dozens of back-to-back cycles, log every failure with cause.
- Confirm graceful fallback to Tier 1 is instant and reliable (button press, no fumbling).

**Milestone:** N consecutive successful cycles at your target handoff speed (e.g. 20/20).

---

## Week 12 (Oct 19–25) — Buffer week
- This slot exists because it will get used — something above will take longer than planned. Spend it closing whatever gap is largest: more data, more integration testing, or just more reps.

---

## Week 13 (Oct 26–31) — Rehearsal & deployment
- Oct 26–29: full dry runs at the deployment location, at night, barrier in place, stand-in "kid" — repeatedly, not once the night before.
- Oct 30: final buffer day.
- **Oct 31 (Saturday) — Halloween night operations:**
  - Supervised mode the entire time, e-stop within reach at all times.
  - Tier 1 fixed-motion fallback ready to switch to instantly if the learned policy has an off night — no shame in this, it's the sane call.
  - Extra candy stock and a manual-handout plan in case hardware fails entirely. The goal is happy trick-or-treaters, not a perfect demo.

---

## Phase 2 (after Halloween) — VLA and RL

Once the dust settles and you actually know the platform (calibration, teleop, LeRobot's dataset/training tooling, where your current policy's weaknesses are), Phase 2 builds on the *same* dataset and hardware setup:

- **VLA:** fine-tune **Isaac GR00T N1.5** (has an SO-101 tuning guide) and/or **SmolVLA** (lighter, faster to iterate) on your existing IL dataset plus a language instruction per episode. Compare against your ACT baseline — this is where you'd expect better generalization to candy/positions the original data didn't cover.
- **RL:** two flavors worth learning — **Isaac Lab's** GPU-parallelized sim RL (a different, non-human-in-the-loop way to learn reward-shaping and large-scale domain randomization) and **HIL-SERL** on the real robot (starts from your existing demos, then you correct it live via the leader arm) to sharpen whatever's still imprecise — typically release timing or grip force.

No need to plan this in detail now — it'll be a much shorter runway once you're not also learning calibration and safety wiring for the first time.

---

## Key resources
- LeRobot docs: huggingface.co/docs/lerobot · SO-101 guide: huggingface.co/docs/lerobot/en/so101
- `TheRobotStudio/SO-ARM100` (GitHub) — hardware/assembly reference
- Phase 2 reference: `isaac-sim/Sim-to-Real-SO-101-Workshop` (GitHub), GR00T N1.5 SO-101 tuning (huggingface.co/blog/nvidia/gr00t-n1-5-so101-tuning), Isaac Lab docs (isaac-sim.github.io/IsaacLab)

## Notes
- Weeks 3–13 are weekly milestones, not 7 daily tasks each — at 1–2 hrs/evening that's roughly one sub-task every 1–2 days. Happy to expand any week into full daily detail as you get closer to it.
- If you're ever short on time, cut scope from the bottom of this list up (Week 12 buffer first, then theming polish). Never cut Week 1–2 safety work or the Tier 1 fallback — that's the one piece with zero acceptable failure mode.
