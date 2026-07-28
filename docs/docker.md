# Running audio.cpp in Docker

## Table of Contents

- [Prerequisites](#prerequisites)
- [Image Variants](#image-variants)
- [Published Images](#published-images)
- [Build Images locally](#build-images-locally)
- [Usage](#usage)
- [Examples](#examples)

## Prerequisites

- Docker must be installed and running on your system.
- For CUDA:
  - The [NVIDIA container toolkit](https://github.com/NVIDIA/nvidia-container-toolkit) must be installed.

## Image Variants

The following image variants are available:

- **full**: Provides the main tools **cli** and **server** and test binaries in one image. When running the container, the first argument selects the tool to execute.

The following backends are supported:
- **cuda12**
- **cuda13**
- **cpu**

The following architectures are supported:
- **amd64**
- **arm64**

## Published Images

Docker images are published daily when new commits are available. The publishing
workflow can produce multiarch images (amd64/arm64), though it is not currently
configured to — see the note below.

Pull the latest images using these tags:
- **cuda12**: `ghcr.io/0xshug0/audio.cpp:full-cuda12`
- **cuda13**: `ghcr.io/0xshug0/audio.cpp:full-cuda13`
- **cpu**: `ghcr.io/0xshug0/audio.cpp:full-cpu`

> **Only `full-cuda13` (amd64) is currently published, and it carries device code
> for Blackwell only** (`sm_120` — RTX 50xx). The `cpu` and `cuda12` tags, and all
> arm64 variants, are disabled in `.github/workflows/docker.yml`; whatever is in
> the registry under those tags is stale and no longer refreshed. `full-cuda13`
> will not start on a pre-Blackwell card — expect `no kernel image is available
> for execution on the device`. On any other hardware, or for the other variants,
> build locally as shown below, or re-enable the matrix entries you need.

Images for a specific day/commit can be found in the
[versions](https://github.com/0xShug0/audio.cpp/pkgs/container/audio.cpp/versions?filters%5Bversion_type%5D=tagged)
history.
The format is: `full-<backend>-<date>-<shortsha>`, e.g. `full-cuda12-20260725-db7d2c4`


## Build Images locally

If you would like to build the images locally, you can use the available
Dockerfiles in `.devops`.

### CUDA

Build with the default CUDA 12.x version. See `.devops/cuda.Dockerfile`.

```bash
docker build -f .devops/cuda.Dockerfile -t local/audio.cpp:full-cuda12 .
```

Build with a specific CUDA version, for example 13.3.0:

```bash
docker build -f .devops/cuda.Dockerfile -t local/audio.cpp:full-cuda13 --build-arg CUDA_VERSION=13.3.0 .
```

Both commands build for every architecture the toolkit supports, from Turing to
Blackwell. To build only for your own GPU — a much shorter build, and a smaller
image — pass `CUDA_ARCHS`:

```bash
docker build -f .devops/cuda.Dockerfile -t local/audio.cpp:full-cuda12 \
  --build-arg CUDA_ARCHS=120a-real .   # 86-real = RTX 30xx, 89-real = RTX 40xx, 120a-real = RTX 50xx
```

A `-real`-only list embeds no PTX, so the image runs *only* on that architecture
and fails loudly anywhere else (`no kernel image is available for execution on
the device`). The default list is the quiet case instead: it includes virtual
architectures, so a card newer than anything compiled for will JIT and run —
but ggml selects kernels from the *compiled* list, so it silently uses an older
card's code paths and is merely slower. Either way, verify what shipped:

```bash
cuobjdump --list-elf /app/libggml-cuda.so | grep -oE 'sm_[0-9]+' | sort -u
```

### CPU

```bash
docker build -f .devops/cpu.Dockerfile -t local/audio.cpp:full-cpu .
```

## Usage

The model directory `<models-dir>` must be mounted into the container.
An additional `<output-dir>` should be mounted for TTS tasks.

### CUDA

```bash
docker run --rm --gpus all -v "<models-dir>:/models:ro" ghcr.io/0xshug0/audio.cpp:full-cuda12 <cli|server> --model /models/<model> <...>
```

### CPU

```bash
docker run --rm -v "<models-dir>:/models:ro" ghcr.io/0xshug0/audio.cpp:full-cpu <cli|server> --model /models/<model> <...>
```

See the fully working [examples](#examples) below.

## Examples

Examples for Docker, including CUDA and CPU, are available in `examples/docker`.

### CLI

The **[examples](../examples/docker/cli/EXAMPLE.md)** in `examples/docker/cli`
demonstrate how to run the audio.cpp CLI with `docker run`. The examples include:

- **PocketTTS:** Text-to-Speech
- **Qwen3-TTS:** Text-to-Speech with Voice Cloning

### Server

The **[examples](../examples/docker/server/EXAMPLE.md)** in `examples/docker/server`
demonstrate how to run the audio.cpp server with `docker compose`. The examples include:

- **PocketTTS:** Text-to-Speech
- **Qwen3-TTS:** Text-to-Speech with Voice Cloning
