# SO-101 Candy Delivery Bot — Halloween Roadmap (Real-World)

## 0. What you're building

A stationary (or lightly mobile) SO-101 that hands out or places candy for trick-or-treaters. This is much closer to what the hardware and the LeRobot ecosystem were actually designed for than the crawler project: fixed-base pick-and-place with a gripper. Your main technical approach here is **imitation learning (IL)**, not RL — you demonstrate the task by teleoperating the arm, collect a dataset, and train a policy to reproduce it. That's faster to a working result and is the other core pillar of the LeRobot stack (SO-ARM's leader/follower teleop setup exists specifically to make this easy).

Given a Halloween deadline, this roadmap is organized so you have a *working fallback* at every stage — you're never more than a step away from something demoable, even if the fancy version isn't done yet.

---

## 1. Safety first — non-negotiable, do this before anything else

This runs near children. Decide and build these before writing a single line of policy code:

- **Hardware force/speed limiting:** cap servo torque and joint velocity in firmware/software to levels that can't cause injury even at full extension into an obstruction. Test this by literally putting your hand in the arm's path at max settings.
- **Physical workspace limiting:** mount and orient the arm, and set joint limits, so it is *physically impossible* for the end-effector to reach face/eye height of a kneeling child, regardless of what the policy outputs. This should be a hard mechanical/software constraint, not something you rely on the policy to respect.
- **Hard e-stop:** a physical, always-reachable kill switch (not just a keyboard Ctrl+C) that cuts power to the motors, not just sends a stop command.
- **Barrier/enclosure:** a simple physical guard (plexiglass, table edge, rope barrier) that keeps kids' hands out of the arm's motion envelope except at a defined candy pickup point.
- **Supervision:** you (or another adult) physically present and able to hit the e-stop the entire time it's running on Halloween night. This is a hard requirement, not a nice-to-have.
- **Fallback mode:** a simple non-autonomous mode (manual teleop, or a fixed pre-recorded motion triggered by a button) you can drop into instantly if the learned policy misbehaves.

Build the e-stop and workspace limits into the hardware/control stack now, so every later stage inherits them for free.

---

## 2. Task design — pick the simplest version that still feels magical

Don't design in "full autonomy" from day one. Rank these by difficulty and pick your v1 target, with the others as stretch goals:

1. **Easiest, still great:** kid presses a button/steps on a mat → arm runs a fixed pre-programmed motion to drop candy into an outstretched bucket/hand at a marked spot. No ML at all. This is your guaranteed fallback — build it first, in week one, so Halloween is never at risk.
2. **Middle:** arm picks a candy from a bin and places it into a bucket at a *roughly* consistent but not perfectly fixed position, using vision to find the bucket. Imitation-learned pick + vision-based placement.
3. **Ambitious:** arm detects an approaching hand/bucket with a camera, times the handoff, adapts grip based on candy shape/size variety in the bin. Full IL + perception pipeline.

Given the deadline, treat (1) as the floor you ship no matter what, and spend remaining time pushing toward (2) or (3).

---

## 3. Hardware setup

- Mount the follower arm securely at a fixed station — bolted down, correct orientation, cabling routed so a kid can't yank it.
- Set up the leader arm (or another teleop method) for demonstration collection — this is the standard LeRobot teleoperation workflow.
- Mount a camera (or two — wrist-mounted + a static overhead/front view is the common LeRobot setup) with a clear view of the candy bin and the handoff zone.
- Set up good, consistent lighting at the demo/deployment location — vision policies are sensitive to lighting changes between training and deployment, and porch/yard lighting at night is exactly the kind of shift that breaks this if you don't plan for it. Test under actual Halloween-night lighting conditions, not daytime.

---

## 4. Data collection (imitation learning)

