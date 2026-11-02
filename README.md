# Unraid Community Apps templates by Aurial

Two templates live in this repository: **Strata AMD** (the AMD/ROCm builds of Strata, a local LLM server) and **Phonon TTS** (the official speech server for Gradium's Phonon text-to-speech models).

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

## Phonon TTS

An Unraid Community Applications template for [ptts-openai-server](https://github.com/gradium-ai/xn-ptts/tree/main/ptts-openai-server), the official speech server for **Phonon**, [Gradium](https://gradium.ai/phonon)'s open-weight on-device text-to-speech models. It serves an **OpenAI-compatible API** (`POST /v1/audio/speech`) and the **Wyoming protocol**, so it plugs into Open WebUI, SillyTavern, Home Assistant and anything else with an "OpenAI TTS" setting. It runs fully local, on the CPU only, and fast enough to speak faster than real time even on a Raspberry Pi.

Search for **Phonon TTS** in the Apps tab; the defaults work out of the box: API port `8880`, language `en`, and a model/voices cache (under about 1 GB) downloaded on first start and kept across restarts. The first start logs little until the model loads, so watch the log. For Home Assistant, set **Wyoming address** to `0.0.0.0:10200`, map the **Wyoming port**, then add the **Wyoming Protocol** integration pointing at this server. The template's Overview explains every setting, including other models (`hf://owner/name`), gated 40M checkpoints, voice aliases and OpenAI voice-name compatibility.

> **The server has no authentication on any port.** Run it on a trusted network; never expose it to the internet.

The template follows `latest`, which Gradium moves with each release; pick a pinned tag (such as `0.5.0`) under **Branch** at install to stay on one release.

---

## Credits and licence

[Strata](https://github.com/Niko1221/Strata) and its images are by Niko1221 and the Strata contributors, under the MIT licence. The Strata icon is Strata's own. The Strata template follows the structure of the NVIDIA [strata-docker](https://github.com/williamkunz/strata-docker) template by William Kunz (MIT). Quality figures in the model table come from Strata's `docs/QUANTS.md`.

[Phonon](https://gradium.ai/phonon) and the [xn-ptts](https://github.com/gradium-ai/xn-ptts) server are by [Gradium](https://gradium.ai) and contributors, Apache-2.0; the models themselves carry their own licences on Hugging Face (the 31M models are Apache-2.0; the 40M models are gated, so check their model cards). The Phonon icon in this repository is an original work by the template maintainer, MIT.

This repository is MIT licensed; see [LICENSE](LICENSE).