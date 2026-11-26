Title: NEW finding — VRAM-pressure knobs `VICTORY_CARD_ROOM_FOR_TARGETS_MB` and `VICTORY_TEXTURE_ROOM_PERCENT` are unset in the GTA6 logs; the "no room for new ones" / "texture budget reached" stalls are the symptom they target (untested)

Author / collaborators: dona (investigation) — AI-assisted (Claude Code). Runtime logs provided by Start0-0 (tester, DexNom Runtime build). Finding pending verification.

Document attachment or link + revision/date: Revision 1 — 2026-11-26. Companion to `gta6-viewheap-cpu-findings.md` (Rev 1) and `gta6-async-submit-NEW-finding.md` (Rev 1), same date. Repo studied: `Belobug/VIctory-Belobug-AMD` @ `a-publier`, commit `cd8dfc06ace7ccb91a19431e74bc2d0de2a10145` (2026-11-26). Logs analyzed: `run-1795717757360-10792.log`, `run-1795718392207-12468.log`, `run-1795719053819-22780.log`, `warm-1795717207664-25980.log` (in `C:\Users\dona\Desktop\VIctory\`).

Area: memory (VRAM pressure / texture residency) — secondary performance

Question or problem investigated: The GTA6 logs show VRAM pressure — `VRAM texture budget reached; new allocations wait` and `a texture already on the card is brought up to date although the card has no room for new ones`. Are there existing, off/default switches that reserve card memory for render targets and relax the texture-copy budget, and were they active in the tester's runs?

Repository + exact commit (if applicable): `Belobug/VIctory-Belobug-AMD`, branch `a-publier`, commit `cd8dfc06ace7ccb91a19431e74bc2d0de2a10145`. Logs produced by the "DexNom Runtime" build (`%LOCALAPPDATA%\DexNom Runtime\victory.exe`), commit unknown — recorded as unknown.

Short summary / key findings:
- `targets_room_kept()` (`resource.rs:1822`) reserves card memory for render targets / host copies the host writes itself. Default `VICTORY_CARD_ROOM_FOR_TARGETS_MB = 3072` MB (clamped to ≤8192; `0` = old behavior of reserving none). It is ON by default at 3072.
- `room_percent()` (`resource.rs:1862`) caps the share of the card budget used for index-picked texture copies: default `VICTORY_TEXTURE_ROOM_PERCENT = 90` (clamped 50–97). At 90, ~1,020 MB of a 10,200 MB budget is left idle while thousands of textures wait (code cites runs 170–174); raising to ~94 gives textures about 449 MB back.
- Code comments (runs 193/197/198) describe the exact failure the logs show: card at 99–100.5% of budget, buffer copies permanently refused (31 of 95 made — the 355 ms frame where 95 give 238 ms), texture queue refused from the first world frame. Reserving targets' room makes the queue stop earlier during world load and reclaim it as targets are made.
- In the tester's log the card budget is ~8,179–9,283 MB dedicated (~15,163 MB shared). At `18:30:12` the budget-reached WARN and the "no room for new ones" INFO fire back-to-back, with in-use (9,270 MB) over budget (8,652 MB) — i.e. the card is oversubscribed and leaning on shared memory. Neither room flag appears set in the logs (no `ROOM_PERCENT` / `CARD_ROOM` lines).
- Because both flags default to non-zero values, the tester's old DexNom build may predate them or override them; whether they are actually in effect is unknown. This is a Hypothesis.

Sources / relevant files and functions:
- `crates/victory-gpu/src/resource.rs:1822-1835` — `targets_room_kept()` / `VICTORY_CARD_ROOM_FOR_TARGETS_MB` (default 3072 MB, cap 8192).
- `crates/victory-gpu/src/resource.rs:1862-1871` — `room_percent()` / `VICTORY_TEXTURE_ROOM_PERCENT` (default 90, clamp 50–97).
- `crates/victory-gpu/src/resource.rs:1837-1853` — `room_lately()` (10 Hz-cached room check).
- `crates/victory-gpu/src/shadow.rs:269-298` — `VICTORY_CARD_ROOM_FOR_BUFFERS` (related; comment: run 193 on still made 31 of 95 copies).
- `crates/victory-gpu/src/draw.rs` — emitter of `VRAM texture budget reached; new allocations wait` and `...no room for new ones`.
- Logs above — `victory_gpu::host` budget lines (9283→8179→8433→8714 MB) and `victory_gpu::draw` pressure lines at 18:30:12.

Method / reproduction steps: Static read of the room/budget functions and their env parsing in `a-publier`; extraction of every `budget of the card` / `no room` / `texture budget` line from the 4 logs; grep confirming no `VICTORY_CARD_ROOM_FOR_TARGETS_MB` / `VICTORY_TEXTURE_ROOM_PERCENT` setting appears in the logs. No runtime test was performed by the author.

Hardware / OS / driver / settings (if measured): From log headers — console profile Xbox Series X, 3840x2160, ray tracing on, `VICTORY_LATE_WAIT=1`, `VICTORY_RELAY=1`, `host wave lanes min=32 max=32` (RDNA3-class AMD wave32). Card budget observed 8,179–9,283 MB dedicated, 15,163 MB shared; in-use peaked at 9,270 MB (over the 8,652 MB budget at that instant). `VICTORY_CARD_ROOM_FOR_TARGETS_MB` and `VICTORY_TEXTURE_ROOM_PERCENT` not shown as set. CPU/GPU model, OS build, driver: unknown (not printed).

Evidence: Hypothesis (the frame-time/VRAM benefit of tuning these is inferred from code comments + logs, not re-run). Reproduced (the `VRAM texture budget reached` and `no room for new ones` lines, and the over-budget in-use figures, are present verbatim in the logs). Code defaults and parsing independently verified by direct read of commit `cd8dfc0`.

Verifier + supporting result (if verified): Pending — needs the tester to re-run the same GTA6 save/scene on `a-publier` with explicit `VICTORY_CARD_ROOM_FOR_TARGETS_MB` and `VICTORY_TEXTURE_ROOM_PERCENT` values and report dedicated+shared VRAM, the frequency of the two pressure lines, and median/p95 frame time.

Limitations / open questions: Logs came from an outdated DexNom build, so it is unknown whether these flags already exist there or what their effective values were. The card is genuinely small for this title (8–9 GB dedicated), so reserving 3 GB for targets could starve textures further on this specific GPU — the right target-room value is hardware-dependent and untested here. Raising `ROOM_PERCENT` trades idle headroom for texture residency; the sweet spot (code suggests ~94) is from runs on a different box. Interaction with the shared-memory fallback (15 GB) is not modeled. No evidence yet that either change reduces frame time rather than shifting the stall.

Recommended next experiment or action: On `a-publier`, run the same save/scene as control, then with `VICTORY_TEXTURE_ROOM_PERCENT=94` (give textures ~449 MB back), and separately with an explicit `VICTORY_CARD_ROOM_FOR_TARGETS_MB` (e.g. 2048) if the card is oversubscribing. Capture the two `victory_gpu::draw` pressure lines and the `budget of the card` lines; compare their frequency and frame time. Combine with the async-submit / `VICTORY_PORT_*` experiments only after each is measured alone.

Issue / PR links: None yet. Candidate upstream area: validating `VICTORY_CARD_ROOM_FOR_TARGETS_MB` / `VICTORY_TEXTURE_ROOM_PERCENT` on a small-VRAM (8–9 GB) title.

Review or help requested: A Lead Developer to confirm whether these flags exist and are effective in the DexNom build the logs came from, and the recommended target-room value for an ~8 GB card; a tester with the game + known hardware to produce before/after logs.