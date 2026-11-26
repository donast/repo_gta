Title: View-heap overflow and CPU-bound frame time in GTA6 runs; `SPECULATIVE_COPIES` obsolete upstream, `VICTORY_PORT_*` patches are the real levers

Author / collaborators: dona (investigation) — AI-assisted (Claude Code). Runtime logs provided by Start0-0 (tester, DexNom Runtime build). Findings pending verification.

Document attachment or link + revision/date: Revision 1 — 2026-11-26. Local logs in `C:\Users\dona\Desktop\VIctory\`: `run-1795717757360-10792.log`, `run-1795718392207-12468.log`, `run-1795719053819-22780.log`, `warm-1795717207664-25980.log`. Repo studied: `Belobug/VIctory-Belobug-AMD` @ `a-publier`, cloned 2026-11-26 (pushed 2026-11-26T17:52Z).

Area: performance (primary) / memory (secondary)

Question or problem investigated: How to reduce frame time and VRAM in GTA6 under VIctory, and whether an earlier recommendation to lower `SPECULATIVE_COPIES` is still valid against current upstream.

Repository + exact commit (if applicable): `Belobug/VIctory-Belobug-AMD`, branch `a-publier` (default; `rx9070xt` is identical, `main` is 550 commits behind). Compared against local outdated copy `VIctory-main/VIctory-main` and `CRAZECRAZE/VIctory` @ `main`. The logs themselves were produced by the "DexNom Runtime" build (`%LOCALAPPDATA%\DexNom Runtime\victory.exe`), commit unknown — recorded here as unknown.

Short summary / key findings:
- The 4 logs show a CPU-bound title: GPU 47–87% busy, frame times 100–941 ms, submit thread ~40 ms (`the card's clock per frame` lines). Not a GPU/VRAM bottleneck.
- View-heap overflow confirmed in 3 of 4 logs: `host view heap capped at 1000000; views past it read a null slot count=1245184` — 245,184 views read the null slot (visual corruption).
- VRAM pressure in 2 of 4 logs: `a texture already on the card is brought up to date although the card has no room for new ones`; also `VRAM texture budget reached; new allocations wait`.
- The earlier `SPECULATIVE_COPIES` recommendation is obsolete: upstream replaced it with `COPIES = (1<<12)-1` + an arena/sweep system (`memory.rs` 861→1304 lines). Local copy is ~589 commits behind.
- Real levers already exist upstream, off by default: `VICTORY_PORT_CBV` (-4.7 ms), `VICTORY_PORT_GUARD` (-2 ms), `VICTORY_PORT_JOURNAL` (-1 to -3 ms), `VICTORY_PORT_DESCRIPTORS`, `VICTORY_DYNAMIC_SLOTS` (view-heap fix), `VICTORY_CARD_ROOM_FOR_TARGETS_MB` (VRAM).
- Project is AMD-first by design; NVIDIA (0x10de) has no dedicated code path (`indirect.rs:285` only checks vendor 0x1002). NVIDIA is documented as untested upstream.

Sources / relevant files and functions:
- `crates/victory-gpu/src/memory.rs:80` — `const COPIES: u32 = (1<<12)-1;` (replaces `SPECULATIVE_COPIES`)
- `crates/victory-gpu/src/descriptor/host.rs:1522` — `PORTABLE_VIEWS = 1_000_000`; `:1528` `UNCAPPED_VIEWS = 1<<21`; `dynamic_slots()` + `VICTORY_DYNAMIC_SLOTS` (~line 1610)
- `crates/victory-gpu/src/indirect.rs:285` — `amd()` vendor check (0x1002 only)
- `README.md` (Belobug) lines 30–67 — perf table, `VICTORY_PORT_*` patches, known issues
- Logs above — `victory_gpu::descriptor::host`, `victory_gpu::host` (`the card's clock per frame`, `budget of the card`), `victory_gpu::draw`

Method / reproduction steps: Static diff of local vs cloned `a-publier`; grep of env-var surface (`VICTORY_*`) across both; extraction of `card's clock`, `view heap`, `budget of the card`, `no room`, and header spec lines from the 4 logs. No runtime test was performed by the author.

Hardware / OS / driver / settings (if measured): From log headers — console profile Xbox Series X, 3840x2160, ray tracing on, sparse texture mips + dirty-page readback + RT write stats requested, `VICTORY_LATE_WAIT=1`, `VICTORY_RELAY=1`. `host wave lanes min=32 max=32` (RDNA3-class AMD wave32; not RDNA3 wave64). VRAM budget fluctuated ~8.1–9.3 GB dedicated, ~15 GB shared. `VICTORY_PORT_*` NOT enabled (old build). CPU model, OS build, GPU model and driver: unknown (not printed in logs). Belobug README reference box: i9-12900K, 64 GB, RX 7900 XTX, RT off.

Evidence: Hypothesis (CPU-bound diagnosis and patch impact are inferred from logs + README, not re-run locally). Reproduced (view-heap cap message `count=1245184` and VRAM-pressure messages are present verbatim in the logs). Code-version drift is independently verified by file diff against the cloned repo.

Verifier + supporting result (if verified): Pending — needs the tester to re-run with `a-publier` + `VICTORY_PORT_*` and report median/p95 frame time, dedicated+shared VRAM, dropped draws.

Limitations / open questions: Logs came from an outdated DexNom build, so patch impact is unquantified for GTA6 specifically. `PORTABLE_VIEWS` cap still 1,000,000 upstream; whether the uncapped `1<<21` heap is actually selected on the tester's GPU is unknown. CPU/GPU/OS/driver of the log source are unknown, blocking independent reproduction. Some materials render black (upstream known issue). Several pipeline creation methods unimplemented (slots 0x218, 0x1c8), forcing fallbacks.

Recommended next experiment or action: Re-run the 4 scenarios on `a-publier` with `VICTORY_PORT_CBV=1 VICTORY_PORT_GUARD=1 VICTORY_PORT_JOURNAL=1 VICTORY_PORT_DESCRIPTORS=1` and `VICTORY_DYNAMIC_SLOTS` set; capture the same log lines. Compare frame time and confirm the `capped at 1000000` warning is gone.

Issue / PR links: None yet. Candidate upstream area: `VICTORY_PORT_*` validation on GTA6; `VICTORY_WRITE_MARKS` (CRAZECRAZE `raw-write-marks`, untested on GPU).

Review or help requested: A Lead Developer to confirm which `VICTORY_PORT_*` combination is safe to default-enable, and a tester with the game + known hardware to produce verification logs.