- Use LeRobot's teleoperation + recording tools to collect demonstration episodes: reach into bin → grasp a candy → move to handoff point → release.
- Collect deliberate **variation** in your demonstrations: different candy positions in the bin, different candy shapes/sizes if you're using a mixed bag, slightly different bucket/hand positions at the handoff. A policy trained on identical repeated demos will not generalize to a real kid's real bucket in a real spot.
- Rough starting point: aim for on the order of 50–100 demonstration episodes for a task this simple before your first training run — you'll likely need more once you see where it fails, so budget time for a second collection pass.
- Label/organize the dataset the way LeRobot's dataset format expects (it has tooling for this — use it rather than rolling your own format).

## 5. Train the policy

- Use one of LeRobot's existing imitation learning recipes rather than building an architecture from scratch:
  - **ACT (Action Chunking Transformer)** — good default starting point for short manipulation tasks like this, known for working well with modest dataset sizes.
  - **Diffusion Policy** — more robust to multimodal demonstrations (e.g., multiple valid ways to grasp), but more expensive to train; consider if ACT struggles with the variation in your data.
- Train first on a small subset to make sure your whole pipeline (data → training → deployment back onto the real arm) works end to end before investing in a big data collection push. This is your version of the crawler project's "checkpoint" step — confirm the loop works before scaling effort.
- Validate in a *safe, kid-free* test setting extensively before ever running it near an actual child.

## 6. Perception (if going beyond the fixed-motion fallback)

- Start with the simplest detector that works: a fixed marker or colored bucket is far easier and more reliable than general hand/bucket detection, especially at night. Don't reach for a sophisticated vision model if a piece of reflective tape on the bucket solves the problem.
- If you do want general hand/bucket detection, a small pretrained object detector fine-tuned on your own captured images (bucket, hand, in your actual porch lighting) will be far more reliable than something generic — collect that data on-site if possible, in the evening.

## 7. Integration testing

- Full dry runs at the actual deployment location, at night, with a barrier/enclosure in place, and a stand-in "kid" (an adult, or a bucket on a stick) — repeatedly, over several days, not once the night before.
- Explicitly test failure modes: candy bin runs low/empty, bucket held at an unexpected angle/height, kid grabs at the arm, camera view partially blocked, power blip/restart. Decide the safe fallback behavior for each (e.g., stop and wait, revert to fixed-motion mode) before Halloween, not during it.
- Time it — a real trick-or-treat interaction needs to be fast (a few seconds), or you'll create a bottleneck line of impatient candy-seekers.

## 8. Halloween night operations

- Run in supervised mode the entire time, e-stop within arm's reach of you at all times.
- Have the fixed-motion fallback (Step 2, option 1) ready to switch to instantly if the learned policy has an off night — no shame in this, it's the sane engineering choice for something running unattended-ish near kids.
- Have a stock of extra candy and a plan for manual handout if hardware fails entirely — the actual goal is happy trick-or-treaters, not a perfect demo.

---

## Suggested pacing against a ~14-week runway to Halloween

- **Weeks 1–2:** safety systems (e-stop, workspace limits, mounting) + the guaranteed fixed-motion fallback (Step 2 option 1) working end to end. This is your insurance policy — don't skip or delay it.
- **Weeks 3–5:** teleop setup, first data collection pass, first ACT training run on a subset, confirm the full pipeline works.
- **Weeks 6–9:** scale up data collection with real variation, retrain, start integration testing at the real location.
- **Weeks 10–12:** perception layer if pursuing option 2/3, more integration testing under realistic (night) conditions, failure-mode handling.
- **Weeks 13–14:** buffer for things going wrong (they will), full dry runs, final fallback confirmation.

---

## Key resources
- LeRobot (Hugging Face) — teleoperation tools, dataset format/recording tools, ACT and Diffusion Policy training recipes, all built around SO-ARM as a reference platform.
- `TheRobotStudio/SO-ARM100` (GitHub) — hardware/assembly reference, applies to your SO-101 hardware too.
- LeRobot's example notebooks/scripts for data collection and policy training are your fastest path to a working pipeline — adapt rather than rebuild.
