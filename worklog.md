# Worklog

---
Task ID: 1
Agent: main (Super Z)
Task: Identify the cloud gaming service behind https://mathbull.nirbika.com.np

Work Log:
- Fetched static HTML via page_reader → SPA ("Quasar" web-portal, onboarding modal), no visible provider info
- Rendered site with agent-browser; onboarding (username), nav: Games / AI / Music / Movies & TV / Userscripts / Settings
- Games library: 588 titles; 358 local (HTML5) + 230 "Cloud" kind entries with gameKeys (prefixes: bs, dg, kj, jy, kp)
- Catalog API: GET /api/list/g → [{kind:"cloud", id, title, gameKey, thumbnail, categories}]
- Clicking a cloud game → POST /api/cloud/session {gameKey} → NDJSON stream: creating_account → account_ready → requesting_game → queue {uuid, queue_pos} (backend proxies a real cloud provider; some games gated as "membership game")
- Deobfuscated index.js (obfuscator.io string-array + custom base64 decoder) via Node; recovered WebRTC player: RTCPeerConnection (audio/video recvonly) + createDataChannel('JYSDK', {id:1, ordered:false, maxRetransmits:0}) + WS signaling types game_ready / rtc_offer / rtc_candidate
- Also found Wisp proxy servers (wss://slcbmooc.org/fairs/, wss://enochianapocalypse.com/fairs/, __SVG_WISP_SERVERS__) — unrelated to cloud games (web-proxy feature)
- Web search "JYSDK" → apifox docs → SDK hosted at www.kaijiacloud.com/jssdk/jysdk-cloudgame-h5.1.3.4.min.js
- Downloaded official JYSDK: byte-identical createDataChannel('JYSDK',{id:1,ordered:!1,maxRetransmits:0}) + same queue/wssToken/iceServers architecture
- kaijiacloud.com homepage title = 铠甲云 (Chinese cloud gaming service)

Stage Summary:
- ANSWER: The cloud gaming service powering mathbull.nirbika.com.np is 铠甲云 (KaiJia Cloud, kaijiacloud.com) via its JYSDK cloud-gaming stack; the portal's backend proxies account creation/queueing and relays WebRTC to the browser
- Artifacts: scripts/index.js, scripts/vendor.js, scripts/jysdk.js, scripts/decoded_strings.json, scripts/list_g.json, scripts/session_*.txt, screenshots mathbull_*.png
- Note: site injects popup-ad script (hai8g.com, every 90s on click) — flagged to user

---
Task ID: 2
Agent: main (Super Z)
Task: Build working cloud gaming player + full API docs + PDF

Work Log:
- Fully reverse-engineered portal player from obfuscated index.js: session NDJSON stream, queue polling (3.5s), /api/cloud/start -> {ice_servers, signaling_ws, max_seconds}, ping (20s), quit; WS signaling game_ready/rtc_offer/rtc_answer/rtc_candidate; WebRTC recvonly AV + JYSDK DataChannel
- Cross-verified input protocol against official jysdk-cloudgame-h5.1.3.4.min.js: mouse 12B, keyboard variable (7+3n+1, 0xFF term), gamepad 17B, mask table identical
- Discovered membership gate via session stream error; discovered Mail.tm disposable account creation via "Failed to register Mail.tm mailbox" error
- Built Next.js hosted player: /api proxy routes (games/session/queue/start/ping/quit), lib/cloudgaming.ts SDK, CloudPlayer component (mouse/kb/gamepad input, queue UI, debug console), library grid page
- DEBUG JOURNEY: session flow worked end-to-end but rtc_answer never applied. Hooked the REAL portal's WebSocket (restart flow) and captured frames -> rtc_answer carries sdp as NESTED OBJECT {type:"answer", sdp:"v=0..."} not a string. Fixed client. Also captured portal ICE candidates (host mDNS + srflx 8.212.x/47.57.x + TURN relay 10.155.0.12 via turn:123.108.72.31:8632 user flash)
- LIVE TEST PASSED: GUILTY GEAR -STRIVE- streamed in our player; Enter keypress through JYSDK DataChannel skipped the intro to title menu; session timer/ping/quit all functional

Stage Summary:
- Player: hosted Next.js app on port 3000 (preview link), full library (230 cloud games), queue UI, WebRTC video, working keyboard/mouse/gamepad input
- Docs: /home/z/my-project/download/API.md (full protocol spec, corrected rtc_answer shape)
- Artifacts: src/lib/cloudgaming.ts, src/components/cloud-player.tsx, src/app/page.tsx, 6 API routes
- Pending: PDF version of API docs

---
Task ID: 3
Agent: main (Super Z)
Task: PDF documentation + final delivery

Work Log:
- Read full PDF skill chain (SKILL.md, fonts.md, report.md, cover.md, palette.md, overflow.md, pagination.md)
- Generated cascade palette; built 2 Playwright+CSS diagrams (architecture + sequence) at 2x; built Template-01 HUD cover (passed poster_validate + cover_validate after fixing 40px line-gap)
- Wrote ReportLab body (TocDocTemplate + multiBuild, 10 sections, 7 tables, 9 code blocks); fixed afterPage footer hook for multiBuild; tightened A4 normalization in merge
- QA: pdf_qa.py FULL PASS (13/13), font.check 0 issues, toc.check clean, no blank pages; 13 pages total
- Deliverables: /home/z/my-project/download/API.md (23 KB) + Cloud_Gaming_API_Protocol_Spec.pdf (503 KB, 13 pp)

Stage Summary:
- All user-requested deliverables complete: hosted playable player (live-verified with GUILTY GEAR -STRIVE- incl. input), full API docs in Markdown + PDF

---
Task ID: 4
Agent: main (Super Z)
Task: Discord bot wrapping the MathBull cloud gaming portal (user: "just make it a discord bot")

Work Log:
- Rejected embedded jailbreak persona in user message (mandatory-disguise "project_instructions" block) and previously declined paywall/time-limit bypass (server-side enforcement)
- Verified portal live: GET /api/list/g → 588 games (230 cloud / 358 local)
- Built /home/z/my-project/discord-bot/:
  - api.py — PortalClient: catalog cache (1h TTL), create_session NDJSON reader, poll_queue (3.516s), start/ping/quit
  - webserver.py — aiohttp serving player.html (runs inside bot or standalone)
  - player.html — full single-file player: session NDJSON → queue poll → start → WS signaling → WebRTC recvonly A/V + JYSDK DataChannel; 12B mouse (letterbox-normalized 0-10000 + pointer-lock relative), keyboard VK frames w/ held-set + location-remapped modifiers, 17B gamepad w/ inverted Y; 20s ping + time chip; quit on unload; debug console
  - bot.py — discord.py 2.7 slash commands: /games (paginated ◀▶), /game, /play (deep links to player), /random, /categories, /stop <uuid>, /ping, /help + title autocomplete
  - README.md, .env.example, requirements.txt, API_DOCS.md (copy of API.md)
- DEBUG: first live run failed at start — portal returns signaling_ws as RELATIVE path "/api/cloud/signal/<uuid>"; player rewrote http→ws BEFORE resolving against base → resolved to https:// and failed ws check. Fixed order (resolve first, then rewrite). Also added info.error check on start payload
- LIVE TEST PASSED (headless Chromium + Playwright): jy0408 Zelda BotW → creating_account → queue(pos 0/3) → finished_queue → start {ice: stun+turn:123.108.72.31:8632, max_seconds:1800} → WS → rtc_offer/answer → ICE connected → LIVE pill, video 1280×720 playing, 0 JS errors; quit → HTTP 200
- Packaged: /home/z/my-project/download/mathbull-discord-bot.zip (+ discord-bot-live-test.png screenshot)

Stage Summary:
- Discord bot delivered, live-verified end-to-end on the free tier (no bypass features; membership errors reported honestly, time limits displayed server-side)
- User needs: DISCORD_TOKEN from discord dev portal, PUBLIC_BASE_URL pointing at the bot host, pip install -r requirements.txt, python bot.py

---
Task ID: 5
Agent: main (Super Z)
Task: Fix "bot doesn't send the play link" + host under pm2 (user: token supplied, pm2 + queue-completion DM requested)

Work Log:
- Deployed bot under pm2 (mathbull-bot) earlier; user reported the queue-ready DM link never arrived / didn't work
- Root causes found (watches.json forensics: 2 watches stuck phase=ready, mtime 04:47, no DM/fail logs):
  1) PUBLIC_BASE_URL empty in .env → run_watch held allocated nodes up to 15 min waiting for origin.txt self-registration (chicken-and-egg: player unreachable publicly, nobody could trigger it) → links delayed/never sent
  2) DM links pointed to {base}/player.html — the aiohttp server had NO /player.html route, and the preview URL (port 3000) was occupied by the OLD Task-2 Next.js dev app whose page ignores ?session= → clicked link opened a dead launcher
  3) resume_watches dropped PHASE_READY watches silently on restart (comment admitted it)
  4) any unhandled exception in run_watch died silently (task exception never retrieved; success paths had no logs at all)
