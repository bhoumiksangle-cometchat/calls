# Jitsi Meet — custom local setup (Skynet live transcription)

This is a local Jitsi Meet stack with **live transcription / subtitles** powered by
**Skynet** (Faster‑Whisper), running **CPU‑only**.

Base distribution: **docker‑jitsi‑meet `stable-9646`**. Host used: **macOS, arm64**,
Docker 29.4.3, Compose v5.1.4.

> **Scope.** The base distribution still ships a SIP gateway (`jigasi.yml` + the `jigasi/`
> dir), but the mock Asterisk PBX that backed it has been removed — nothing here registers a
> SIP account. A soundboard is planned as a **custom jigasi plugin** exposing a `/play` REST
> endpoint, replacing the earlier Asterisk‑ConfBridge approach; it is not part of this stack yet.

---

## Run it

The stack is **two stacked compose files**:

```bash
cd docker-jitsi-meet-stable-9646

# start
docker compose -f docker-compose.yml -f skynet.yml up -d

# status
docker compose -f docker-compose.yml -f skynet.yml ps

# logs (the transcriber)
docker compose -f docker-compose.yml -f skynet.yml logs -f jigasi-transcriber

# stop
docker compose -f docker-compose.yml -f skynet.yml down
```

Then open **https://localhost:8443** (accept the self‑signed cert) and join a room.

> Tip: `alias jit='docker compose -f docker-compose.yml -f skynet.yml'`, then `jit up -d`, `jit ps`, etc.
>
> GPU host? Layer the GPU override **last**: `jit -f skynet.gpu.yml up -d`
> (needs the GPU image built as `skynet-gpu:local` + NVIDIA drivers + nvidia‑container‑toolkit).

---

## Containers (6) and versions

| Service | Image | Role |
|---|---|---|
| web | `jitsi/web:stable-9646` | Nginx + Jitsi Meet front‑end |
| prosody | `jitsi/prosody:stable-9646` | XMPP server (the hub) |
| jicofo | `jitsi/jicofo:stable-9646` | conference focus / orchestrator |
| jvb | `jitsi/jvb:stable-9646` | media (audio/video bridge) |
| jigasi-transcriber | `jitsi/jigasi:stable-9753` | jigasi in transcriber mode → Skynet |
| skynet | `skynet-cpu:local` (built locally) | live transcription (Faster‑Whisper `tiny.en`, int8, CPU) |

`jigasi-transcriber` is intentionally a **newer** jigasi: `stable-9646` jigasi has no Whisper
support; `stable-9753` is the closest version that does. It uses the **published image
directly** — it does *not* build from the stock `jigasi/` dir.

All containers share one Docker network, **`meet.jitsi`**, and resolve each other by name /
alias (`xmpp.meet.jitsi` → prosody, `skynet` → skynet).

---

## How it's connected

```
Browser ──HTTPS :8443──> web ──XMPP/BOSH──> prosody  (everyone authenticates here)
                                               │
        jicofo, jvb, jigasi-transcriber ───────┘   all log into prosody
          │
          │  jicofo owns the "jigasibrewery" MUC; the transcriber waits in it.
          │  When a meeting asks for subtitles, jicofo dispatches the transcriber:
          │     subtitles → jigasi-transcriber ──audio / WebSocket──> skynet (Whisper) ──text──> room
```

> jicofo only creates/owns the `jigasibrewery` when `JIGASI_SIP_URI` is **set** at start, so
> that variable stays set in `.env` even though there's no SIP gateway now (see fixes below).

### Ports
| Mapping | Purpose |
|---|---|
| host `8443` (HTTPS), `8000` (HTTP) → web | join meetings |
| host `10000/udp` → jvb | media |
| host `127.0.0.1:8009` → skynet `8000` | debugging only |
| internal `xmpp.meet.jitsi:5222` | jvb / jicofo / transcriber → prosody |
| internal `skynet:8000` | transcriber → Whisper websocket |

---

## Custom files added to the distribution

| File | What it is |
|---|---|
| `skynet.yml` | the `skynet` container + the `jigasi-transcriber` container |
| `skynet.gpu.yml` | optional GPU override for `skynet` (CUDA, `small.en`); layer it last |
| `.env` | all the knobs below (**git‑ignored** — holds passwords + an RSA key) |
| `../skynet/` | cloned `jitsi/skynet` source; the CPU image is built from it as `skynet-cpu:local` |

Rebuild the Skynet CPU image (if needed):

```bash
docker build \
  --build-arg BASE_IMAGE_BUILD=ubuntu:22.04 \
  --build-arg BASE_IMAGE_RUN=ubuntu:22.04 \
  --build-arg BUILD_WITH_VLLM=0 \
  -t skynet-cpu:local ../skynet
```

---

## Key `.env` settings

`.env` is git‑ignored (it holds secrets); recreate it locally. The transcriber needs:

