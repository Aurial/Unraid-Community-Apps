# Unraid Community Apps templates by Aurial

Two templates live in this repository: **Strata AMD** (the AMD/ROCm builds of Strata, a local LLM) and
**Phonon TTS** (the official speech server for Gradium's Phonon text-to-speech models).

## Strata AMD

An Unraid Community Applications template for the **AMD (ROCm) builds** of [Strata](https://github.com/Niko1221/Strata),
which runs the Qwen3.8-Flash-Next mixture-of-experts model on an AMD GPU plus system RAM. It serves an OpenAI- and
Anthropic-compatible API, a chat page and a live monitor.

The template runs Strata's official image, `ghcr.io/niko1221/strata:latest-rocm-gfx120x`. Report template or Unraid
problems in this repository's [issues](https://github.com/Aurial/strata-unraid-amd/issues), and problems with
Strata itself to the [Strata project](https://github.com/Niko1221/Strata/issues).

## Why a separate listing

The Community Apps catalog already lists **Strata**, an NVIDIA-only template by
[williamkunz](https://github.com/williamkunz/strata-docker) (CUDA images, Nvidia Driver plugin, `--runtime=nvidia`).
That template cannot run on AMD cards. This listing covers the AMD side of the same upstream project: different image
flavors (`rocm-gfx120x` / `rocm-gfx110x` / `rocm-gfx1151`), different host requirements (the `amdgpu` kernel driver
Unraid already ships — no driver plugin), and device passthrough via `/dev/kfd` and `/dev/dri` instead of the NVIDIA
runtime. Both templates use Strata's official image and expose the same settings.

## Install

Search for **Strata AMD** in the Apps tab. Before installing you need:

- an AMD card from one of the families Strata publishes images for. At install, pick your card under **Branch** —
  the template offers all three ROCm flavors as a dropdown:

  | Branch shown at install | Card |
  | --- | --- |
  | `latest-rocm-gfx120x` (default) | RX 9000 series, Radeon AI PRO R9700 (RDNA 4) |
  | `latest-rocm-gfx110x` | RX 7000 series (RDNA 3) |
  | `latest-rocm-gfx1151` | Ryzen AI Max "Strix Halo" (Radeon 8060S) |

  The RX 6000 series has no image yet. The host needs only the `amdgpu` kernel driver (Linux 6.8+), which Unraid ships
  as standard — no driver plugin to install.
- 32 GB of RAM or more; more RAM allows a larger, better model ([Strata's model guide](https://github.com/Niko1221/Strata/blob/main/docs/MODELS.md))
- room for the model on an SSD pool, mapped as a direct pool path (`/mnt/cache/...`), not `/mnt/user`

Do not set a memory limit on the container: Strata plans from the server's whole RAM. Do not pin CPUs: the engine sets
its own thread affinity.

The first start downloads the model and prepares it. Later starts go straight to serving. The template's Overview
explains each setting, and Strata's [Docker guide](https://github.com/Niko1221/Strata/blob/main/docs/DOCKER.md) covers
them in full.

An API key is always on. If you leave it empty, the first start makes one; show it with:

```bash
docker exec "Strata AMD" strata launch --show-key --preset docker
```

(The container name contains a space, so quote it.)

## Multi-GPU

The **GPUs** field takes the card numbers as the monitor lists them (`0,1` splits the model across both by layer
parallelism), or `all` / empty for the engine's choice. Every named card must match the Branch family. Under Advanced
there is a **GPU roles** field for finer control (`1=main,0=helper`): *main* runs the final stage the answer comes
from, *helper* holds a share of the model's layers, *off* leaves a card alone; cards you do not name are off, and
Strata's automatic choice (faster card as main) is right for almost everyone. A split adds capacity more than
speed — see [Strata's multi-GPU guide](https://github.com/Niko1221/Strata/blob/main/docs/MULTI_GPU.md).

## Updates

The template follows `latest-rocm-gfx120x`, which Strata's maintainer moves to each new release when its image is
published; the same holds for the other two Branch tags. Unraid shows the update on the Docker tab for whichever
branch you installed. To stay on one release, change the Repository tag to a version such as `0.2.7-rocm-gfx120x`.

## Phonon TTS

An Unraid Community Applications template for [ptts-openai-server](https://github.com/gradium-ai/xn-ptts/tree/main/ptts-openai-server),
the official speech server for **Phonon**, [Gradium](https://gradium.ai/phonon)'s open-weight on-device text-to-speech
models. It serves an **OpenAI-compatible API** (`POST /v1/audio/speech`) and the **Wyoming protocol**, so it plugs into
Open WebUI, SillyTavern, Home Assistant and anything else with an "OpenAI TTS" setting — fully local, CPU-only, and
fast enough to speak faster than real time even on a Raspberry Pi.

Search for **Phonon TTS** in the Apps tab; the defaults work out of the box: API port `8880`, language `en`, and a
model/voices cache (under about 1 GB) downloaded on first start and kept across restarts. The first start logs little
until the model loads — watch the log. For Home Assistant, set **Wyoming address** to `0.0.0.0:10200`, map the
**Wyoming port**, then add the **Wyoming Protocol** integration pointing at this server. The template's Overview
explains every setting, including other models (`hf://owner/name`), gated 40M checkpoints, voice aliases and OpenAI
voice-name compatibility.

> **The server has no authentication on any port.** Run it on a trusted network; never expose it to the internet.

The template follows `latest`, which Gradium moves with each release; pick a pinned tag (such as `0.5.0`) under
**Branch** at install to stay on one release.

## Credits and licence

[Strata](https://github.com/Niko1221/Strata) and its images are by Niko1221 and the Strata contributors, under the MIT
licence. The Strata icon is Strata's own. The Strata template follows the structure of the NVIDIA [strata-docker](https://github.com/williamkunz/strata-docker)
template by William Kunz (MIT).

[Phonon](https://gradium.ai/phonon) and the [xn-ptts](https://github.com/gradium-ai/xn-ptts) server are by
[Gradium](https://gradium.ai) and contributors, Apache-2.0; the models themselves carry their own licences on
Hugging Face (the 31M models are Apache-2.0; the 40M models are gated — check their model cards). The Phonon icon in
this repository is an original work by the template maintainer, MIT.

This repository is MIT licensed; see [LICENSE](LICENSE).
