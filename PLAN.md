# TexRig road stream 0.1 — build plan

**Status: awaiting approval. No code has been written.**

How to read this:

- Section 1 is what I checked before designing. Several choices below hinge on these facts, so I verified them against the source of the exact versions this build pins.
- Section 2 is every place where I propose to deviate from the letter of the brief, each with the reason. These become the first entries of `DECISIONS.md`.
- Section 3 is the file list — every file, what it does.
- Section 4 is the build and test order.
- Section 5 is the short list of things I need from you.

If you approve as-is, I start with `capture/`.

---

## 1. Facts I verified first

Sources: FFmpeg `release/7.1`, MediaMTX `v1.21.1`, gortmplib `v1.0.3`, rsync `master` man page and source. Dated 2026-09-26.

1. **The concat demuxer cannot follow a playlist that grows.** It reads the whole list when the input is opened (`libavformat/concatdec.c`, `concat_parse_script` loops until EOF) and never re-reads it. Appending lines after FFmpeg has started has no effect. "One long-running FFmpeg reading a playlist the Python process appends to" is not something the concat demuxer can do.

2. **`-re` bursts after a stall.** `-re` compares each packet's timestamp with the wall clock since the input was opened (`fftools/ffmpeg_demux.c`, `readrate_sleep`). If the input stalls for H seconds — a dropout — FFmpeg then reads at full speed until the stream timeline catches up. H seconds of road go out in a couple of seconds, MediaMTX's write queue overflows, every platform gets disconnected, and the delay you wanted to grow shrinks back instead. FFmpeg 7.1 has no option to cap that catch-up. Conclusion: FFmpeg must not be the thing that paces a stream that is allowed to stall.

3. **FFmpeg stitches timestamp resets on its own.** The MPEG-TS demuxer carries the `AVFMT_TS_DISCONT` flag; when a packet's timestamp jumps backward by more than 0.1 s, the CLI applies an offset so the output stays continuous (`ffmpeg_demux.c`, `ts_discontinuity_detect`). This is what lets segments written with `-reset_timestamps 1` be fed into one publisher and come out as one continuous stream.

4. **Segment filenames: strftime or counter, not both.** The segment muxer expands the output name with *either* `strftime` *or* the `%d` counter (`libavformat/segment.c`, `set_segment_filename`). `seg-%Y%m%d-%H%M%S-%06d.ts` cannot be produced by FFmpeg directly. Something has to rename.

5. **MPEG-TS clock cadence.** In VBR mode FFmpeg's MPEG-TS muxer writes a clock reference (PCR) roughly every 100 ms — "the highest multiple of frame duration which is less than 100 ms" (`libavformat/mpegtsenc.c`). That is fine-grained enough to pace playback from Python by reading the segment's own clock.

6. **MediaMTX v1.21.1 forwards natively.** A path can carry `forward:` with any number of `dest:` entries, `rtmp://` and `rtmps://` included. Each destination runs on its own with a 5-second retry on error (`internal/forward/dest_handler.go`); one platform failing does not touch the others. The control API reports each destination (`/v3/paths/forward-dests/list?path=road`): `idle` / `forwarding` / `error`, `lastError`, bytes sent. No FFmpeg is needed for fan-out, and nothing is transcoded.

7. **MediaMTX keeps stream keys out of its log.** Its destination format is `rtmps://host/app#STREAMKEY` — the key rides as the URL fragment — and the log line "forwarding to '…'" strips the fragment (`sanitizeDestURL`). Error strings from its RTMP client (gortmplib `client.go`: "bad result", "authentication error", dial errors) do not contain the URL.

8. **MediaMTX drops a silent publisher after `readTimeout` (default 10 s).** `internal/servers/rtmp/conn.go` sets a read deadline per message. During a hold the publisher sends nothing, so `readTimeout` must be raised or every hold longer than 10 s tears the whole stream down.

9. **rsync.** `--append-verify` implies `--inplace`, and `--inplace` conflicts with `--partial-dir` (man page). So a half-transferred segment sits on the relay *under its final name*. Something other than the name has to say "this file is complete." Also verified: `--remove-source-files` removes a source file that is already identical on the receiver (`generator.c`: `quick_check_ok` → `send_msg_success`), and `rrsync` — the restricted-rsync shell in the rsync package — allows it unless started with `-ro` or `-no-del`.