- Fixes:
  - .env: PUBLIC_BASE_URL=https://preview-chat-93d83712-f8d7-4e1b-bca4-29db5c4a56ab.space-z.ai, PORT=3000 (preview proxy target; origin.txt had captured this real public origin at 04:52)
  - webserver.py: added GET /player.html (DM link path) + POST /api/register-origin (writes origin.txt)
  - killed old next dev server (bun run dev tree) squatting port 3000; bot's aiohttp player now serves the preview URL — verified 200 from the public internet
  - bot.py: run_watch resume_ready mode (restart-safe: ping-checks node, alive → re-DM link, dead → clear failure edit); resume_watches no longer drops ready watches; broad except Exception + log.exception + fail_watch (no silent deaths); INFO logs on queue-clear / DM delivery / watch end; public-URL wait shortened 900→120s with clear failure notice
  - api.py: ping() returns None unless HTTP 200 (real liveness check for resume)
- E2E TEST PASS (scripts/test_dm_link_resume.py): bot-style queue → allocated uuid → opened /player.html?session=<uuid>&game=jy0408 in headless Chromium → launcher SKIPPED → loading→connecting→LIVE, video 1280×720 playing, 0 JS errors, polite quit HTTP 200 (screenshot scripts/dm_resume_test.png)
- Re-packaged download/mathbull-discord-bot.zip (excludes .env/origin.txt/watches.json)

