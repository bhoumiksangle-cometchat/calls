# Project Overview — Self-Hosted Jitsi Meet + Live Transcription + Soundboard

A self-hosted video-conferencing stack based on **docker-jitsi-meet**, extended with:
1. **SIP gateway** (jigasi + mock Asterisk) — dial into meetings over SIP
2. **Live transcription** (jigasi transcriber + Skynet/Whisper) — subtitles in-meeting
3. **Discord-style soundboard** — toolbar button plays sound clips into the call

Repo: `https://github.com/bhoumiksangle-cometchat/calls.git` (this directory is
`docker-jitsi-meet-stable-9646/` inside it).

---

## Architecture

```
Browser ──HTTPS:8443──> web (nginx + jitsi-meet UI)
Browser ──UDP:10000───> jvb (video bridge: all audio/video media)

web ──> prosody (XMPP signaling) <── jicofo (conference focus/orchestrator)

TRANSCRIPTION:
jicofo ──dispatch──> jigasi-transcriber (joins room as hidden participant)
jigasi-transcriber ──audio──> skynet ws://skynet:8000 (Faster-Whisper) ──text──> subtitles

SIP:
jigasi (SIP mode) <──SIP──> sipserver (Asterisk mock PBX)

SOUNDBOARD:
Browser ──POST /soundboard/play──> nginx ──> soundboard-rest (Python, stdlib)
soundboard-rest ──AMI──> sipserver plays WAV into ConfBridge
jigasi-soundboard (SIP bot in that ConfBridge) relays the clip into the Jitsi room
```

## Components & versions

| Service | Image | Role |
|---|---|---|
| web | `jitsi/web:stable-9753` | UI + nginx (SSI hook injects soundboard JS) |
| prosody | `jitsi/prosody:stable-9753` | XMPP |
| jicofo | `jitsi/jicofo:stable-9753` | conference focus; owns `jigasibrewery` |
| jvb | `jitsi/jvb:stable-9753` | media bridge (UDP 10000) |
| jigasi | `jitsi/jigasi:stable-9753` | SIP gateway mode |
| jigasi-transcriber | `jitsi/jigasi:stable-9753` | `JIGASI_MODE=transcriber` |
| skynet | `skynet-gpu:local` (built from `resources/skynet/`) | streaming Whisper (GPU cuda/float16 or CPU int8) |
| sipserver | Asterisk (mock-sip/) | PBX + ConfBridge + AMI |
| jigasi-soundboard | `jitsi/jigasi` (SIP mode) | soundboard audio relay |
| soundboard-rest | built from `soundboard/rest/` | HTTP→AMI trigger, cooldowns |

