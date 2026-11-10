# Unraid Community Apps templates by Aurial

Three templates live in this repository: **Strata AMD** (the AMD/ROCm builds of Strata, a local LLM server), **Radiance-LLM** (radiance, a modular AMD inference engine serving language, image, video, speech and music models) and **Phonon TTS** (the official speech server for Gradium's Phonon text-to-speech models).

---

## Strata AMD

An Unraid Community Applications template for the **AMD (ROCm) builds** of [Strata](https://github.com/Niko1221/Strata). Strata runs the Qwen3.8-Flash-Next mixture-of-experts model on an AMD GPU plus system RAM and serves an OpenAI- and Anthropic-compatible API, a chat page and a live monitor.

The template runs Strata's official image, `ghcr.io/niko1221/strata:<version>-rocm-<flavor>`. Report template or Unraid problems in this repository's [issues](https://github.com/Aurial/Unraid-Community-Apps/issues), and problems with Strata itself to the [Strata project](https://github.com/Niko1221/Strata/issues).

> **About the numbers on this page.** Everything marked *measured* was measured on one server: two Radeon AI PRO R9700 (32 GB each), an Intel i5-13600K, 94 GiB DDR5-4800, an XFS NVMe cache pool, Unraid, with about 60 other containers running. Your numbers will differ. Strata's own quality figures are credited to Strata. Only the `rocm-gfx120x` image has been run by the template author; the `gfx110x` and `gfx1151` branches are Strata's images and are untested here.

### Why a separate listing

The Community Apps catalog already lists **Strata**, an NVIDIA-only template by [williamkunz](https://github.com/williamkunz/strata-docker) (CUDA images, Nvidia Driver plugin, `--runtime=nvidia`). That template cannot run on AMD cards. This listing covers the AMD side of the same project: different image flavors, the `amdgpu` kernel driver Unraid already ships (no driver plugin), and device passthrough via `/dev/kfd` and `/dev/dri`. Both use Strata's official image and expose the same settings.

---

### Before you install

1. **An AMD card from a supported family.** Pick it under **Branch** at install:

   | Branch | Card |
   |---|---|
   | `latest-rocm-gfx120x` (default) | RX 9000 series, Radeon AI PRO R9700 (RDNA 4) |
   | `latest-rocm-gfx110x` | RX 7000 series (RDNA 3) |
   | `latest-rocm-gfx1151` | Ryzen AI Max "Strix Halo" (Radeon 8060S) |

   The RX 6000 series has no image. The host needs only the `amdgpu` kernel driver (Linux 6.8+), which Unraid ships.
2. **Enough RAM, counted after your other containers.** Strata plans from the server's *total* RAM and cannot see what your other containers use. See [RAM](#ram-the-setting-that-matters-most) before you press Apply.
3. **Room on an SSD pool** for the model files (table below), mapped as a **direct pool path** (`/mnt/cache/...` or `/mnt/POOLNAME/...`), never `/mnt/user`. The engine reads the model files while it runs.
4. **Nothing else using the GPU memory.** Strata fills the cards' memory on purpose. Stop other GPU containers (other LLM servers, transcoders) first.
5. **No memory limit and no CPU pinning** on the container. Strata sizes itself from the whole machine and sets its own thread affinity.

### Install

Apps tab, search **Strata AMD**, click Install. The fields that matter:

| Field | What to enter | Notes |
|---|---|---|
| Branch / Repository | your card family | The default tags (`latest-rocm-*`) move when the maintainer publishes. Once your setup is tuned, pin a version, see [Updating](#updating-and-pinning). |
| WebUI / API port | `8080` or another free port | 8080 is often already in use on Unraid; the host port is the only thing to change. |
| Data | `/mnt/POOLNAME/appdata/strata-amd` | Model, prepared pack, presets, tuning and the API key. **Keep this path stable**: a new, empty folder starts a full download again. |
| GPUs | `0` for one card, `0,1` for two | Name your cards on a multi-GPU server. See [Multi-GPU](#multi-gpu). |
| Model | for example `IQ3_XXS` or `IQ3_S` | Empty lets Strata choose. See [Choosing a model](#choosing-a-model). |
| Model family | `qwen` for the original packs | Empty lets the installer choose, which may pick a different family and download different files. |
| Context length | `256k` for agent clients, `128k` for chat | The client's own configured window must not be larger than this. |
| Vision | `on` if you need pictures | About 0.9 GB more to download. |
| API key | `openssl rand -hex 24`, or empty | Empty: the first start makes one and keeps it in the Data folder. |
| Parallel requests | `2` on a busy server | Each extra slot costs RAM. Empty: fitted to the card. |
| Engine settings | see [RAM](#ram-the-setting-that-matters-most) | The most important advanced field on a shared server. |

### First start

The first start downloads the model and prepares it. Watch it with `docker logs -f Strata-AMD`. A stopped download resumes where it left off, and later starts go straight to loading. Downloads are the model (table below) plus a draft layer of about 5 GB that makes answers about twice as fast, plus the 0.9 GB picture reader if Vision is on. Loading takes from under a minute to about 75 seconds. The container shows `healthy` in `docker ps` once the model answers.

When the log ends with `Strata is serving.`, open `http://SERVER-IP:PORT/` for the chat page and monitor.

Note: `docker logs` replays everything the container ever printed, including earlier starts and stops, so a long log full of "Stop requested" and "Starting Strata" blocks is history, not a restart loop. Per-request lines are in the engine log (see [Logs](#logs-monitoring-and-stopping)).

---

### RAM: the setting that matters most

By default Strata copies the model's experts into system RAM as well as keeping a copy on the GPU, even experts the cards already hold. On a dedicated machine that is fine. On a shared Unraid server it can use most of your memory and leave nothing for the other containers; with no swap, the kernel's out-of-memory killer then picks a victim.

**Measured, two R9700 (64 GB VRAM), IQ3_XXS:**

| Setting | Container RAM | RAM available to the server | Cold 40,000-token prompt |
|---|---|---|---|
| default (4 slots) | about 59 GiB | **4 GiB** | 11.2 s |
| `resident-budget-gib=8`, 2 slots | about 23 GiB | **41 GiB** | 12.2 s |

The cap is set in **Engine settings**:

```
resident-budget-gib=8
```

It limits the experts kept in RAM *that the GPUs do not hold*; the rest are read from the model files. How to choose it:

- **Two 32 GB cards:** the cards hold most of IQ3-sized models, so a budget of 8 to 12 worked well here (about 9% slower cold prompts at 8). Strata says the budget is a ceiling, not a target.
- **One card:** most experts live outside the GPU, so a very small budget sends many reads to the SSD, which Strata's own documentation says is much slower on a small GPU. Start with what your free RAM allows and raise it if decoding is slow.
- **Check it works** in the engine log: the line `expert cache hit rate` should stay high (99% here) and `blob reads from the file` should stay at 0. If reads from the file rise, raise the budget.

```
docker exec Strata-AMD tail -n 6 /data/state/strata/launch/docker.engine.log
free -h
docker stats --no-stream --format '{{.Name}} {{.MemUsage}}'
```

If `available` in `free -h` falls under about 10 GiB, stop the container and lower the budget or Parallel requests.

---

### Multi-GPU

With several cards Strata splits the model's layers across them ("layer split"). It does not need GPU-to-GPU links, so it works on boards where cards cannot talk to each other directly. A split adds **capacity** more than speed.

**Measured:** two R9700s read long prompts faster than one card did (a cold 40,000-token prompt took 22.9 s on one card with an older engine and 10.7 to 12 s on two with newer ones; the engine version changed too, so treat that as indicative). Decode speed did not improve, as expected for a model that already fits one card.

- Enter the cards explicitly in **GPUs** (`0,1`). Every named card must match the Branch family.
- **A caution about `all`:** on the two-card test server, one benchmark right after install ran at about half the normal speed (prose decode 38 tok/s, cold prompt 24 s) with GPUs set to `all`; after switching to `0,1` and restarting, it returned to normal (about 90 tok/s, 10.7 s). The restart happened at the same time, so the cause was not isolated. Naming the cards costs nothing.
- **GPU roles** (Advanced) gives finer control, for example `1=main,0=helper`. Strata's automatic choice is right for almost everyone; the field cannot be used together with GPUs.

---

### Context length

Longer context costs RAM and VRAM. `128k` suits chat. Agent clients that keep long tool histories (for example Hermes) want `256k`. **The client's configured window must not exceed the value here**: a client that believes it has 262,144 tokens against a container set to 128k fails near 128,000 tokens. Check what the server really has:

```
curl -s -H "Authorization: Bearer YOUR-KEY" http://SERVER-IP:PORT/v1/models
```

`meta.n_ctx` is the window (`262144` for `256k`).

### Vision

`Vision = on` downloads a 0.9 GB picture reader. On cards of 24 GB or more it runs on the GPU. **Measured:** text speed was unchanged (decode about 88 tok/s with and without; cold prompt 12.4 s against 12.2 s), and VRAM use rose to within about 2 GB of full on each 32 GB card, so a very large image is the place to watch on a tight setup. After install, `/v1/models` lists `image` among `input_modalities`.

### Using model files you already have

Point **Existing model folder** at a host folder that holds one model's GGUF shards **directly** (not a folder of folders), mounted read-only at `/models`. Then set **Existing model files** to `/models`, and set **Model** and **Model family** to match the files (for example `IQ3_S` and `qwen`). Nothing is downloaded for the model itself; the prepared pack is made in the Data folder. Leave the folder empty if you do not use this, and no mount is created.

---

### Choosing a model

Download sizes are from Strata's documentation. Quality columns are Strata's own measurement against the Q8_0 reference (about 5,000 text positions, on an AMD box); the last two columns are ours.

| Model | Download | Same top token as Q8_0 | Mean KL (lower is closer) | Our GSM8K (300 questions) | Our tool-picking test |
|---|---|---|---|---|---|
| `IQ3_XXS` | 76 GB | 84.5% | 0.202 | 291/300 (97.0%) | 94 of 100 |
| `IQ3_S` | 84 GB | 87.7% | 0.144 | 293/300 (97.7%) | 88 to 93 of 100 |
| `UD-IQ4_XS` (family `unsloth`) | 94 GB | 88.9% | 0.101 | not tested | not tested |
| `UD-Q4_K_XL` (family `unsloth`) | 111 GB | 93.1% | 0.056 | not tested | not tested |

- **Disk space** needed in the Data folder: about 80 GB for IQ3_XXS and about 90 GB for IQ3_S (model, a 6 GB draft layer, a small pack).
- **RAM needed** is the model's experts plus about 10 GB (Strata's figure): roughly 53 GB for IQ3_XXS and 60 GB for IQ3_S in total, before you cap it as above.
- **GSM8K and the tool-picking test did not separate IQ3_XXS from IQ3_S.** A two-question gap on 300 is noise. Both models made the same mistakes on terse prompts such as "Finish task 9" (choosing the wrong tool). These tests do not measure long-conversation quality, so they neither confirm nor refute the usual advice that a larger quant is better.
- **The 4-bit files** are a much bigger step: their experts are 55 to 72 GiB, so they need an explicit RAM budget and are likely slower on 64 GB of VRAM. The plan screen accepts an R9700 for them; the template author has not run them.

### Performance reference (measured)

IQ3_S, two R9700, Strata 0.2.9 (engine build 0.1.55), `resident-budget-gib=12`, 2 slots, 256k context, vision on:

| Test | Result |
|---|---|
| Cold 40,818-token prompt | 10.7 s |
| Same prompt with a new ending (prefix cached) | 3.8 s |
| Decode, prose (600 tokens) | about 87 to 94 tok/s |
| Decode, code | about 109 tok/s |
| GSM8K, 300 questions (temperature 0, thinking off) | 293/300 (97.7%) on this build, the same as on the older 0.1.52 build |
| Tool-picking test, 3 runs of 25 | 70 of 75 (93%) |

Earlier measurements, IQ3_XXS on engine 0.1.48 (older), for scale: a cold 122,000-token prompt took 29 s, and a 99% expert-cache hit rate.

---

### Connecting clients

- **OpenAI-compatible apps:** base URL `http://SERVER-IP:PORT/v1`, API key = your key, model name from `GET /v1/models`.
- **Anthropic-compatible apps** (such as Claude Code): base URL `http://SERVER-IP:PORT` (no `/v1`), same key.
- Every request must send `Authorization: Bearer KEY`; `GET /health` answers without one, `GET /v1/models` does not.
- In our testing the server also answered requests that used a different model name than the one it reports; do not rely on that. Use the name from `/v1/models`.
- `docker exec Strata-AMD strata connect opencode` writes the settings file for several coding apps and prints it.

### The API key

The key is always on. Leave the field empty and the first start makes one and keeps it in the Data folder. Show it:

```
docker exec Strata-AMD strata launch --show-key --preset docker
```

The startup log prints the **first ten characters** of the key. Strip that line before you paste a log anywhere public. Do not expose the port to the internet without a reverse proxy and your own authentication.

---

### Updating and pinning

The default tags (`latest-rocm-*`) follow the maintainer, who moves them to each release once its image is published. Not every release gets an image: at the time of writing there are images for `0.2.1`, `0.2.3`, `0.2.7` and newer, and none for `0.2.4` to `0.2.6`. Check what exists before you pin:

```
for t in 0.2.7-rocm-gfx120x 0.2.9-rocm-gfx120x; do printf "%s: " $t; docker manifest inspect ghcr.io/niko1221/strata:$t >/dev/null 2>&1 && echo exists || echo missing; done
```

To stay on one release, edit **Repository** to `ghcr.io/niko1221/strata:0.2.7-rocm-gfx120x` (your tag) and Apply. Your models, pack and settings stay in the Data folder and nothing is downloaded again. After an engine update, rerun a quick check (a long prompt and a short chat) before trusting it, because speed and memory use can change between engine builds.

### Moving from a hand-run container or an older template

- Point **Data** at the same folder. The first start recognises the model and says `already installed with these choices`.
- A data folder from the older 0.1.x image is migrated automatically on the first start (a rename inside the folder; nothing is copied or downloaded).
- Stop the old container first: two containers cannot share the GPUs or the port.
- Unraid keeps a saved copy of each template in `/boot/config/plugins/dockerMan/templates-user/`. If you tested an older version of this template, move that file aside before installing, or the Add Container dropdown may offer the stale copy.

### Logs, monitoring and stopping

```
docker logs -f Strata-AMD                         # startup (and, in full, all earlier starts)
docker exec Strata-AMD tail -f /data/state/strata/launch/docker.engine.log    # per-request lines, cache hit rate
docker exec -it Strata-AMD strata monitor         # live monitor (q leaves it)
docker exec Strata-AMD strata --version
curl -s -o /dev/null -w "%{http_code}\n" http://SERVER-IP:PORT/health         # 200 when ready
```

The template sets `--stop-timeout 120` so a large model has two minutes to unload. Docker's default of 10 seconds can kill the container mid-unload (exit code 137). Nothing is lost, but the next start takes longer.

---

### Troubleshooting

| Symptom | Likely cause | What to do |
|---|---|---|
| The server's free RAM collapses after install | The default copies experts into RAM too | Set Engine settings to `resident-budget-gib=8` (two cards) and Parallel requests to `2`, Apply. |
| It downloads the whole model again after you edit the template | The Data path changed to a new empty folder | Put Data back, or copy the old folder across. |
| Much slower than expected on a multi-GPU server | Cards not named, or the server restarted cold | Set GPUs to `0,1`, restart, and compare again. Check VRAM on both cards (`/sys/class/drm/card*/device/mem_info_vram_used`). |
| `401 Unauthorized` from a client | The key is missing or different | Send `Authorization: Bearer KEY`; show the key with the command above. |
| A client fails near 128,000 tokens | Container context is `128k`, client assumes more | Set Context length to `256k` and make sure the client's window matches. |
| `docker logs` looks like a restart loop | It replays all history | Read the end of the log, or the engine log. |
| Exit code 137 on stop | Docker killed it mid-unload | The `--stop-timeout 120` option is in Extra Parameters; do not remove it. |
| Pictures are ignored | Vision is `off` | Set Vision to `on` and check `input_modalities` in `/v1/models`. |
| A pinned tag is not found | That release has no published image | Run the manifest check above and choose a tag that exists. |
| `docker exec Strata AMD` fails | Docker container names cannot contain a space | The container is named `Strata-AMD`. |
| Container will not start: port in use | Another app uses the host port | Change **WebUI / API port**. |

---

## Radiance-LLM

An Unraid Community Applications template for **radiance**, a modular inference engine for AMD GPUs by [StillDeadcode](https://codeberg.org/StillDeadcode/radiance) (Apache-2.0). One server, one card or several, speaks the **OpenAI** (chat, completions, Responses, images, audio, videos), **Anthropic Messages** and **llama-server** APIs, so their clients and SDKs work unchanged — and it does more than chat: language models plus image, video, speech and music generation, with a live dashboard at the root URL.

The template runs the official image, `stilldeadcode/radiance`. Report template or Unraid problems in this repository's [issues](https://github.com/Aurial/Unraid-Community-Apps/issues), and problems with radiance itself to the [radiance project](https://codeberg.org/StillDeadcode/radiance/issues).

> **About the numbers on this page.** Everything marked *measured* was measured on one server: two Radeon AI PRO R9700 (32 GB each), an Intel i5-13600K, 94 GiB DDR5-4800, an XFS NVMe cache pool, Unraid, with about 60 other containers running. Your numbers will differ. Figures credited to radiance come from its own docs and compose files. Only the RDNA4 (gfx1201) target has been run by the template author; the RDNA3 and Strix Halo kernels ship in the same image but are untested here.

### How this differs from Strata AMD

Both templates on this page run large Qwen models on AMD cards and both serve OpenAI-compatible APIs, but they are different engines with different model formats and cannot share files. **Strata** streams a mixture-of-experts model from system RAM (GGUF packs, ~80 GB download, tens of GB of RAM budget to tune) and adds a chat page and monitor. **Radiance** uses its own `.rad` model packs, keeps the model on the cards (host memory is a tier, not the primary store), is a smaller and faster-starting engine built for AMD, and covers image/video/speech/music generation as well as text. Pick by model and features, not by name; several people run both.

---

### Before you install

1. **A supported card.** The image's bundled kernels cover **RDNA 4** (RX 9000 series, Radeon AI PRO R9700), **RDNA 3** (RX 7000 series) and **Ryzen AI Max** (Strix Halo). The engine starts on any AMD family, but another card needs a kernel plugin ([radiance docs](https://codeberg.org/StillDeadcode/radiance/src/tag/v1.17.0/docs/PLUGIN.md)). The host needs only the `amdgpu` kernel driver Unraid ships — no ROCm, no compiler, no driver plugin.
2. **Resizable BAR** (Above 4G Decoding) on in the firmware. Without it the engine keeps kernel arguments in pinned host memory and reads them over the PCIe link on every dispatch.
3. **Room on an SSD pool for the model packs** (table below), as a **direct pool path** (`/mnt/cache/...` or `/mnt/POOLNAME/...`), never `/mnt/user`: the engine reads models by direct I/O while serving, and Qwen3.8-Flash-Next streams its n-gram table from the folder.
4. **Enough RAM for one-card Flash-Next** — that profile pins about 49 GiB of host memory (upstream's figure; it expects a 64 GB host). Every other single-card model on this page keeps its weights on the card and needs little beyond headroom.
5. **One model at a time.** The engine fills the cards on purpose. Stop other GPU containers (transcoders, other LLM servers) first.
6. **An API key ready** — see [The API key](#the-api-key-never-auto-generated). This engine refuses to start without one.

### Install

Apps tab, search **Radiance-LLM**, click Install. The fields that matter:

| Field | What to enter | Notes |
|---|---|---|
| Branch / Repository | `1.17.0` (default) or `latest` | Pin a version once tuned; see [Updating](#updating-and-pinning-1). |
| WebUI / API port | `8180` or another free port | The engine listens on 8000 inside the container; this is the host port. 8080 and 8000 are commonly taken on Unraid. |
| Models | `/mnt/cache/appdata/radiance/models` | The folder you put the `.rad` packs in, mounted read-only at `/models`. |
| API key | `openssl rand -hex 24` | Required; the server will not start without it. |
| Post Arguments | see below | The whole engine command line lives here. The default is safe for one card. |

Download a model into the Models folder (from a shell):

```
wget -P /mnt/cache/appdata/radiance/models https://huggingface.co/StillDeadcode/qwen3.8-27b-mxfp4/resolve/main/qwen3.8-27b-mxfp4.rad
```

Then put its exact filename after `/models/` in **Post Arguments**. Advanced: to pin a one-card model to a specific card on a multi-card box, use the form's *Add another parameter (ENV)* mechanism with `RADIANCE_GPU=0` — the template deliberately ships no empty row for it.

### Choosing a model

Sizes are upstream's. All fit **one 32 GB card** unless noted; vision and generation models are served by the same engine.

| Model pack | Size | Notes |
|---|---|---|
| `qwen3.8-27b-mxfp4.rad` | 19 GiB | The single-card default: 27B dense, MXFP4, DFlash2 drafter, vision. |
| `qwen3.8-27b-iq2r-v4.rad` … `iq8r-v4.rad` | 10–28 GiB | Same model, pick a size that leaves room for your context. |
| `qwen3.8-27b-fp8.rad` | 29 GiB | Highest quality the dense line ships; tight on 32 GB with a big context. |
| `qwen3.6-35b-a3b-fp8.rad` / `ornith-1.5-35b-a3b-w4a8.rad` | 35 / 20 GiB | MoE, MTP; the w4a8 Ornith fits one card. |
| `qwen3.8-next-flash-fp8-iq4r-moe.rad` | 114 GiB | The flagship MoE. Two cards are its home profile; the one-card profile pins ~49 GiB of host RAM (upstream's numbers: ~2,545 tok/s prefill of 26K prompts at 4096-token steps, decode step 28.0 ms at one sequence). |
| Image / video / speech / music packs | 3–62 GiB | Flux 3 Klein 9.3 GiB, Krea 2 16 GiB, Breeze TTS 5.5 GiB, LTX-2.5 video 37 GiB and more — see the [upstream README table](https://codeberg.org/StillDeadcode/radiance#models). |

### Launch options (Post Arguments)

Everything after the image name goes to the engine. The template default is

```
--model /models/qwen3.8-27b-mxfp4.rad --api-key-env RADIANCE_API_KEY
```

which is one card, engine defaults everywhere else. What the useful flags do, with single-card advice first and two-card recommendations after:

| Flag | What it does |
|---|---|
| `--model PATH` | The `.rad` pack to serve. The only required flag. Path is inside the container, so your Models folder is `/models`. |
| `--api-key-env NAME` | Read the API key from environment variable `NAME`. Kept out of the command line, where every process on the host could read it. An empty or unset variable **refuses startup** — that is fail-closed working, not a bug. |
| `--host ADDR` / `--port N` | Listen address (default `0.0.0.0`) and port (default 8000). Keep the port equal to the container side of your port mapping; usually leave both alone. |
| `--tp N` | Tensor parallelism: how many cards split the model. One card: omit it (or `1`). Two cards: `--tp 2`. It adds capacity more than speed; decode barely changes when the model already fits one card. *Measured two R9700: prose decode 137–170 tok/s at `--tp 2` with Flash-Next.* |
| `--tp-wire W` | How the ranks exchange (two cards). `wht6` is the wide wire; upstream's production Flash-Next profile uses it. Harmless at `--tp 1`, so the template does not carry it. |
| `--max-num-seqs N` | Sequences (requests) decoded together. Each costs KV cache. One card with a big model: `2` to `4`. Two cards running Flash-Next: `4` to `8` (upstream production: 8). |
| `--max-model-len N` | The context window, in tokens. Agents that keep long tool histories want `200000` or `262144`; the client's own configured window must not exceed it. Context is carved from VRAM: lower it to fit more sequences. |
| `--kv-cache-dtype fp8` | Conversation memory at 8-bit float: roughly halves KV RAM for a quality change nobody notices in chat. Upstream ships it in every production profile. |
| `--placement expert_tiered` | MoE models only: experts the cards cannot hold go to a pinned host pool instead of being re-read from disk every token. Two cards: always worth it. One card with Flash-Next: keep it and set the pool below. |
| `--host-pool-mib N` | Size of that pinned host pool (MiB). Upstream: 12288 for two cards on Flash-Next; **43000** for one card (that is the ~49 GiB pin). Leave empty/absent on dense models that fit the card. |
| `--gpu-headroom-mib N` | VRAM to leave free per card. `96` is upstream's; leave it unless another process shares the card. |
| `--ngram-placement disk` | Flash-Next only: stream the n-gram embedding table from the model file (needs the fast pool and unconfined seccomp). `ram` would lock 47.7 GiB in RAM. |
| `--prefix-cache-mib N` | Finished conversations kept in host memory so repeat prompts skip prefill. Upstream: `4096`. *Measured: a 101,111-token prompt answered with 90,112 tokens already cached, time to first token 2.5 s.* |
| `--num-speculative-tokens N` | MTP drafter depth: predict N tokens per step and verify in parallel. `3` is upstream's production value; it is most of the reason decode is as fast as it is on models that ship a drafter. |
| `--max-num-batched-tokens N` | Prefill chunk per step. `2048` is the general default; upstream raises it to `4096` for one-card Flash-Next because prefill there is PCIe-link-bound and a bigger step amortises expert copies (2,545 vs 1,413 tok/s on 26K prompts, upstream's measurement). With 4096 steps also set `--checkpoint-interval 4096` so a prefill chunk never straddles a checkpoint. |
| `--served-model-name NAME` | The model id `/v1/models` advertises. Cosmetic; set it if a client insists on a particular name. |

**Two-card recommended (measured on the author's server, Qwen3.8-Flash-Next):**

```
--model /models/qwen3.8-next-flash-fp8-iq4r-moe.rad --api-key-env RADIANCE_API_KEY --tp 2 --tp-wire wht6 --max-num-seqs 4 --max-model-len 262144 --placement expert_tiered --host-pool-mib 12288 --gpu-headroom-mib 96 --kv-cache-dtype fp8 --prefix-cache-mib 4096 --ngram-placement disk --num-speculative-tokens 3 --max-num-batched-tokens 2048
```

This is the upstream production profile (`deploy/compose/flashnext.yaml`) with a 262144-token window and 4 slots instead of upstream's 200000/8, because the author's box serves one agentic client with a large window. Raise `--max-num-seqs` toward 8 when several clients share the server.

**Two-card recommended (dense 27B, most users' starting point):**

```
--model /models/qwen3.8-27b-mxfp4.rad --api-key-env RADIANCE_API_KEY --tp 2 --max-model-len 262144 --kv-cache-dtype fp8 --num-speculative-tokens 3
```

### The API key (never auto-generated)

Unlike some servers, radiance **never creates a key for you** and never writes one to disk. Every request except `/health`, `/ping` and the dashboard must carry `Authorization: Bearer KEY` (or `X-Api-Key: KEY`), and **the server refuses to start if the key variable is empty** — so a blank API key field is an immediate, deliberate failure, not a silent open server. Make a key:

```
openssl rand -hex 24
```

Without any key flag at all the server runs open on the LAN, with `/diagnostics` and the admin routes restricted to loopback clients. On a trusted network that is a choice; never expose an open server to the internet.

### First start, logs, stopping

Loading a 19 GiB pack takes about a minute on an NVMe pool; the flagship 114 GiB pack takes longer, so give the container a few minutes before judging. The log prints a startup report and ends with the server listening.

```
docker logs -f Radiance-LLM                               # startup report and per-request lines
curl -s -o /dev/null -w "%{http_code}\n" http://SERVER-IP:8180/health   # 200 when serving
curl -s -H "Authorization: Bearer KEY" http://SERVER-IP:8180/v1/models  # the served model name
curl -s -H "Authorization: Bearer KEY" http://SERVER-IP:8180/diagnostics | head -40   # the startup report as JSON
```

`GET /` opens the live dashboard (requests, cache, tokens per second). `/metrics` speaks Prometheus. The image is a `FROM scratch` rootfs: the only shell is busybox (`docker exec -it Radiance-LLM sh`), and there is no package manager by design.

On stop the engine finishes the step in flight and releases the cards; the template's `--stop-timeout 120` gives it room (Docker's 10 s default kills it mid-unload, exit 137).

### Performance reference (measured)

Author's server, Qwen3.8-Flash-Next on two R9700, engine 1.17.0, the two-card line above, serving live agent traffic (from the container's own request log over ~30 minutes):

| Test | Result |
|---|---|
| Decode, mixed tool-call traffic | about 137 to 170 tok/s (a short cached request peaked at 228) |
| Time to first token, warm prefix cache | 0.4 to 1.6 s |
| Cold 130,008-token prompt | 14.3 s total, TTFT 2.7 s, 144.5 tok/s |
| Prefix cache on a 101k-token agentic prompt | 90,112 of 101,111 tokens served from cache |

### Troubleshooting

| Symptom | Likely cause | What to do |
|---|---|---|
| Exits at once with a message about the api key | The API key field is empty; the engine fails closed by design | Put a key in the API key field (`openssl rand -hex 24`). |
| Exits with an unknown-option error mentioning an older version's flags | Branch moved, or a flag was mistyped | Check the flag list against the docs of **your** tag; pin `1.17.0`. |
| Model file not found | Filename in Post Arguments does not match the pack in the Models folder exactly | `ls` the folder; copy the exact name after `/models/`. |
| Flash-Next cannot serve / stalls reading the model | Mounted over `/mnt/user` (fuse), or Docker's default seccomp | Models must live on a direct pool path; keep `--security-opt seccomp=unconfined` in Extra Parameters. |
| `401 authentication_error` from a client | Missing or wrong key | Send `Authorization: Bearer KEY` (also accepted: `X-Api-Key`). |
| Client fails near a token count | The container's `--max-model-len` is smaller than the client's configured window | Raise the flag or lower the client's window; they must agree. |
| Port shows another app's UI | Host port already taken (UniFi takes 8080; Grafana 3000) | Change the host port. |
| Card hangs / engine refuses to serve at startup | Compute wave save/restore disabled (`amdgpu.cwsr_enable=0`) | It is the driver default on; if it was changed, restore it (or `--proceed-with-incompatible-configuration`, unsupported). |
| Another GPU container fails after installing this app | Radiance fills the cards on purpose | Run one LLM server at a time; `docker stop` the other. |

### Updating and pinning

`latest` follows each release. To stay put, keep the pinned tag under Branch. Releases move fast and occasionally rename flags, so after any engine update re-run a long prompt and a short chat before trusting it. Check a tag exists before pinning:

```
docker manifest inspect stilldeadcode/radiance:1.17.0 >/dev/null && echo exists
```

---

## Phonon TTS

An Unraid Community Applications template for [ptts-openai-server](https://github.com/gradium-ai/xn-ptts/tree/main/ptts-openai-server), the official speech server for **Phonon**, [Gradium](https://gradium.ai/phonon)'s open-weight on-device text-to-speech models. It serves an **OpenAI-compatible API** (`POST /v1/audio/speech`) and the **Wyoming protocol**, so it plugs into Open WebUI, SillyTavern, Home Assistant and anything else with an "OpenAI TTS" setting. It runs fully local, on the CPU only, and fast enough to speak faster than real time even on a Raspberry Pi.

Search for **Phonon TTS** in the Apps tab; the defaults work out of the box: API port `8880`, language `en`, and a model/voices cache (under about 1 GB) downloaded on first start and kept across restarts. The first start logs little until the model loads, so watch the log. For Home Assistant, set **Wyoming address** to `0.0.0.0:10200`, map the **Wyoming port**, then add the **Wyoming Protocol** integration pointing at this server. The template's Overview explains every setting, including other models (`hf://owner/name`), gated 40M checkpoints, voice aliases and OpenAI voice-name compatibility.

> **The server has no authentication on any port.** Run it on a trusted network; never expose it to the internet.

The template follows `latest`, which Gradium moves with each release; pick a pinned tag (such as `0.5.0`) under **Branch** at install to stay on one release.

### Phonon troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| The log ends with `Permission denied (os error 13)` while fetching `voices.jsonl` (or the model), and the container exits | The image runs as its own user (uid 1000) and cannot write to the cache folder Unraid created as root. The message suggests a gated repo; here it is a file-permission problem. | Set **Extra Parameters** to `-u 0:0`, or run `chown -R 1000:1000` on the cache folder. Check with `docker inspect Phonon-TTS --format 'user={{.Config.User}}'` (it should print `0:0` for the first fix). |
| The container exits right after loading the model with `no voice catalog at ...` | A voice catalog set to a bare repo name (`gradium/phonon-voices`) is read as a local path, and an explicitly set catalog must load | Use `hf://gradium/phonon-voices` (it needs no token), or leave the catalog unset. An empty value starts fine but serves only the `default` voice. |
| `curl` on the API port returns an HTML "404 Not Found" page | Another container already uses that host port. Unraid's UniFi Network Application publishes `8880` and `8080` by default. | Change the host port in the template, then check with `docker port Phonon-TTS` and `ss -ltnp \| grep ':8880 '`. |
| Home Assistant cannot find the Wyoming server | The Wyoming address needs a port | Set `PTTS_WYOMING_ADDR` to `0.0.0.0:10200` (a bare `0.0.0.0` is not enough) and map port 10200. |

Checked on a working install: `GET /health` returns 200, `GET /v1/audio/voices` lists the voices (38 for English on the default model), and `POST /v1/audio/speech` returns MPEG audio (24 kHz, mono).

---

## Credits and licence

[Strata](https://github.com/Niko1221/Strata) and its images are by Niko1221 and the Strata contributors, under the MIT licence. The Strata icon is Strata's own. The Strata template follows the structure of the NVIDIA [strata-docker](https://github.com/williamkunz/strata-docker) template by William Kunz (MIT). Quality figures in the model table come from Strata's `docs/QUANTS.md`.

[Phonon](https://gradium.ai/phonon) and the [xn-ptts](https://github.com/gradium-ai/xn-ptts) server are by [Gradium](https://gradium.ai) and contributors, Apache-2.0; the models themselves carry their own licences on Hugging Face (the 31M models are Apache-2.0; the 40M models are gated, so check their model cards). The Phonon icon in this repository is an original work by the template maintainer, MIT.

[radiance](https://codeberg.org/StillDeadcode/radiance) is by [StillDeadcode](https://codeberg.org/StillDeadcode), Apache-2.0; the `stilldeadcode/radiance` image is the upstream's, and model packs are downloaded separately from Hugging Face under their own licences. The Radiance icon in this repository is an original work by the template maintainer, MIT (no official radiance icon is published).

This repository is MIT licensed; see [LICENSE](LICENSE).
