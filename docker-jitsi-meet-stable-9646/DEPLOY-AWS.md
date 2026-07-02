# Deploying Jitsi + GPU Transcription (skynet) + Soundboard to AWS

End-to-end guide for a **g4dn.xlarge** (NVIDIA T4) instance, from a blank Ubuntu box.
Every step here encodes a problem that bit us during the first deploy — follow the order.

> Placeholders: replace `PUBLIC_IP` with the instance's public IPv4 everywhere it appears.

---

## Phase 0 — Launch the EC2 instance

1. **Instance type:** `g4dn.xlarge` (4 vCPU, 16 GB RAM, 1× NVIDIA T4).
2. **AMI:** Ubuntu 22.04 LTS is fine. (The "Deep Learning" AMI ships drivers pre-installed, but a plain Ubuntu + the manual driver install in Phase 1 is predictable and what this guide assumes.)
3. **Disk:** set the root EBS volume to **40 GB** at launch. The default 8 GB is far too small — the skynet GPU image alone is ~17 GB. (If you forget: EC2 → Volumes → Modify → 40, then `sudo growpart /dev/nvme0n1 1 && sudo resize2fs /dev/root`.)
4. **Security group — inbound rules** (this is what makes the meeting reachable):

   | Type       | Protocol | Port  | Source    | Purpose              |
   |------------|----------|-------|-----------|----------------------|
   | Custom TCP | TCP      | 8443  | 0.0.0.0/0 | Jitsi web (HTTPS)    |
   | Custom UDP | UDP      | 10000 | 0.0.0.0/0 | JVB media (REQUIRED) |
   | SSH        | TCP      | 22    | your IP   | admin                |

   Without **10000/UDP** you'll see the meeting page but audio/video never connects.

---

## Phase 1 — GPU drivers, Docker, Compose

```bash
# NVIDIA driver (T4). Installs, then REBOOT.
sudo apt update
sudo apt install -y nvidia-driver-535-server nvidia-utils-535-server
sudo reboot
```

Reconnect after ~60 s, then verify the GPU is visible:

```bash
nvidia-smi          # must show a Tesla T4
```

Docker is usually preinstalled on modern Ubuntu AMIs; if `docker --version` fails, `sudo apt install -y docker.io`. Then the NVIDIA container runtime + Compose:

```bash
# NVIDIA Container Toolkit (lets Docker use the GPU)
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg
curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
  sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
  sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list
sudo apt update && sudo apt install -y nvidia-container-toolkit
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker

# docker-compose v1 (the classic `docker-compose` binary; `docker compose` plugin isn't on this image)
sudo apt install -y docker-compose
```

Verify Docker can see the GPU:

```bash
docker run --rm --gpus all ubuntu nvidia-smi     # must show the T4
docker-compose version                            # 1.29.x is fine
```

> **docker-compose 1.29.2 quirk:** it throws `KeyError: 'ContainerConfig'` when *recreating* a
> container whose image was built by BuildKit. Always do `down` then `up` (never rely on
> in-place recreate). This guide's commands already follow that pattern.

---

## Phase 2 — Get the code

```bash
git clone https://github.com/bhoumiksangle-cometchat/calls.git
cd calls/docker-jitsi-meet-stable-9646
```