| Variable | Value | Why |
|---|---|---|
| `ENABLE_TRANSCRIPTIONS` | `1` | web CC button + jicofo transcriber dispatch |
| `JIGASI_SIP_URI` | any value (e.g. `transcriber@meet.jitsi`) | must be **set** so jicofo creates/owns the `jigasibrewery` the transcriber joins; it no longer needs to reach a real SIP server |
| `JIGASI_XMPP_PASSWORD` | (set) | brewery control connection (`jigasi@auth`) password, shared by the transcriber |
| `JIGASI_TRANSCRIBER_PASSWORD` | (hex) | password for `transcriber@auth.meet.jitsi` |
| `JIGASI_TRANSCRIBER_WHISPER_PRIVATE_KEY` / `_NAME` | RSA key / `jitsi` | jigasi signs a JWT before opening the Whisper socket (Skynet bypasses auth and ignores it, but jigasi still needs a key) |

### Out‑of‑band (not in a file)
- Prosody transcriber account registered by hand (the `recorder` domain doesn't exist in this build):
  ```bash
  docker exec <prosody> prosodyctl --config /config/prosody.cfg.lua \
    register transcriber auth.meet.jitsi "$JIGASI_TRANSCRIBER_PASSWORD"
  ```
- Skynet model volume must be writable by uid 1001:
  ```bash
  docker run --rm -v docker-jitsi-meet-stable-9646_skynet-models:/models alpine \
    chown -R 1001:1001 /models
  ```

### Notable `jigasi-transcriber` env (in `skynet.yml`)
- `JIGASI_MODE=transcriber`
- `XMPP_RECORDER_DOMAIN=auth.meet.jitsi` (transcriber account lives here)
- `JIGASI_TRANSCRIBER_CUSTOM_SERVICE=org.jitsi.jigasi.transcription.WhisperTranscriptionService`
- `JIGASI_TRANSCRIBER_WHISPER_URL=ws://skynet:8000/streaming-whisper/ws`
- `JIGASI_CONFIGURATION=org.jitsi.jigasi.xmpp.acc.SERVER_ADDRESS=xmpp.meet.jitsi`  ← critical override

### Notable `skynet` env (in `skynet.yml`)
`ENABLED_MODULES=streaming_whisper`, `BYPASS_AUTHORIZATION=true`,
`WHISPER_MODEL_NAME=tiny.en`, `WHISPER_DEVICE=cpu`, `WHISPER_COMPUTE_TYPE=int8`, `BEAM_SIZE=1`.

---

## How to test

- **Meeting:** open https://localhost:8443, join a room.
- **Subtitles:** ⋯ (More) → **Subtitles** → English → speak. Captions appear after a short delay.
- **Health checks:**
  ```bash
  curl -s http://127.0.0.1:8009/ -o /dev/null -w "%{http_code}\n"                    # skynet up
  docker compose -f docker-compose.yml -f skynet.yml logs -f jigasi-transcriber       # watch it join + transcribe
  ```

> **Not re‑verified since SIP removal.** This stack previously ran the SIP gateway *alongside*
> the transcriber. Running transcription **without** the SIP gateway (as documented here) is the
> intended setup, but re‑test dispatch on your next run: if subtitles never appear, confirm
> `JIGASI_SIP_URI` is set in `.env` and the prosody `transcriber@auth.meet.jitsi` account exists.

---

## Problems hit during setup, and the fixes

1. **`stable-9646` jigasi has no Whisper** — added a dedicated jigasi (`stable-9753`) in
   transcriber mode (the `jigasi-transcriber` service; this is the "update to 9753").
2. **Skynet is GPU/amd64 by default; host is arm64 CPU** — rebuilt CPU‑only (`ubuntu:22.04`
   base, `BUILD_WITH_VLLM=0`), `streaming_whisper` only, `tiny.en`/`int8`.
3. **jicofo didn't dispatch a transcriber** — jicofo only manages the `jigasibrewery` when
   `JIGASI_SIP_URI` is set at start; keep it set in `.env` (it need not resolve) and
   regenerate jicofo config.
4. **`recorder.meet.jitsi` account didn't exist** — registered `transcriber@auth.meet.jitsi` instead.
5. **Whisper model download `PermissionError`** — `chown` the model volume to uid 1001.
6. **Transcriber `No server addresses found`** — the newer template aimed the conference login
   at `PUBLIC_URL` (`localhost:8443` = the container itself); override
   `xmpp.acc.SERVER_ADDRESS=xmpp.meet.jitsi`.
7. **`Failed generating JWT`** — jigasi always signs a JWT even when Skynet bypasses auth;
   gave it a throwaway RSA key.

---

## Caveats / tuning

- Transcription is **Whisper `tiny.en` on CPU** — fast/free tier, modest accuracy + a little
  latency. For better quality set `WHISPER_MODEL_NAME=base.en` (or `small.en`) in `skynet.yml`
  and `docker compose -f docker-compose.yml -f skynet.yml up -d --force-recreate skynet`.
  On a GPU host, use `skynet.gpu.yml` instead.
- If you change `.env`, the affected container's generated config under
  `~/.jitsi-meet-cfg/<svc>` may need clearing before `up -d --force-recreate <svc>` so it regenerates.

---

## Not included (yet)

- **SIP gateway** — `jigasi.yml` + the `jigasi/` dir still ship with the base distribution, but
  the mock Asterisk PBX that backed them was removed; no SIP account is registered.
- **Soundboard** — to be added as a **custom jigasi plugin** exposing a `/play` REST endpoint
  (and other custom features), replacing the earlier Asterisk‑ConfBridge soundboard.