10. **Versions available today.** Debian 13 "trixie" is stable: `ffmpeg 7:7.1.5-0+deb13u1`, `rsync 3.4.1`, `python3 3.13.5`. Homebrew: `ffmpeg 9.0.2`, `rsync 3.5.1`, `hidapi 0.15.0`, `python@3.13 3.13.15`. PyPI: `streamdeck 0.10.0` (needs the hidapi shared library), `Pillow 12.3.0`. MediaMTX latest release: `v1.21.1`, assets `mediamtx_v1.21.1_linux_amd64.tar.gz` and `checksums.sha256`.

11. **Apple's hardware H.264 encoder honours forced keyframes** (`libavcodec/videotoolboxenc.c`: an I-frame request becomes `kVTEncodeFrameOptionKey_ForceKeyFrame`) and honours `-maxrate`/`-bufsize` (`kVTCompressionPropertyKey_DataRateLimits`).

---

## 2. Where I propose to deviate from the letter of the brief, and why

**D1. Playout feeds bytes, not a playlist.** One long-running publisher:

    ffmpeg -f mpegts -i pipe:0 -c copy -f flv rtmp://127.0.0.1:1935/road

It never restarts between segments. Python opens each segment in order and writes its bytes into that pipe at real-time pace, using the segment's own PCR clock (fact 5) to decide when each ~100 ms slice is due. FFmpeg's built-in discontinuity handling (fact 3) joins the segments into one continuous stream. Why not the concat demuxer: fact 1. Why not `-re`: fact 2. What you get: a hold is simply "Python stops writing"; when the next segment arrives it continues at 1× from the pointer, and the delay has grown by the length of the gap. Nothing ever bursts. The "playlist" you asked for still exists — as `playout.log`, one line per segment aired — but FFmpeg does not read it.

**D2. MediaMTX does the eight-way fan-out itself.** Fact 6. No FFmpeg forwarder processes on the relay. Copy only: MediaMTX re-packetizes, it does not transcode. Kill or stop = stop the publisher; MediaMTX tears down the eight outgoing connections when the path loses its source.

**D3. Keys: `.env` → rendered config on tmpfs.** `/etc/relay/relay.env` (mode 600, root, not in git) holds `YOUTUBE_URL=` / `YOUTUBE_KEY=` pairs, one pair per platform. When MediaMTX starts, `render_config.py` (an `ExecStartPre=`) fills `mediamtx.yml.template` and writes `/run/relay/mediamtx.yml` — tmpfs, so RAM only, mode 600, owned by the `mediamtx` user, gone at reboot. Keys are never in git, never in process arguments, never in the log (fact 7). The template in git has no secrets. A platform with an empty key is left out of the rendered config and shown as "not configured" in `status`, rather than becoming a forward that fails forever. (MediaMTX could also take these straight from environment variables — `MTX_PATHS_ROAD_FORWARD_0_DEST=…` — which would need no rendering code at all. I chose the template because it keeps human names next to the values and matches your brief. Recorded as the alternative.)

**D4. The deck talks to the playout controller, not to MediaMTX's API.** Start, stop, kill and brb are decisions about which bytes go into the pipe; MediaMTX has no notion of them. Each press is one SSH command to the relay: `relayctl <command>`. The deck's SSH key is pinned in `authorized_keys` to that one command (`command="…"`), and `relayctl` accepts only `toggle`, `kill`, `brb`, `status --json` when invoked that way — so the key cannot do anything else even if the MacBook walks off. SSH rather than HTTP because: the truck→relay SSH path must exist anyway for rsync; there is no server to write, no TLS, no token scheme; and an SSH `ControlMaster` connection kept open makes a press cost about 50 ms. MediaMTX's control API (localhost only) is used by `relayctl status` to report the path and the eight destinations.

