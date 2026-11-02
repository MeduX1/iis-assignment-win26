# Review of `Assignment1/L01_Foundations_Tutorial.ipynb`

Reviewer note: this is a technical review of the submitted notebook (commit "Add Assignment 1 notebook"), based on reading all 29 cells and re-running the final code cell in a fresh interpreter. Numbers below were re-computed, not eyeballed.

## TL;DR

The conceptual understanding is good: the architecture mapping (Part 1 E5, Q1.1) is correct, and the audit-trail idea (Q5.2) is well argued. The problem is the **code and its demonstrations**: several outputs contradict the claims made next to them, and the final cell does not run on its own. Fixing the items under "Blocking" is about an hour of work and moves this from "plausible" to "verifiably correct".

## Blocking issues

### B1. Hard-coded verdicts contradict the computed results (cell 23, Step 4/5)
```
Warrant Checks  : {... 'safe': False, ... 'energetically_viable': False}
Gate Verdict    : APPROVED
```
The `print` for Attempt 2 is the literal string `"Gate Verdict    : APPROVED"`, not derived from `warrant_v2["passed"]`. The checks right above it show the packet is rejected (Force 7.03 N vs. limit 4.5 N, wave 3.43 mm vs. limit 1.5 mm). Consequences downstream:
- Step 6 (`if warrant_v2["passed"]`) never runs, so no execution happens.
- Attempt 2 is never recorded in the audit trail (only 1 entry exists).
- The audit analyzer therefore reports `Total Approved Executions: 0` and `Avg Force Sim-to-Real Variance: 0.000 N`, which looks like a measurement but is just an empty list.

Fix: print `'APPROVED' if warrant_v2['passed'] else 'REJECTED'` for both attempts, and record both attempts via `record_trace`. Then choose a second proposal that actually passes (see B3).

### B2. Task 4.2 demonstration never triggers the required redirect (cell 20/23)
The redirect is guarded by `failed_criterion == "energetically_viable"`, but `failed_criterion` returns only the **first** failing check in dict order, and `safe` comes before `energetically_viable`. The demo packet (v = 0.45 m/s) predicts 7.1 N, so `safe` fails first. The recorded output is `Failed On: safe`, and the "[SYSTEM ALERT] ... Requesting recharge action" branch never prints. The task asked to show that a failed battery check redirects the workflow; the submitted run does not show this.

Fix: return **all** failed criteria (`failed = [k for k, ok in checks.items() if not ok]`), and use a packet that is otherwise safe (e.g. v = 0.25 m/s) so that the battery check is the only failure. Also make the redirect an actual workflow branch (e.g. return a `RechargeAction`) instead of only printing.

### B3. The default `BeliefState` has `battery_level_pct = 12.0`, so every scenario fails the energy check
Because 12 % − drain < 15 %, `energetically_viable` is `False` in all runs that use the default state (visible in Step 4 and Step 5 output). Every "approve" path in the notebook is therefore unreachable. Default the battery to e.g. 80 % and set it to 12 % only in the Task 4.2 scenario.

Related: the criterion implemented is `(battery - drain) > 15`, i.e. "at least 15 % charge remains". The task text says "estimated drain stays below 15 % of total capacity". These are different conditions; state which one you implement and why, or implement the one in the task. The drain model `v * 10 + 2` is an unexplained magic number; document it as an assumption.

### B4. The final cell is not self-contained
Run alone in a fresh interpreter it fails:
```
NameError: name 'interpreter' is not defined   (line 207, Task 4.2 demo)
```
The demo block sits between class definitions and uses `interpreter`, which is only created later inside `if __name__ == "__main__":`. It works in the notebook only because of leftover kernel state from earlier cells. Run **Kernel → Restart & Run All** before submitting; the notebook should pass that.

## Part 2 correctness and rigor

### Task 2 (contact mechanics)
- `simulated_contact_depth_m` defaults to a constant `0.025`, which is always above the `0.02` threshold. The bottoming-out branch is therefore **always** active and the "dual-stage" behaviour is never exercised. Contact depth is not computed from the approach velocity, so the text "the simulated contact depth exceeds 0.02 m" in Q2.2 is an assumption, not a simulation result.
- The curve is discontinuous: force jumps by 80 % at exactly 0.02 m. Acceptable for the task as worded, but worth a sentence; a smooth transition is more physical.
- Q2.2 text says ≈ 4.868 N; the code prints 4.67 N. Both are explainable (noise-free analytic value vs. the −0.2…+0.3 N "epistemic" noise in the twin), but the discrepancy is not mentioned. The test is also unseeded and has no assertion.
- The test packet uses `tilt_rate_radps=0.1`, which predicts ≈ 4.2 mm of slosh vs. a 1.5 mm limit. The packet is therefore rejected on **two** criteria and the test does not show that the *force* check is what intercepted it. Use a tilt rate that is safe on its own, so only the force check can fail, and assert on that.
- Side effect worth noticing: with the 1.8× factor, any v above ≈ 0.33 m/s exceeds 4.5 N. The notebook's own "safe" fallback (0.45 m/s) predicts 7.1 N and is not safe.