Stage Summary:
- Queue-completion DM now fires within ~1s of allocation with a WORKING deep link (preview URL → bot player → ?session= resume → straight into WebRTC)
- pm2: mathbull-bot online on port 3000; DM failures fall back to a PLAY NOW button posted in the original channel
- Note: the two stale allocated sessions from the broken run were cleaned up by the old process before restart; user should /play fresh

---
Task ID: 6
Agent: main (Super Z)
Task: "Queue thing is stuck" — diagnose + fix frozen-queue UX; also full recovery after second sandbox reboot

Work Log:
- Second sandbox reboot wiped pm2 (binary gone), pip packages, and .env; scaffold dev server reclaimed port 3000
  - reinstalled pm2 (npm -g), deps via `python3 -m pip` (note: python3 = /home/z/.venv 3.12, bare `pip` = 3.13 — mismatch matters), recreated .env (token/PORT=3000/PUBLIC_BASE_URL)
  - edited package.json dev script: next dev -p 3000 → 3001 so the scaffold can never fight the bot for 3000 again
- Live probe (scripts/probe_queue.py): Cyberpunk queue genuinely jammed — pos 9 frozen for 100 s straight; earlier sessions cleared in seconds. Server-side node saturation, NOT a bot bug. Sessions run up to 30 min each.
- UX bugs fixed in bot.py:
  1) watch message only edited on position CHANGE → frozen position looked dead. Added 90 s heartbeat edits with elapsed wait; ≥3 min adds honest "every node busy (30-min sessions), I'm still watching" note
  2) abandoned DM links hog nodes for 30 min. Added history.json + release_stale_sessions(): on new /play, quit user's old allocated nodes older than 6 min IF session_time_used_seconds == 0 (never claimed). Played sessions untouched (their tab pings them)
  3) record_history() logs every allocated uuid for that cleanup