**D5. Capture names files on completion.** FFmpeg writes `writing/seg-000123.ts` (its counter). The segment muxer writes one file at a time, so any file in `writing/` that is not the highest-numbered is finished. `capture.py` renames it to `segments/seg-YYYYMMDD-HHMMSS-000123.ts` — wall-clock from the file's birth time (macOS keeps it), in **UTC** so names keep sorting correctly through DST changes — and writes a sidecar `segments/seg-….ts.ok` holding byte size, SHA-256, duration, start time and the stream signature (codec, size, frame rate, sample rate, channels). This is your "temporary name then rename," and it also solves fact 4. On restart the counter continues from the highest number found in `writing/` or `segments/`. A leftover in `writing/` after a crash is moved to `segments/` as-is: MPEG-TS, still playable; nothing is deleted.

**D6. Every segment has an audio track.** If no audio device is configured, capture encodes silence (`anullsrc` → AAC) instead of omitting audio. Two reasons: the stream layout must be identical across every segment for D1 to work, and MediaMTX's own docs warn that YouTube silently rejects video-only streams. The BRB clip is rendered with the same layout.

**D7. The `.ok` sidecar is the completeness signal on the relay.** Fact 9: with `--append-verify`, a half-transferred segment sits under its final name. Playout treats a segment as ready only when `.ts` and `.ok` are both present and the size matches; the SHA-256 is checked once, the first time the segment is seen. rsync transfers files one at a time in sorted order, so `seg-X.ts` lands before `seg-X.ts.ok`; the size and hash checks cover the corner cases anyway. The archive uses the same sidecar for its verification.

**D8. Hold = stop feeding.** With copy-only on the relay, nothing can synthesize a frozen last frame — that is an encode. What viewers see when the buffer runs dry is whatever their player does with a stalled stream: a frozen picture, then a spinner. The publisher connection and the eight forwards stay open (`readTimeout` raised, fact 8). Each platform drops on its own timeout if the hold outlasts it; MediaMTX reconnects within 5 s once data flows again. Every gap is logged with start, length and pointer. If you later want a rendered "holding" card instead of a stall, that is one pre-rendered segment away — not in 0.1.

**D9. Archive pull in three steps.** (1) `rsync` `aired/` from the relay to local storage — nothing removed. (2) Verify every new file against its `.ok` sidecar (size and SHA-256), locally. (3) `rsync --remove-source-files --files-from=<verified list>` — the files are identical so nothing is re-sent, and rsync removes exactly those source files (fact 9). Anything that fails verification stays on the relay and is reported. The isolated box's key on the relay is confined with `rrsync /srv/relay/aired`: it can read and delete inside that one directory and nothing else.