### Task 3 (viscosity coupling)
- The linear 1 %/°C degradation is implemented as asked. The reference temperature of the base viscosity (1.2 Pa·s) is not stated; say that it applies at 20 °C.
- Q3.2 asks *how the temperature changes* the wave height but gives only the 28 °C value. Baseline for the same packet: 6.25 mm at 20 °C, 6.35 mm at 21.5 °C, 6.79 mm at 28 °C, i.e. about +7 %. State the delta.
- Because the limit is 1.5 mm, the gate rejects at **every** temperature. The safety violation is therefore not caused by the heat, and the experiment does not demonstrate the coupling. Show a configuration where the verdict actually flips with temperature, or say explicitly that it does not.
- The wave model `h = tilt_rate * r / viscosity` is not dimensionally a length (rad/s · m / Pa·s). Fine as a toy model, but label it as such rather than "fluid dynamics".
- Text says ≈ 6.79 mm, code printed 6.88 mm (noise again). Same remark as above.

### Task 1.2 (Proposer)
- The Gaussian is clipped with `max(0.05, ...)` / `max(0.01, ...)`, which makes it a censored distribution with a point mass at the clip value, not a Gaussian. For strictly positive, bounded quantities a truncated Gaussian or log-normal is easier to defend. "Most mathematically justified" overclaims; "simple and symmetric around a learned mean" would be accurate.
- The `Proposer` is never used. All demo packets are hand-written, so the resampling loop described in Q1.1 ("forces the Proposer to resample") does not exist in code. Either implement `propose → validate → resample (max N tries)` or soften the text.
- Quantitatively: the proposer mean of 0.5 m/s predicts about 8.3 N against a 4.5 N limit; roughly 87 % of samples would be rejected on force alone. Worth discussing in the justification.

### Task 5.1 (audit analyzer)
- Item 3 asks for the *variance* between predicted and observed force; the code computes the mean absolute error. Name it MAE or compute variance/standard deviation of the residuals.
- `max(set(reasons), key=reasons.count)` has a nondeterministic tie-break; use `collections.Counter(...).most_common(1)` and sort ties.
- With B1 fixed, run several episodes (seeded) so the statistics are non-trivial.

## Part 1 (theory)

- **E2** is cut off mid-sentence ("...will likely result in a spin") and never names the physical mechanism. Say what happens (slosh amplitude vs. freeboard, lateral acceleration `v²/r` in the turn) and ideally give an order-of-magnitude estimate.
- **E3** asks for the *single* item of evidence. You give the right answer (ask) but frame it as an absence of evidence. Name the one item that would license proceeding, e.g. "an explicit confirmation from Eva that the white mug is acceptable", and say briefly why asking beats observing here.
- **E6**: "pause and re-plan" is too vague for "state the exact evidence criterion". Give measurable conditions, e.g. fresh pose estimate of Eva (age < X ms, covariance < Y), handover pose reachable, predicted slosh below limit, then re-pass the gate. "pause" in "her new pause" should be "pose".
- E1, E4, E5 are fine. E4's T2 satisfies the "changed condition a video cannot answer" requirement.
- Typos: "coffe", "succeded", "mouvement", "Exacted", "t must ask".

## Part 3 (anatomy)

Correct but generic. The task says "based on the above picture", and the answer never refers to anything specific in `robot-body.jpg`. Name the actual links, joints, base type and sensors visible there. The sentence "the `energetically_viable` constraint **you** implemented" addresses the reader instead of the author; rewrite in your own voice.

## Repository / submission hygiene

- `coffee-task-photo.png` and `robot-body.jpg` are referenced but **not in the repo** (only the notebook is). Both images render as broken. The guidelines say to put all media inside `Assignment1/`.
- The ~14–17 kB class definition is pasted in six cells, so the notebook is ~160 KB and the versions drift apart. Define the classes once, then extend via subclassing or small patches in later cells.
- Outputs come from different kernel runs (different timestamps), which is how B4 stayed hidden. Re-run top to bottom and commit the fresh outputs.
- One empty trailing code cell; no README.
- **The submission requires inviting the lecturer as collaborator. The invitation went to the wrong Michael Beetz** (see the pull request description). Per the notebook, the deadline is **03.11.2026, 4 PM**, so please invite the correct account before then.

## What is already good

- Correct module-to-architecture mapping and a clean separation of belief, observation, goal, proposal, prediction, verdict and outcome.
- Q5.2 reasoning about the "asked / belief / committed / observed" separation is sound.
- Analytic checks in Q2.2 and Q3.2 are arithmetically correct (13.0625 × 1.8 × 0.35^1.5 = 4.868; 1.2 × 0.92 = 1.104; 0.15 × 0.05 / 1.104 = 6.79 mm).