**Rule: keep every jitsi/* image on the SAME version tag** (`JITSI_IMAGE_VERSION`).
Mixed versions break transcriber↔bridge behavior.

## Key custom files

```
soundboard.yml            # compose override: soundboard services + mounts
soundboard/               # JS/CSS UI, REST service, Asterisk conf, WAV clips
sip-mock.yml + mock-sip/  # Asterisk mock PBX + dialplans (incl. soundboard contexts)
skynet.yml                # transcriber + skynet (CPU defaults)
skynet.gpu.yml            # GPU override (cuda/float16/small.en) — layer LAST
resources/skynet/         # skynet source + Dockerfile (chown patched to 1001:1001)
aws.override.yml          # AWS-only: SIP-jigasi transcriptions off, host-gateway route
DEPLOY-AWS.md             # full AWS deployment guide + troubleshooting table
CUSTOM-SETUP.md           # local-setup history and detailed explanations
```

## Setup in 8 steps (condensed — full detail in DEPLOY-AWS.md)

1. **Host**: any Docker host. For GPU transcription: AWS `g4dn.xlarge` (T4), **40 GB disk**,
   NVIDIA driver 535 + nvidia-container-toolkit. Open **TCP 8443** and **UDP 10000**.
2. **Clone** the repo, `cd docker-jitsi-meet-stable-9646`.
3. **Env**: `cp env.example .env && ./gen-passwords.sh`, then set:
   `PUBLIC_URL=https://<IP>:8443`, `JVB_ADVERTISE_IPS=<IP>`, `HTTP_PORT=8000`,
   `HTTPS_PORT=8443`, `JITSI_IMAGE_VERSION=stable-9753`, `ENABLE_TRANSCRIPTIONS=1`,
   `JIGASI_SIP_URI=transcriber@meet.jitsi` (any non-empty value — it makes jicofo
   create the brewery), `JIGASI_TRANSCRIBER_PASSWORD=<random>`, and the Whisper JWT key:
   ```bash
   JIGASI_TRANSCRIBER_WHISPER_PRIVATE_KEY_NAME=jitsi-key
   JIGASI_TRANSCRIBER_WHISPER_PRIVATE_KEY=$(openssl genrsa 2048 | openssl pkcs8 -topk8 -nocrypt -outform DER | base64 -w 0)
   ```
   ⚠️ Key MUST be PKCS#8-DER→base64 (~1625 chars). base64 of the PEM fails (`invalid key format`).
4. **Build skynet**: `docker build --build-arg BUILD_WITH_VLLM=0 -t skynet-gpu:local -f resources/skynet/Dockerfile resources/skynet/`
   (on non-GPU hosts add `--build-arg BASE_IMAGE_BUILD=ubuntu:22.04` and use CPU env).
5. **Up**: `mkdir -p ~/.jitsi-meet-cfg/{web,transcripts,prosody/config,prosody/prosody-plugins-custom,jicofo,jvb,jigasi}` then
   `docker-compose -f docker-compose.yml -f jigasi.yml -f skynet.yml -f skynet.gpu.yml -f aws.override.yml up -d`
   (add `-f sip-mock.yml -f soundboard.yml` for SIP + soundboard).
6. **Post-start (order matters!)**:
   ```bash
   docker run --rm -v docker-jitsi-meet-stable-9646_skynet-models:/models alpine chown -R 1001:1001 /models
   docker restart docker-jitsi-meet-stable-9646_skynet_1     # MUST restart after chown
   docker exec docker-jitsi-meet-stable-9646_prosody_1 prosodyctl --config /config/prosody.cfg.lua \
     register transcriber auth.meet.jitsi <JIGASI_TRANSCRIBER_PASSWORD>
   docker restart docker-jitsi-meet-stable-9646_jigasi-transcriber_1
   ```
7. **Verify skynet**: log must show `WHISPER MODEL INFO` with **no PermissionError** —
   otherwise subtitles silently never work while everything "looks Up".
8. **Test**: `https://<IP>:8443` → join room → ⋯ menu → Start Subtitles → speak.
   Health check: `docker logs <skynet> | grep Closed` → want `Audio: >0s`.

## Top gotchas (full table in DEPLOY-AWS.md)

- **skynet model volume is root-owned; skynet runs as uid 1001** → crashes loading the
  model while the container shows "Up". chown + restart skynet (step 6). This was the
  single biggest time sink of the whole project.
- **Whisper JWT key encoding** — PKCS#8 DER→base64, not PEM→base64.
- **Prosody `transcriber` user is never auto-created** — register it manually.
- **docker-compose 1.29.2 `KeyError: 'ContainerConfig'`** — never recreate in place;
  `down` then `up` (or `docker rm -f` the container first).
- **SIP-mode jigasi crashes if it inherits `ENABLE_TRANSCRIPTIONS=1`** — override to 0.
- **Template-generated configs** (nginx meet.conf, jicofo.conf) — patch the template or
  use custom-config hooks; edits to rendered files vanish on restart. jicofo only
  regenerates its config on fresh container creation.
- **P2P**: 1:1 browser calls bypass JVB, so the transcriber hears nothing. Disable with
  `config.p2p.enabled = false;` in `~/.jitsi-meet-cfg/web/custom-config.js` if testing
  transcription with two participants.

## Status

- **Working**: core meetings (multi-party AV), SIP gateway, soundboard (tested locally),
  skynet GPU Whisper service, transcriber dispatch/auth/skynet connection.
- **Open (AWS only)**: JVB does not forward participant audio RTP to the transcriber
  (skynet sessions close `Audio: 0s`). Verified NOT caused by: skynet build/permissions,
  Whisper key, ssrc-rewriting, colibri-ws hairpin (this jigasi never dials colibri-ws).
  Prime suspects: P2P during 2-person tests; jicofo→JVB source signaling for the
  transcriber endpoint. Diagnostics: tcpdump inside the transcriber container +
  `curl 127.0.0.1:8080/debug?full=true` (JVB per-endpoint state).
```