**D10. Encoder.** `h264_videotoolbox` (Apple's hardware encoder) by default: near-zero CPU and heat in a cab that sits in the sun. `libx264` is the alternative, one line in `capture.conf`. Both honour forced keyframes at the 20-second boundary (fact 11). Keyframes every 2 s (`-g 60` at 30 fps) because the platforms want them; the 20-second boundaries land on those keyframes.

**D11. After a relay reboot, playout comes up `stopped` with its pointer intact.** Nothing airs until button 1. Predictable beats clever: you might have been BRB, and an unattended auto-resume is a surprise.

**D12. If the publisher process dies on its own** (not stop, not kill), playout logs it, restarts it, and resumes from the beginning of the segment that was airing. At most 20 seconds are seen twice; nothing is lost. This is the one situation where a repeat can happen, and it is logged as such.

---

## 3. Files

Configuration files are INI (`key = value`, stdlib `configparser`) — readable, no dependency. State files are JSON. Logs are plain lines, `2026-09-26T14:30:00Z LEVEL message`. FFmpeg's own stderr goes to its own file, rotated daily. Every `status` command prints for a human and takes `--json` for a machine. Python 3.13 everywhere (Homebrew `python@3.13` on the MacBook — the system `/usr/bin/python3` is 3.9 and too old).

```
Relapse/
├── README.md                 (existing; one section updated, see §5)
├── PLAN.md                   this document
├── DECISIONS.md              the decision log; starts with D1–D12
├── .gitignore                relay.env, __pycache__, .DS_Store, local state
├── main.py, history/         untouched
│
├── capture/                  MacBook
│   ├── README.md
│   ├── capture.conf
│   ├── capture.py
│   ├── com.texrig.capture.plist
│   ├── requirements.txt
│   └── tests/test_capture.py
│
├── ship/                     MacBook
│   ├── README.md
│   ├── ship.conf
│   ├── ship.py
│   ├── com.texrig.ship.plist
│   ├── requirements.txt
│   └── tests/test_ship.py
│
├── relay/                    Debian 13 VM
│   ├── README.md
│   ├── setup_relay.py
│   ├── relay.conf
│   ├── relay.env.example
│   ├── mediamtx.yml.template
│   ├── render_config.py
│   ├── playout.py
│   ├── relayctl.py
│   ├── make_brb.py
│   ├── brb/README.md
│   ├── systemd/mediamtx.service
│   ├── systemd/relay-playout.service
│   ├── requirements.txt
│   └── tests/test_playout.py
│
├── deck/                     wherever the Stream Deck is plugged in
│   ├── README.md
│   ├── deck.conf
│   ├── deck.py
│   ├── com.texrig.deck.plist
│   ├── requirements.txt
│   └── tests/test_deck.py
│
└── archive/                  isolated server
    ├── README.md
    ├── archive.conf
    ├── pull_archive.py
    ├── crontab.example
    └── tests/test_archive.py
```

### 3.1 `capture/` — MacBook, camera → 20-second segments

Working directory on the Mac: `~/roadstream/` with `writing/`, `segments/`, `logs/`, `state/`. 20 s at ~6.2 Mbps ≈ 15.5 MB per segment, ≈ 2.8 GB/hour, ≈ 67 GB/day.

| File | What it does |
|---|---|
| `README.md` | Install (Homebrew `ffmpeg`, pinned; `python@3.13`), grant camera/microphone permission, find the ArduCam with `list-devices`, edit `capture.conf`, load the launchd agent, read `status`, what every log line means, how to test without a camera. |
| `capture.conf` | Video device index, audio device index (blank = silent track), `video_size = 1920x1080`, `framerate = 30`, `pixel_format`, `encoder = h264_videotoolbox` or `libx264`, `bitrate = 6000k`, `segment_seconds = 20`, paths. No secrets; committed. |
| `capture.py` | Single entry point. `capture.py run` — builds the FFmpeg command from the config, starts FFmpeg writing `writing/seg-%06d.ts` with `-segment_start_number` = highest existing + 1, then loops every second: rename finished files into `segments/` with the final name, write the `.ok` sidecar, log `name, bytes, duration`; restart FFmpeg if it exits; on SIGTERM stop FFmpeg cleanly and exit. `capture.py status` — FFmpeg running?, file now being written and its age, last completed segment, segments today, disk free and days of headroom at the current rate, warnings (no segment in 40 s, disk low, FFmpeg restarts). `capture.py list-devices` — runs `ffmpeg -f avfoundation -list_devices true -i ""` and prints the device table. `capture.py probe <index>` — asks FFmpeg for the modes the device supports (useful once, to fill in `capture.conf`). `capture.py run --source test` — same pipeline fed by FFmpeg's test pattern and a tone instead of the camera: lets the whole chain be tested on a desk, and lets me test it here. |
| `com.texrig.capture.plist` | launchd agent: `RunAtLoad`, `KeepAlive` (restart on failure), throttled restarts, stdout/stderr to `logs/`. |
| `requirements.txt` | Stdlib only; the file says so and pins the interpreter (`python 3.13`). |
| `tests/test_capture.py` | Unit tests for: final-name construction, "which files in `writing/` are finished", counter continuation, sidecar contents, FFmpeg command construction (silent vs real audio, both encoders). `python3 -m unittest`. |

FFmpeg command shape (built by `capture.py`, logged in full at start):

```
ffmpeg -nostdin -hide_banner -loglevel warning
  -f avfoundation -framerate 30 -video_size 1920x1080 -pixel_format <from conf> -i "<video>:<audio>"
  [ -f lavfi -i anullsrc=r=48000:cl=stereo   if no audio device ]
  -c:v h264_videotoolbox -b:v 6000k -maxrate 6500k -bufsize 12000k -g 60
  -force_key_frames "expr:gte(t,n_forced*20)"
  -c:a aac -b:a 128k -ar 48000 -ac 2
  -f segment -segment_time 20 -segment_format mpegts -reset_timestamps 1
  -segment_start_number <N> writing/seg-%06d.ts
```

Never deletes. Never overwrites: if a target name already exists (cannot happen with a continuing counter, but Poka-Yoke), it logs an error and leaves the file in `writing/`.

### 3.2 `ship/` — MacBook, segments → relay over rsync

| File | What it does |
|---|---|
| `README.md` | SSH key setup (the relay confines this key with `rrsync -wo /srv/relay/incoming`: it can only write into that one directory), Homebrew `rsync` (macOS's bundled rsync is not the real thing), `ship.conf`, launchd, reading `status`, what happens on a link drop. |
| `ship.conf` | Relay host, port, user, key path, remote directory, poll interval (default 5 s), rsync binary path, paths. Committed. |
| `ship.py` | `ship.py run` — takes a lock (`flock`; a second copy exits immediately), then loops: find `.ts` and `.ok` files in `segments/` not yet recorded as shipped, oldest first; write them to a temporary `--files-from` list; run one `rsync -a --partial --append-verify --files-from=… --timeout=60 -e "ssh … ConnectTimeout=15 ServerAliveInterval=15 ServerAliveCountMax=2 BatchMode=yes"`; exit 0 → record every listed file as shipped in `state/shipped.json`; anything else → log the rsync exit code and message, wait with backoff (5 s doubling to 60 s), try again. The link dropping mid-file is the normal case: `--append-verify` resumes at the byte where it stopped and verifies the whole file. `ship.py status` — segments on disk, shipped, unshipped with minutes of video (from the sidecars), oldest unshipped and its age, last rsync result, current backoff, disk free. |
| `com.texrig.ship.plist` | launchd agent, same shape as capture's. |
| `requirements.txt` | Stdlib only. |
| `tests/test_ship.py` | Backlog selection and ordering, state-file round trip, backlog-minutes arithmetic, rsync command construction, backoff schedule, lock behaviour. |

Never deletes locally. `shipped.json` is the only thing it writes besides logs.

### 3.3 `relay/` — Debian 13 VM: MediaMTX + playout

Layout on the VM: code in `/opt/relay/`; data in `/srv/relay/` — `incoming/`, `aired/`, `brb/`, `control/`, `state/`, `logs/`; keys in `/etc/relay/relay.env`; MediaMTX binary `/usr/local/bin/mediamtx`; rendered config `/run/relay/mediamtx.yml`. Two Unix users: `mediamtx` (runs MediaMTX) and `relay` (runs playout, owns the data, is the SSH user for the truck, the deck and the archive box — three keys, three forced commands). Listening: sshd; MediaMTX RTMP on `127.0.0.1:1935`, API on `127.0.0.1:9997`, HLS on `:8888` (LAN, for you). RTSP, WebRTC, SRT, metrics, playback: off.

| File | What it does |
|---|---|
| `README.md` | Fresh-VM steps (`setup_relay.py`), filling `relay.env`, the three `authorized_keys` lines and why each is confined, watching the stream (browser: `http://<relay>:8888/road/`, VLC: `http://<relay>:8888/road/index.m3u8`), `relayctl` commands, reading `status`, the hold and BRB behaviour, disk maths, what each log line means. |
| `setup_relay.py` | Run once as root on a fresh Debian 13. Installs `ffmpeg rsync python3 openssh-server` from trixie and records the installed versions to `/etc/relay/versions.txt`; creates the users and directories with the right modes; downloads `mediamtx_v1.21.1_linux_amd64.tar.gz` and `checksums.sha256` from the pinned release, verifies the SHA-256 (and refuses to install on mismatch), installs to `/usr/local/bin/mediamtx`; installs the two systemd units, enables them; prints what is left for a human (the `.env`, the keys). Idempotent — safe to run twice. Reports every step as it goes. |
| `relay.conf` | `buffer_minutes = 3`, paths, MediaMTX API address, publisher command options, log settings. Committed. |
| `relay.env.example` | Template for `/etc/relay/relay.env`: eight `NAME_URL=` / `NAME_KEY=` pairs with the documented ingest URL for each platform as a comment ("check your dashboard — these move"). The real file is mode 600 and never committed (`.gitignore`). |
| `mediamtx.yml.template` | The whole MediaMTX config, no secrets: `readTimeout: 30m` (fact 8), RTMP and API on localhost only, HLS on the LAN (`hlsVariant: mpegts` so VLC and the built-in browser player both work), everything else off, one path `road` accepting publish only from `127.0.0.1`, and the eight `forward:` destinations as `${YOUTUBE_URL}#${YOUTUBE_KEY}` placeholders. |
| `render_config.py` | Reads `relay.env`, fills the template, writes `/run/relay/mediamtx.yml` (mode 600, owner `mediamtx`), skips platforms with an empty key, refuses to write if a URL is malformed. Runs as `ExecStartPre=+` of `mediamtx.service`. `render_config.py --check` shows what would be rendered with keys masked. |
| `playout.py` | The long-running feeder (`relay-playout.service`). Loop, twice a second: read `control/command` if present; scan `incoming/` for ready segments (D7) and skip any whose stream signature differs from the pointer's (moved to `aired/` unplayed, logged); compute buffer depth from the pointer to the newest; act on state. `live`: keep the publisher running; feed the pointer segment byte-paced by PCR (D1); when it is fully written, move it to `aired/`, advance the pointer, log one line. If the next segment is not ready: hold — write nothing, log the gap once when it starts and once when it ends. `waiting`: started but buffer < N minutes; becomes `live` on its own when the buffer fills. `stopped`: publisher not running; segments keep landing. `killed`: same as stopped but only `start` gets it out. `brb`: feed `brb/brb.ts` repeatedly instead of road; on resume, jump the pointer to the newest segment that still leaves N minutes ahead, move everything older into `aired/` unplayed (logged as skipped); if the ready segments total less than N minutes the pointer does not move. Writes `state/state.json` every loop. Handles SIGTERM: stop the publisher, write state, exit. |
| `relayctl.py` | The command and status tool. `relayctl start / stop / kill / brb-on / brb-off` write an explicit command to `control/command` (idempotent: last writer wins and that is correct). `relayctl toggle` and `relayctl brb` read `state.json` and resolve to the explicit command — so the deck never has to guess from stale state. `relayctl status` prints playout state, pointer and its age, buffer depth, last gap, BRB, publisher uptime, MediaMTX path state and each of the eight destinations (from the API, names from `relay.env`, never the key), `incoming/` and `aired/` counts and sizes, disk free with a warning under 100 GB, last archive pull. When invoked through the deck's SSH forced command it accepts only the whitelisted commands. |
| `make_brb.py` | Run once (and again whenever you change the card). Renders `brb/brb.ts` from `brb/card.png` and `brb/engine.wav` with the same codec parameters as the road segments — H.264 1920×1080 at the capture frame rate, AAC 48 kHz stereo, 20 seconds, MPEG-TS, keyframe at the start — so the copy-only publisher can switch to it seamlessly. Writes a `brb.ts.ok` sidecar in the same format as capture's, which is how playout knows the signature matches. Rendered once on the relay with Debian's ffmpeg; this is a build step, not live transcoding. |
| `brb/README.md` | What to put there: `card.png` (1920×1080) and `engine.wav` (any length; looped to fill 20 s). |
| `systemd/mediamtx.service` | `User=mediamtx`, `RuntimeDirectory=relay`, `ExecStartPre=+/opt/relay/render_config.py`, `ExecStart=/usr/local/bin/mediamtx /run/relay/mediamtx.yml`, `Restart=always`. |
| `systemd/relay-playout.service` | `User=relay`, `After=mediamtx.service`, `ExecStart=/opt/relay/playout.py`, `Restart=always`, `KillMode=mixed` so the FFmpeg child is stopped with the service. |
| `requirements.txt` | Stdlib only. |
| `tests/test_playout.py` | Readiness rules (D7), buffer-depth arithmetic, pointer selection on BRB resume, the PCR reader against a synthesized transport stream, command-file handling, state-file round trip, signature comparison, gap logging. Where FFmpeg is available, an end-to-end test: generate three test segments, feed them through the real publisher into a local MediaMTX (or a plain file sink), check the output is one continuous stream. |

The `.env` names and the `status` labels: `youtube twitch kick rumble facebook x tiktok dailymotion`.

### 3.4 `deck/` — Stream Deck → relay

| File | What it does |
|---|---|
| `README.md` | `brew install hidapi` (the `streamdeck` library's HID backend), `pip install -r requirements.txt`, the SSH key and the `authorized_keys` line that confines it, `deck.conf`, what each face means, launchd. A note on running it on Linux instead (udev rule) if the deck ends up plugged into something other than the Mac. |
| `deck.conf` | Relay SSH target and key, key numbers for the three buttons (defaults 0, 1, 2), poll interval (3 s), brightness. |
| `deck.py` | Opens the first Stream Deck found, clears every key, draws three faces (Pillow, text on colour), polls `relayctl status --json` over a persistent SSH connection (`ControlMaster`/`ControlPersist`, so a press is one short command) and redraws faces from the answer: **button 1** LIVE (green) / STOPPED (grey) / WAITING 1:20 (blue) → press sends `toggle`; **button 2** KILL (dark red) / KILLED (bright red) → press sends `kill`; **button 3** BRB (yellow when on) → press sends `brb`. After a press it re-polls immediately. Two failed polls in a row → all three faces show a warning (`NO RELAY`) and presses are still attempted. Logs every press and every state change. Runs as a launchd agent. |
| `com.texrig.deck.plist` | launchd agent. |
| `requirements.txt` | `streamdeck==0.10.0`, `Pillow==12.3.0`, interpreter `python 3.13`; `hidapi 0.15.0` via Homebrew noted in the README. |
| `tests/test_deck.py` | Face rendering (produces an image of the right size for each state), state→face mapping, SSH command construction, warning logic after failed polls. No hardware needed. |

### 3.5 `archive/` — isolated server, nightly pull

| File | What it does |
|---|---|
| `README.md` | The key (confined on the relay with `rrsync /srv/relay/aired`), `archive.conf`, the cron line, what the three steps do, reading `status` and the log, what a verification failure looks like and what to do. |
| `archive.conf` | Relay host/user/key, local root (e.g. `/archive/road/`), schedule note, minimum free space to keep. |
| `pull_archive.py` | `pull_archive.py run` — lock; step 1 `rsync -a --partial --append-verify relay:/srv/relay/aired/ /archive/road/incoming/`; step 2 for every `.ts` with a `.ok` in `incoming/`: size and SHA-256 must match the sidecar, else log and leave it; step 3 `rsync --remove-source-files --files-from=verified.list` against the relay (D9); then move verified pairs into `/archive/road/YYYY-MM-DD/` by their name (a local rename, no deletion). Writes `state/last_run.json`. `pull_archive.py status` — last run time and result, files pulled/verified/removed/failed, oldest file still on the relay, local disk free. |
| `crontab.example` | `10 2 * * * /archive/road/pull_archive.py run >> /archive/road/logs/cron.log 2>&1`. |
| `tests/test_archive.py` | Verification against sidecars (good file, wrong size, wrong hash, missing sidecar), date-folder placement, rsync command construction. |

The relay never initiates anything toward this box. This script is the only thing in the whole system that deletes a media file, and only in step 3, only files it verified in step 2.

### 3.6 Root

| File | What it does |
|---|---|
| `DECISIONS.md` | D1–D12 above, in the same words, plus every further decision made while building, each with the alternative and why not. |
| `.gitignore` | `relay.env`, `*.pyc`, `__pycache__/`, `.DS_Store`, `state/`, `logs/`. |
| `README.md` | Existing file kept. I would replace the "Current Contents" list (which says capture scripts are coming in `scripts/`) with a pointer to the five directories, `PLAN.md` and `DECISIONS.md`, and leave your vision, hardware and phases text alone. See §5. |

### 3.7 Pinned versions

| Thing | Pin | Where recorded |
|---|---|---|
| MediaMTX | v1.21.1, `mediamtx_v1.21.1_linux_amd64.tar.gz`, SHA-256 from the release's `checksums.sha256`, checked by `setup_relay.py` | `relay/setup_relay.py`, `relay/README.md` |
| FFmpeg on the relay | Debian trixie package `7:7.1.5-0+deb13u1` (source: Debian main; version recorded at install) | `relay/README.md`, `/etc/relay/versions.txt` |
| FFmpeg on the Mac | Homebrew `ffmpeg 9.0.2`, held with `brew pin ffmpeg` | `capture/README.md` |
| rsync on the Mac | Homebrew `rsync 3.5.1` | `ship/README.md` |
| Python | 3.13 both sides (Homebrew `python@3.13`; trixie `python3 3.13.5`) | each `requirements.txt` |
| streamdeck / Pillow / hidapi | `0.10.0` / `12.3.0` / Homebrew `0.15.0` | `deck/requirements.txt` |

---

## 4. Build and test order

Capture → ship → relay → deck → archive, each tested in isolation before the next, each committed and pushed as it passes.

What "tested" means here, honestly: I am working in a Linux container without a camera, a Mac, a Stream Deck or a satellite link. So:

- Every piece gets unit tests over its pure logic (naming, selection, arithmetic, command construction, state files). Those run here.
- Capture, playout and the archive verifier also get run end to end here with FFmpeg's test pattern standing in for the camera (`capture.py run --source test` on Linux uses `lavfi` instead of `avfoundation`; everything downstream is identical). If this container can reach Debian's mirrors I install FFmpeg and MediaMTX and run the real publisher into a real MediaMTX; if it cannot, I say so and the end-to-end run happens on your relay.
- What only you can test, and the README for each piece will say so explicitly: AVFoundation device indices and the camera's supported modes; macOS camera and microphone permission for a launchd agent (I expect the first run to need a manual launch from Terminal to trigger the permission prompt — the README covers it); the real Stream Deck; the real satellite link; the eight platform ingests.

Step by step:

1. **capture** — write, unit test, run the test-pattern source here for a few minutes, check with `ffprobe` that every segment starts on a keyframe, is 20.0 s, plays standalone, and that names, sidecars and the counter behave across a restart. Commit.
2. **ship** — write, unit test, run here against a local sshd (or a second directory over `localhost`), pull the "cable" by killing the connection mid-transfer, confirm resume with no re-send and no duplicate. Commit.
3. **relay** — `setup_relay.py` exercised here as far as the container allows; playout unit tests; the end-to-end publisher test; `render_config.py` with a dummy `.env`; `make_brb.py` with a generated card. Commit.
4. **deck** — unit tests with the faces rendered to PNG files so you can look at them; the SSH path tested against the relay's `relayctl` here. Commit.
5. **archive** — full three-step run here between two directories over `localhost` ssh, including a deliberately corrupted file that must stay on the source. Commit.
6. `DECISIONS.md` grows throughout; final pass on every README against the definition of done.

---

## 5. What I need from you

1. **Approval of D1–D12**, or the ones you want changed. D1 (bytes through a pipe instead of the concat demuxer) and D8 (a hold is a stall) are the two that most change the shape of the thing.
2. **Where does the Stream Deck plug in?** I am assuming the MacBook in the cab, so `deck/` ships a launchd plist. If it lives elsewhere, say where and I will ship a systemd unit instead.
3. **Root `README.md`**: may I replace the "Current Contents" section as described in §3.6? Everything else in it stays as you wrote it.
4. **How the truck reaches the relay** (port-forward on the fibre router, Cloudflare tunnel, Tailscale) is a config value in `ship.conf` and `deck.conf` — `host`, `port`, optional `ProxyCommand`. I will assume plain SSH to a host:port and not build tunnel setup. Tell me if you want a specific `ProxyCommand` documented.

Two constraints I am not solving in 0.1, so you know they exist:

- **Capture disk.** Nothing on the Mac is ever deleted, and it records ~67 GB/day. A 2 TB disk fills in roughly 30 days of driving. `capture status` shows days of headroom; moving old segments off the Mac is a manual job outside 0.1.
- **Platform bitrate caps.** One stream goes to all eight; nothing is transcoded per platform. 6 Mbps is at or above the documented cap on some of them (Facebook and Dailymotion in particular). If one platform balks, the fix is a lower `bitrate` in `capture.conf` for everyone, or a decision about that platform. I have not verified each platform's current cap and will mark them "check your dashboard" in `relay.env.example`.
