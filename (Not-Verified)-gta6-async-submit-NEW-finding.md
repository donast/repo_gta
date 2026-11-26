Title: NEW finding — `VICTORY_ASYNC_SUBMIT` can move the ~40 ms submit work off the title's thread onto a dedicated fast-core thread (CPU-bound fix candidate, untested)

Author / collaborators: dona (investigation) — AI-assisted (Claude Code). Runtime logs provided by Start0-0 (tester, DexNom Runtime build). Finding pending verification.

Document attachment or link + revision/date: Revision 1 — 2026-11-26. Companion to `gta6-viewheap-cpu-findings.md` (Revision 1, same date). Repo studied: `Belobug/VIctory-Belobug-AMD` @ `a-publier`, commit `cd8dfc06ace7ccb91a19431e74bc2d0de2a10145` (2026-11-26, "PC FidelityFX FSR 4.1.2 to 4K (-Fsr4K)"). Logs analyzed: `run-1795717757360-10792.log`, `run-1795718392207-12468.log`, `run-1795719053819-22780.log`, `warm-1795717207664-25980.log` (in `C:\Users\dona\Desktop\VIctory\`).

Area: performance (CPU-bound frame time / submit-thread contention)

Question or problem investigated: The 4 GTA6 logs show a CPU-bound title (GPU 47–87% busy, frame times 100–941 ms, submit thread ~40 ms). Is there an existing, off-by-default switch that moves submission work off the title's thread, and would it plausibly reduce frame time for this title?

Repository + exact commit (if applicable): `Belobug/VIctory-Belobug-AMD`, branch `a-publier`, commit `cd8dfc06ace7ccb91a19431e74bc2d0de2a10145`. Logs were produced by the "DexNom Runtime" build (`%LOCALAPPDATA%\DexNom Runtime\victory.exe`), commit unknown — recorded as unknown.

Short summary / key findings:
- `VICTORY_ASYNC_SUBMIT=1` makes the runtime spawn a dedicated thread named `victory-submit`, pin it to fast cores, raise its priority (`THREAD_PRIORITY_ABOVE_NORMAL`), and prepare/send command submissions on it instead of on the title's submitting thread.
- This directly targets the observed bottleneck: the logs' ~40 ms submit-thread cost would move off the title's critical thread.
- The flag has two preconditions that the tester's runs ALREADY satisfied except for the flag itself: (1) `started()` requires `late_free_present()` = `late_stage() > 2`, reached with `VICTORY_LATE_WAIT=1` + `VICTORY_RELAY=1` — both were set in the logs; (2) `cuts()` is on by default unless `VICTORY_CUT_WAITS=0`.
- The tester did NOT set `VICTORY_ASYNC_SUBMIT=1`, so the dedicated thread never started. Adding one flag to the same config is a cheap, natural next experiment.
- README mentions the flag only in passing (no frame-time numbers), so real GTA6 impact is genuinely unknown. This is a Hypothesis, not Reproduced.

Sources / relevant files and functions:
- `crates/victory-gpu/src/queue/behind.rs:737` — `on()` gate: `VICTORY_ASYNC_SUBMIT==1 || thin::wanted()`, AND `started()`.
- `crates/victory-gpu/src/queue/behind.rs:758-793` — `started()`: checks `late_free_present()` + `cuts()`; spawns `victory-submit`, sets priority + fast cores; logs `submissions are prepared and sent by a thread of ours ... (VICTORY_ASYNC_SUBMIT=1)`.
- `crates/victory-gpu/src/queue/behind.rs:711-722` — `VICTORY_SUBMIT_DEPTH` (default 64 ≈ two frames) and `VICTORY_SUBMIT_FRAMES` (default 1), related queue-depth knobs.
- `crates/victory-gpu/src/host.rs:2591` — `late_free_present()` = `late_stage() > 2`; `:2536-2543` `late_stage()` / `VICTORY_LATE_WAIT`; `:2595` relay (`VICTORY_RELAY=1`).
- `crates/victory-gpu/src/queue.rs:111` — `cuts()` (on by default; `VICTORY_CUT_WAITS=0` disables).
- `crates/victory-gpu/src/list/thin.rs:56` — `thin::wanted()` (`VICTORY_THIN=1`), an alternate way to trigger async submit.
- `crates/victory-gpu/src/cores.rs:233` — `keep_on_fast_cores()`.
- `README.md:127-130` — passing mention of `VICTORY_ASYNC_SUBMIT`, `VICTORY_RELAY`, `VICTORY_LAND` (no numbers).
- Logs above — submit-thread / `the card's clock per frame` lines.

Method / reproduction steps: Static read of the async-submit gate and its preconditions in the cloned `a-publier` tree; cross-checked the tester's logged env (`VICTORY_LATE_WAIT=1`, `VICTORY_RELAY=1`, no `VICTORY_ASYNC_SUBMIT`) against `started()`'s requirements; confirmed `cuts()` default. No runtime test was performed by the author.

Hardware / OS / driver / settings (if measured): From log headers — console profile Xbox Series X, 3840x2160, ray tracing on, `VICTORY_LATE_WAIT=1`, `VICTORY_RELAY=1`, `host wave lanes min=32 max=32` (RDNA3-class AMD wave32). `VICTORY_ASYNC_SUBMIT` NOT set. CPU model, OS build, GPU model, driver: unknown (not printed in logs).

Evidence: Hypothesis (the frame-time benefit is inferred from code + logs, not re-run locally). Independently verified (the gate logic, preconditions and the tester's missing flag were confirmed by direct file diff/read against commit `cd8dfc0`). The CPU-bound symptom itself (submit thread ~40 ms) is Reproduced — present verbatim in the logs.

Verifier + supporting result (if verified): Pending — needs the tester to re-run the same GTA6 save/scene with `VICTORY_ASYNC_SUBMIT=1` added (keeping `VICTORY_RELAY=1 VICTORY_LATE_WAIT=1`) and report submit-thread time and median/p95 frame time.

Limitations / open questions: Logs came from an outdated DexNom build, so the submit cost is unquantified for `a-publier`. Whether async submit actually helps depends on how much of the 40 ms is genuinely offloadable vs. serial with the present; the code warns it needs lists cut at their waits, which is on by default but could interact with other flags. `thin::wanted()` is an alternate trigger whose interaction is unstudied here. CPU model unknown, so "fast cores" pinning effect is unknown. No evidence yet that moving submit off-thread reduces total frame time rather than just relocating the cost.

Recommended next experiment or action: Re-run the same GTA6 save and camera scene twice on `a-publier` — control (current config) vs. `VICTORY_ASYNC_SUBMIT=1` added — holding `VICTORY_RELAY=1 VICTORY_LATE_WAIT=1`. Capture the `submissions are prepared and sent by a thread of ours, per frame` line and compare submit-thread time and median/p95 frame time. If it helps, test stacking with `VICTORY_PORT_CBV/GUARD/JOURNAL` from the companion doc.

Issue / PR links: None yet. Candidate upstream area: `VICTORY_ASYNC_SUBMIT` validation on a CPU-bound title.

Review or help requested: A Lead Developer to confirm whether `VICTORY_ASYNC_SUBMIT` is safe/expected to combine with `VICTORY_RELAY=1 VICTORY_LATE_WAIT=1` on a title like this, and a tester with the game + known hardware to produce before/after logs.