The skynet source lives in `resources/skynet/` and the Dockerfile there is already patched
(uses numeric `--chown=1001:1001` so the `COPY` steps don't fail on the missing `jitsi` user).

---

## Phase 3 — Configure `.env`

```bash
cp env.example .env
./gen-passwords.sh          # fills in all *_PASSWORD / *_SECRET values
```

Now append the deployment-specific settings. **Set `PUBLIC_IP` first:**

```bash
PUBLIC_IP=$(curl -s ifconfig.me); echo "Public IP: $PUBLIC_IP"
```

Edit `.env` (`nano .env`) and set / add:

```ini
# --- Core networking ---
HTTP_PORT=8000
HTTPS_PORT=8443
PUBLIC_URL=https://PUBLIC_IP:8443
JVB_ADVERTISE_IPS=PUBLIC_IP

# --- Match ALL images to one version (mixed versions break the transcriber) ---
JITSI_IMAGE_VERSION=stable-9753

# --- Transcription (skynet) ---
ENABLE_TRANSCRIPTIONS=1
# Any non-empty value; this is what makes jicofo generate the jigasi brewery
# that the transcriber joins. It does NOT need to be a real SIP account.
JIGASI_SIP_URI=transcriber@meet.jitsi

# --- Colibri websocket bridge channel (modern default in 9753) ---
ENABLE_COLIBRI_WEBSOCKET=1
JVB_WS_SERVER_ID=jvb
```

Then add the transcriber's XMPP password and the Whisper signing key. **The key format is the
#1 gotcha** — jigasi wants base64 of the *PKCS#8 DER* bytes (~1624 chars), NOT base64 of the
PEM file (~2273 chars, double-wrapped → `invalid key format`):

```bash
# transcriber XMPP password
echo "JIGASI_TRANSCRIBER_PASSWORD=$(openssl rand -hex 16)" >> .env

# Whisper JWT signing key — CORRECT single-DER encoding
echo "JIGASI_TRANSCRIBER_WHISPER_PRIVATE_KEY_NAME=jitsi-key" >> .env
echo "JIGASI_TRANSCRIBER_WHISPER_PRIVATE_KEY=$(openssl genrsa 2048 2>/dev/null | openssl pkcs8 -topk8 -nocrypt -outform DER | base64 -w 0)" >> .env

# sanity check: length should be ~1625, and it must decode to a PRIVATE KEY
grep JIGASI_TRANSCRIBER_WHISPER_PRIVATE_KEY= .env | cut -d= -f2- | wc -c
grep JIGASI_TRANSCRIBER_WHISPER_PRIVATE_KEY= .env | cut -d= -f2- | base64 -d | tail -1   # -----END PRIVATE KEY-----
```

Create the config directories:

```bash
mkdir -p ~/.jitsi-meet-cfg/{web,transcripts,prosody/config,prosody/prosody-plugins-custom,jicofo,jvb,jigasi}
```

---

## Phase 4 — AWS override file

The SIP-mode jigasi must NOT inherit `ENABLE_TRANSCRIPTIONS=1` (it panics on missing Google
creds), and the transcriber can't reach the colibri-ws at the public IP from *inside* the box
(AWS won't hairpin to your own public IP) — so route that IP to the host gateway.

```bash
PUBLIC_IP=$(curl -s ifconfig.me)
cat > aws.override.yml <<EOF
services:
  jigasi:
    environment:
      - ENABLE_TRANSCRIPTIONS=0
  jigasi-transcriber:
    extra_hosts:
      - "${PUBLIC_IP}:host-gateway"
EOF
cat aws.override.yml
```

---

## Phase 5 — Build the skynet GPU image

Uses the CUDA base image (the T4 has real drivers now). ~20–40 min; the image is ~17 GB.

```bash
docker build \
  --build-arg BUILD_WITH_VLLM=0 \
  -t skynet-gpu:local \
  -f resources/skynet/Dockerfile \
  resources/skynet/

docker images | grep skynet-gpu     # confirm skynet-gpu:local exists
```

---

## Phase 6 — Bring up the stack

The full compose stack (base + SIP jigasi + skynet + GPU override + AWS override):

```bash
docker-compose \
  -f docker-compose.yml \
  -f jigasi.yml \
  -f skynet.yml \
  -f skynet.gpu.yml \
  -f aws.override.yml \
  up -d
```

> Save yourself the typing — stash the flag list:
> ```bash
> echo 'alias dc="docker-compose -f docker-compose.yml -f jigasi.yml -f skynet.yml -f skynet.gpu.yml -f aws.override.yml"' >> ~/.bashrc && source ~/.bashrc
> ```
> Then it's just `dc up -d`, `dc ps`, `dc logs jvb`, `dc down`, etc.

---

## Phase 7 — Post-start fixes (run once, after first `up`)

> ⚠️ **ORDER MATTERS — this was the #1 real blocker.** The `skynet-models` named volume
> is created **root-owned**, but skynet runs as uid **1001**. If skynet starts first, it
> **crashes trying to download the Whisper model** (`PermissionError: /models/streaming-whisper`)
> and never transcribes — the container still looks "Up", so it's easy to miss. The chown alone
> isn't enough once skynet has already crashed: you must chown **and then restart skynet** so it
> re-loads the model against the now-writable volume.

```bash
# 1. Fix the model-cache ownership, THEN restart skynet so it reloads the model cleanly
docker run --rm -v docker-jitsi-meet-stable-9646_skynet-models:/models alpine chown -R 1001:1001 /models
docker restart docker-jitsi-meet-stable-9646_skynet_1

# 2. Confirm the model actually loaded — you MUST see "WHISPER MODEL INFO" with NO PermissionError:
sleep 10
docker logs docker-jitsi-meet-stable-9646_skynet_1 2>&1 | grep -iE "WHISPER MODEL INFO|Model:|PermissionError|became self aware" | tail -6

# 3. Register the transcriber's Prosody account (Prosody doesn't auto-create it)
TRANSPW=$(grep '^JIGASI_TRANSCRIBER_PASSWORD=' .env | cut -d= -f2-)
docker exec docker-jitsi-meet-stable-9646_prosody_1 \
  prosodyctl --config /config/prosody.cfg.lua register transcriber auth.meet.jitsi "$TRANSPW"

# 4. Restart the transcriber so it authenticates with the now-existing account
#    AND reconnects to the now-healthy skynet
docker restart docker-jitsi-meet-stable-9646_jigasi-transcriber_1
```

> If step 2 shows `PermissionError`, the chown didn't take — re-run step 1. Do **not** proceed
> to testing until you see `WHISPER MODEL INFO` with no permission error, or subtitles will
> silently never appear no matter what else is correct.

Confirm everything is up and on `stable-9753`:

```bash
docker ps --format '{{.Names}}\t{{.Image}}\t{{.Status}}'
```

---

## Phase 8 — Test

1. Browse to `https://PUBLIC_IP:8443` → accept the self-signed cert → create a room.
2. Get a second participant in; confirm audio/video is smooth (validates JVB + 10000/UDP).
3. Click the **⋯ menu → Start Subtitles**, unmute, speak.

**Verify the transcription path:**

```bash
# jicofo must show the brewery (else no transcriber dispatch)
docker exec docker-jitsi-meet-stable-9646_jicofo_1 grep -i jigasi /config/jicofo.conf

# transcriber must auth WITHOUT "not-authorized" and join the control room
docker logs docker-jitsi-meet-stable-9646_jigasi-transcriber_1 2>&1 | grep -iE "not-authorized|Joined call control|Successfully connected"

# skynet must show a websocket open AND audio > 0s while you speak
docker logs docker-jitsi-meet-stable-9646_skynet_1 -f 2>&1 | grep -vi "ping\|pong"
```

---

## Troubleshooting cheat-sheet (the issues we actually hit)

| Symptom | Cause | Fix |
|---|---|---|
| `site can't be reached` | SG missing 8443/TCP | open it |
| Page loads, no audio/video | SG missing 10000/UDP | open it |
| `No space left on device` during build | 8 GB root disk | resize EBS to 40 GB, `growpart`+`resize2fs` |
| `could not select device driver [[gpu]]` | nvidia-container-toolkit not set up | Phase 1 toolkit steps + `systemctl restart docker` |
| `docker: unknown command: docker compose` | no compose plugin | `apt install docker-compose`, use `docker-compose` (dash) |
| **No subtitles, skynet looks "Up"** | **skynet crashed loading the model** (`PermissionError /models/streaming-whisper`) because it started before the volume was chowned | **chown the volume AND `docker restart` skynet** (Phase 7); verify `WHISPER MODEL INFO` prints with no PermissionError |
| skynet `PermissionError /models` | volume owned by root, skynet is uid 1001 | `chown -R 1001:1001` the models volume, then restart skynet |
| GPU whisper won't load (cuDNN/cuBLAS) | T4 lib mismatch in the GPU image | fallback: `WHISPER_DEVICE=cpu` + `WHISPER_COMPUTE_TYPE=int8`, drop `skynet.gpu.yml` |
| No Subtitles button | `ENABLE_TRANSCRIPTIONS` unset / brewery absent | set it + `JIGASI_SIP_URI`; regen jicofo config |
| transcriber `SASLError not-authorized` | Prosody has no `transcriber` user | `prosodyctl register transcriber auth.meet.jitsi <pw>` |
| transcriber `invalid key format` | key is base64 of the PEM (double-wrapped) | regenerate with `openssl pkcs8 -topk8 -outform DER \| base64` (~1625 chars) |
| `EndpointMessageTransport still not connected` | appears for browsers too; NOT fatal | ignore — not the transcription blocker |
| `KeyError: 'ContainerConfig'` | compose 1.29.2 recreate bug | `down` then `up`, or `docker rm -f <name>` then `up` |
| SIP jigasi crash-loops | inherited `ENABLE_TRANSCRIPTIONS=1` | `aws.override.yml` sets it to 0 for that service |

---

## Root cause of the "no subtitles" saga (resolved)

We burned a lot of time suspecting AWS networking (public-IP hairpin, colibri-ws routing,
JVB↔transcriber media). **None of that was the real cause.** Two red herrings to ignore:

- `EndpointMessageTransport still not connected` — appears for **browsers too**, and the meeting
  is fine, so it is **not** the transcription blocker.
- The jigasi transcriber in this build **does not use the colibri websocket at all** (confirmed:
  it only ever opens `ws://skynet:8000`). So colibri-ws / hairpin / `extra_hosts` routing is
  **irrelevant** to transcription. (The `extra_hosts` line is harmless; leave or drop it.)

**The actual blocker:** skynet crashed on startup with
`PermissionError: /models/streaming-whisper` because the model-cache volume was root-owned and
skynet runs as uid 1001. The Whisper model never loaded, so audio could never be transcribed —
but the container still reported "Up," which sent us chasing the network for hours. Fixed by
chowning the volume **and restarting skynet** (Phase 7). Confirm success by seeing
`====== WHISPER MODEL INFO ======` with **no** `PermissionError` in the skynet log.

> Also note: localhost used the **CPU** skynet image (`skynet-cpu`, int8); AWS uses the **GPU**
> image (`skynet-gpu`, cuda/float16). They are different code paths — "worked on localhost" never
> validated the GPU path. If the GPU model errors on cuDNN/cuBLAS, fall back to the CPU env vars
> (see troubleshooting table) to get subtitles working, then revisit the GPU build.