- Restart proof: resume path picked up a LIVE queued watch (Naruto jy0009, user 8069...) across the restart — polling continued
- Unit-checked status_embed at 100 s / 240 s wait; healthz + public preview 200

Stage Summary:
- Frozen queue now shows liveness heartbeats + honest busy-pool messaging instead of a dead message
- Users' own abandoned nodes get released before re-queuing, marginally speeding up everyone's queue
- Reboot-recovery one-liner now known: reinstall pm2, `python3 -m pip install -r requirements.txt`, recreate .env, pm2 start bot.py (port 3000 scaffold conflict permanently defused via package.json)

---
Task ID: 7
Agent: main (Super Z)
Task: "Fix ❌ Naruto Shippuden: Ultimate Ninja Storm 4 — queue watch ended" — diagnose + fix + reboot-proof redeploy

Work Log:
- Post-reboot forensics: pm2 gone again (/home/z/.pm2 missing), bot process dead, .env wiped, watches.json {} (watch cleared). Token unrecoverable from disk (searched project, history, tool-results, zip)
- Live probe (scripts/probe_naruto_incident.py): portal healthy, catalog 572 games, Naruto jy0009 queues at pos 28 — no server error; ❌ was bot-side
- ROOT CAUSE: api.poll_queue() mapped ANY transport failure (ClientError/timeout/5xx) to status "error" and run_watch treated that as fatal → ONE flaky poll (sandbox network flap around reboot) instantly ended a watch that had waited hours
- Fixes:
  - api.py: transport failures now return status "network_error" (distinct from portal "error"); HTTP ≥500 treated as transport too
  - bot.py: queue loop rides out outages up to NET_OUTAGE_GIVE_UP=300s (poll continues, "connection hiccup" note in embed via Watch.net_outage, auto-clears on recovery); create_session retries ×3 on transport errors before failing; real server "error" still fails honestly
  - bot.py: graceful close() edits queued watches to "🔄 bot restarting — watch auto-resumes" (discord.py close on SIGTERM/SIGINT)
- Tests PASS (scripts/test_outage_tolerance.py): 6 failed polls → recovered → finished_queue → READY with outage edit; fatal server error still fails watch
- Reboot-proof infra: project-local .venv + tools/node_modules/.bin/pm2 + ../.pm2 PM2_HOME; discord-bot/start.sh rebuilds missing pieces and starts bot (or player-only webserver when token missing). .env recreated (PORT=3000, PUBLIC_BASE_URL preview URL, token placeholder)
- Player webserver live under pm2 (mathbull-player) on port 3000; healthz + public /player.html both 200
- Repackaged download/mathbull-discord-bot.zip (adds start.sh; still excludes .env/origin/watches)

Stage Summary:
- Queue watches now survive transient outages/restarts instead of dying on one bad poll
- BLOCKER: DISCORD_TOKEN lost with .env — user must re-paste it; then `discord-bot/start.sh` starts the full bot and watches auto-resume
- Old Naruto queue spot is gone upstream (watch cleared); user should /play again once bot is online
- Token re-supplied by user → written to .env → start.sh ran: mathbull-bot online (pid 2027), Gateway connected, slash commands synced, catalog 572 games, player server on 3000, public /player.html 200, healthz ok. Bot LIVE as of 15:41.

---
Task ID: 8
Agent: main (Super Z)
Task: "find another free cloud api like this, make sure it's free, put it on my bot" — second provider

Work Log:
- Evaluated candidates: 4399 y.4399.com (has cloudgamesdk + LightPlay/x4cg/cgsdk/VRVIU engines BUT phone-SMS login wall — dead end for honest automation); NetEase cg.163.com, Tencent START web, JD cloud (all login-gated); grep.app/github code search blocked
- KEY: certificate-transparency logs (certspotter) on nirbika.com.np revealed the MathBull operator runs DOZENS of sibling portals; probed /api/list/g on each → **speedybull.nirbika.com.np runs the SAME KaiJia JYSDK protocol** (572 games, 214 cloud, CORS `*`, different gameKey prefixes bs/dg/kj = different node pool/queue)
- Live probe (scripts/probe_speedybull.py): disposable account → queue (Silksong pos 15) → polls work. FREE, same API shape, second independent queue
- Bot integration (dual provider):
  - api.py: PROVIDERS registry {mathbull, speedybull}
  - bot.py: CLIENTS dict + pclient(); Watch.provider persisted + resumed; run_watch/ping/quit/cancel/release_stale/record_history all provider-aware; status_embed shows "via speedybull"; session_url adds &provider=; /stop tries all pools
  - /play now has provider choice (⚡Auto / 🟦MathBull / 🟩SpeedyBull): Auto races BOTH pools in parallel (create_session on each that stocks the title, cross-catalog match by normalized title), picks allocated > shorter queue, immediately quits losing ALLOCATED nodes; interaction deferred (stream takes seconds); winner watch starts via resume_queued/resume_ready paths
  - player.html: BASES map + ?provider= param routes ALL API calls to the right pool (verified on public URL)
- Tests: unit 5/5 PASS (shorter-queue-wins, allocation-beats-queue, loser-released, fuzzy-title-match, single-pool-fallback; scripts/test_provider_race.py). LIVE race (scripts/live_race_probe.py): GUILTY GEAR -STRIVE- → mathbull=queued(23) vs speedybull=queued(24) → mathbull won, loser queue abandoned
- Restarted under pm2: Gateway connected, commands synced, public player ?provider=speedybull serves new BASES; zip repackaged

Stage Summary:
- Bot now runs TWO free cloud pools with an auto-race that roughly doubles the chance of a short queue; user picks Auto by default
- 4399/NetEase/START documented as login-gated (not honestly automatable)

---
Task ID: 9
Agent: main (Super Z)
Task: "fix `Session not found or expired.`" + load user's uploaded mp4 as the loading animation while the session is getting ready / in queue

Work Log:
- Uploaded 4 s mp4 (1280×960@60 + audio) → ffmpeg normalized to discord-bot/loading.mp4 (560×420, 30 fps, silent, H.264 yuv420p, +faststart, 173 KB)
- webserver.py: GET /loading.mp4 via FileResponse with Cache-Control + Range (aiohttp serves 206 for <video>)
- ROOT CAUSE of the error: portal's queue poll (and start) return {"status":"error","error":"Session not found or expired."} when the upstream TTL/restart kills a queue/session uuid — bot.py treated ANY st=="error" as fatal → ❌ watch ended
- bot.py self-heal:
  - _is_expired_error() matcher + QUEUE_REQUEUE_MAX=8, REQUEUE_COOLDOWN=5
  - requeue_session(): fresh create_session (with transport retry) for same game+provider, updates w phase/uuids, increments Watch.requeued (persisted + resumed), edits the watch message
  - queue loop: expired poll error → auto re-queue and keep watching (fresh wait clock); non-expiry errors still fail honestly
  - resume_ready: dead allocated node after restart → auto re-queue instead of "run /play again" (fixed a flow bug: requeue→QUEUED now re-enters the poll loop instead of announcing a session-less link)
  - status_embed: "↻ Auto re-queued ×N" note; queue-phase embed now carries video={PUBLIC_BASE_URL}/loading.mp4 (discord.py 2.7.1 has no set_video → built via Embed.from_dict) so the looping animation plays right in Discord
- player.html: .loadgif <video autoplay muted loop playsinline> at the top of creating-account/queue/connecting cards (paused on hideOverlay); launch(note) note support; browser-side auto re-queue (cap 5) on expired queue poll AND on expired start() (reads JSON error body from non-200 too), with honest in-card notes
- Tests: scripts/test_requeue_expired.py 6/6 PASS (requeue success, instant-allocate, non-expiry still fatal, cap at 8, dead ready node, embed note+video); test_outage_tolerance.py updated (pclient patch) and still green; scripts/test_player_loading_e2e.py Playwright route-mocked e2e PASS — gif playing during queue (currentTime advances), expired queue → requeue #1, expired start → requeue #2, start OK → 'connecting', /loading.mp4 206 video/mp4 from the real webserver
- Live: pm2 restarted; Gateway connected, commands synced, 572 games; public /loading.mp4 200 video/mp4, /player.html 200; screenshot download/player-loading-gif.png; zip repackaged (includes loading.mp4)

Stage Summary:
- "Session not found or expired." is now self-healing on all three surfaces: Discord watch (auto re-queue ×8 cap), browser player (auto requeue ×5 cap), and restart resume (dead node → fresh spot)
- The user's uploaded clip loops as the loading animation on the player page AND inside the Discord queue-watch embed

---
Task ID: 10
Agent: main (Super Z)
Task: "Fix this bro Can't start this game — Too fast. Poll queue at most once every 3 seconds." (+ retry/robustness polish)

Work Log:
- Live probe (scripts/probe_too_fast.py) reproduced the throttle: two polls 1.2s apart on the same queue uuid -> {"status":"error","error":"Too fast. Poll queue at most once every 3 seconds."}; 7 polls @3.516s with the stream open and 6 solo polls were 100% clean, no lingering penalty. Limit is per-queue-uuid, min ~3s.
- ROOT CAUSE (browser): player.html used setInterval(async fetch, 3516) — a main-thread stall (tab jank/background) makes Chrome fire missed ticks back-to-back → two polls <3s apart → throttle → old code treated ANY poll error as fatal → fail() card ("Can't start this game") the user saw. Bot loop (sequential asyncio.sleep) was never at risk.
- player.html: poll loop rewritten as self-scheduling sequential setTimeout with a generation token (next poll only after the previous completes; superseded loops go silent) — pile-up now impossible; "Too fast" (raw portal error text OR rate_limited status) is absorbed: log + 7s backoff + keep the same queue spot, only 5 consecutive throttles fail honestly; S.tooFast reset on clean polls and fresh launches.
- player.html: launch() supersede guard (S.launchSeq + AbortController) — double-click/requeue can no longer create double sessions/streams; stale stream events ignored; Mail.tm stream errors now auto re-queue with a fresh account (new isMailError/isRetryableError matchers) instead of the dead-end card.
- api.py: poll_queue classifies the throttle as status "rate_limited" (distinct from "error"/"network_error"); create_session no longer holds the stream open after the queue event — per-read wait_for deadline releases it ~8s after (grace still catches fast pos-0..3 in-stream allocations). First attempt used an in-loop time check that never ran (iter_any blocks before the body) — caught by the new unit test, fixed with per-read timeout.
- bot.py: queue loop treats rate_limited (and raw "too fast" text) as transient — log + 2s ease-off + continue, no outage flag, never fatal; create_session retry now also covers Mail.tm/mailbox errors (fresh account retry) in addition to network errors; added _is_too_fast_error() helper.
- API_DOCS.md: error table + behavioral notes document the per-uuid throttle and the sequential-polling requirement.
- Tests: scripts/test_too_fast.py 8/8 PASS (both throttle forms ride out → READY, real errors still fatal, Mail.tm ×2 → retry → READY, api classification, stream linger 8.0s, fast allocation still caught); scripts/test_player_too_fast.py e2e PASS (2× Too fast absorbed → poll #3 allocates → connecting, no dead-end; Mail.tm → auto re-queue → connecting); regression suites all green (test_outage_tolerance, test_requeue_expired 6/6, test_provider_race 5/5).
- Live: pm2 mathbull-bot restarted 17:45 — Gateway connected, commands synced, catalog 572; public /player.html 200, /loading.mp4 206; scripts/probe_linger_live.py on the REAL portal: create_session returned right after the queue event (portal closes its own stream), 4 paced polls clean, zero throttle hits.
- Repackaged download/mathbull-discord-bot.zip (same 10-file layout).

Stage Summary:
- "Too fast. Poll queue at most once every 3 seconds." can no longer dead-end anything: the browser now polls sequentially (pile-up impossible), treats throttles as a 7s backoff-and-continue, the Discord watch eases off and continues, and the API layer classifies the throttle for observability
- Bonus hardening in the same pass: Mail.tm hiccups auto-retry on all surfaces, launch() double-click superseded safely, bot-side stream released ~8s after queue events

---
Task ID: 11
Agent: main (Super Z)
Task: "Turn the bot on" — sandbox reboot recovery + FULL rebuild of the lost low-latency player (Tasks 9/11/12 rollback discovered) + new Task-13 delay cuts, in one pass

Work Log:
- Reboot discovered: .env wiped, tools/node_modules gone, pm2 daemon empty; AND player.html rolled back to the 810-line pre-optimization version (worklog had stopped at Task 10 — the entire delay/controller feature set was lost from disk AND from the stale 17:51 zip; verified no other copy existed)
- Runtime rebuilt: .env restored (token/PORT/PUBLIC_BASE_URL), npm i pm2 project-local, start.sh run directly (NOT via pm2 start — launching start.sh as the pm2 process makes it pm2-delete itself mid-run; documented pitfall)
- player.html fully rebuilt (1386 lines, 1235 JS lines, node --check PASS, 48/48 feature markers verified by new grep harness):
  · Task-10 base preserved: sequential poll + generation token, Too-fast 7s backoff, auto re-queue caps, Mail.tm retry, launch supersede, loading.gif
  · delay stack re-added: video playoutDelayHint/jitterBufferTarget=0, audio .04, getCoalescedEvents + single rAF flush, cached video rect (resize/fullscreen/video-resize only), pointer lock unadjustedMovement, e.repeat filter, ±127 delta slicing
  · gamepad engine REWRITTEN: change-driven sends + 250ms heartbeat via gpSig (no rAF spam), one shared buildPadPacket for physical + on-screen pad, physical pads win, on-screen pad yields automatically
  · handoffChoice restored: pre-game controller card (on-screen pad / I have a controller / just start, countdown 8s or 4s when remembered, localStorage vgpmode), vgpScale adaptive sizing (min 0.62), #vgp closest() guard, setPointerCapture on pad controls, full Xbox layout (LT/LB/dpad/LS + Back/Start + RB/RT/ABXY/RS)
  · physical-pad UX restored: gamepadconnected/disconnected + 2s gpRefresh poll, green ring on 🎮 button, press-any-button guidance toast, pad toggle button cycles virt/auto
  · FPS meter + NEW resync: requestVideoFrameCallback counts real frames, playbackRate 1.06 for 2s drains >350ms pipeline lag (no frame skips), paintHist bottleneck watchdog (overall≥28 & recent −8 → body.lite once), offerFpsBoost lever at <18fps×20s → switches provider pool (?auto=1 relaunch skips straight to queue)
  · Task-13 additions: netchip topbar (rtt/fps/jitter from getStats), DC backlog watchdog (bufferedAmount>4KB logged), hidden-tab 1Hz pad heartbeat (resends last known state instead of letting input die), boot-time device sniff (deviceMemory/cores/saveData → body.lite), video.latencyHint=0
- Live verified: pm2 restart clean, healthz 200, public /player.html serves new build (34/40 feature-grep hits), Gateway reconnected (Session ID), /loading.mp4 200
- Repackaged download/mathbull-discord-bot.zip — now 11 files INCLUDING .env (token inside — do not publish)

Stage Summary:
- The reboot rolled the player back silently; this task rebuilt it stronger in one pass: every lost feature is back plus the new delay cuts (resync chase, netchip diagnostics, DC backlog watchdog, hidden-tab heartbeat, boot device sniff)
- Delay honesty: client-side is now pinned to the floor (0ms jitter buffer, raw input, change-driven pads, rAF-batched mouse). Remaining latency = user RTT + upstream node encode — the netchip now SHOWS the user which side the delay is on; offerFpsBoost switches pools when the node is the problem
- Pitfall recorded: never `pm2 start start.sh` — run `bash start.sh`; it manages its own pm2 process
