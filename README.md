<div align=center>
    <img src="https://parcel.pyke.io/v2/cdn/assetdelivery/diffusers/doc/diffusers.png" width="100%" alt="pyke Diffusers">
    <hr />
</div>

# Please see the [v2](https://github.com/pykeio/diffusers/tree/v2) branch and related PR: [#28](https://github.com/pykeio/diffusers/pull/28)
Diffusers 1.0 has enhanced documentation, graph fusion optimizations, CLIP skip, and textual inversion.

<hr />

pyke Diffusers is a modular [Rust](https://rust-lang.org/) library for pretrained diffusion model inference to generate images using [ONNX Runtime](https://onnxruntime.ai/) as a backend for accelerated generation on both CPUs & GPUs.

- [Features](#features)
- [Prerequisites](#prerequisites)
  * [LMS notes](#lms-notes)
- [Installation](#installation)
- [Usage](#usage)
  * [Examples](#examples)
  * [Converting models](#converting-models)
  * [ONNX Runtime binaries](#onnx-runtime-binaries)
  * [CUDA and other execution providers](#cuda-and-other-execution-providers)
  * [Low memory usage](#low-memory-usage)
    + [Quantization](#quantization)
- [Roadmap](#roadmap)

## Features
- Text-to-image for Stable Diffusion v1 & v2
- Optimized for both CPU and GPU inference
- Memory-efficient pipelines to run with <2GB of RAM!
- \>77 token prompts
- Prompt weighting, e.g. `a (((house:1.3)) [on] a (hill:0.5), sun, (((sky))).`
- Implements DDIM, DDPM, DPM/DPM++, Euler & Euler a, LMS schedulers

## Prerequisites
You'll need **[Rust](https://rustup.rs) v1.62.1+** to use pyke Diffusers.

- If using CPU: recent (no earlier than Haswell/Zen) x86-64 CPU for best results. ARM64 supported but not recommended. For acceleration, see notes for [OpenVINO](https://onnxruntime.ai/docs/execution-providers/OpenVINO-ExecutionProvider.html#requirements), [oneDNN](https://onnxruntime.ai/docs/execution-providers/oneDNN-ExecutionProvider.html), [ACL](https://onnxruntime.ai/docs/execution-providers/ACL-ExecutionProvider.html), [SNPE](https://onnxruntime.ai/docs/execution-providers/SNPE-ExecutionProvider.html)
- If using CUDA: **CUDA v11.[x](https://docs.nvidia.com/deploy/cuda-compatibility/#minor-version-compatibility)**, **cuDNN v8.2.x** <sup>[more info](https://onnxruntime.ai/docs/execution-providers/CUDA-ExecutionProvider.html#requirements)</sup>
- If using TensorRT: **CUDA v11.[x](https://docs.nvidia.com/deploy/cuda-compatibility/#minor-version-compatibility)**, **TensorRT v8.4** <sup>[more info](https://onnxruntime.ai/docs/execution-providers/TensorRT-ExecutionProvider.html)</sup>
- If using ROCm: **ROCm v5.2** <sup>[more info](https://onnxruntime.ai/docs/execution-providers/ROCm-ExecutionProvider.html)</sup>
- If using DirectML: **DirectX 12 compatible GPU**, **Windows 10 v1903+** <sup>[more info](https://onnxruntime.ai/docs/execution-providers/DirectML-ExecutionProvider.html)</sup>

Only generic CPU, CUDA, and TensorRT have prebuilt binaries available (*for now*). Other execution providers will require you to manually build them; see the ONNX Runtime docs for more info. Additionally, you'll need to [make `ort` link to your custom-built binaries](https://github.com/pykeio/ort#execution-providers).

### LMS notes
> **Note**:
> **By default, the LMS scheduler is not enabled**, and this section can simply be skipped.

If you plan to enable the `all-schedulers` or `scheduler-lms` feature, you will need to install binaries for the GNU Scientific Library. See the [installation instructions for `rust-GSL`](https://github.com/GuillaumeGomez/rust-GSL#installation) to set up GSL.

## Installation
```toml
[dependencies]
pyke-diffusers = "0.2"
# if you'd like to use CUDA:
pyke-diffusers = { version = "0.2", features = [ "ort-cuda" ] }
```

The default features enable some commonly used schedulers and pipelines.

## Usage
```rust
use pyke_diffusers::{
    EulerDiscreteScheduler, OrtEnvironment, SchedulerOptimizedDefaults, StableDiffusionOptions, StableDiffusionPipeline,
    StableDiffusionTxt2ImgOptions
};

let environment = OrtEnvironment::default().into_arc();
let mut scheduler = EulerDiscreteScheduler::stable_diffusion_v1_optimized_default()?;
let pipeline = StableDiffusionPipeline::new(&environment, "./stable-diffusion-v1-5", StableDiffusionOptions::default())?;

let mut imgs = StableDiffusionTxt2ImgOptions::default()
    .with_prompts("photo of a red fox", None)
    .run(&pipeline, &mut scheduler)?;

imgs.remove(0).into_rgb8().save("result.png")?;
```

### Examples
`pyke-diffusers` includes an interactive Stable Diffusion demo. Run it with:
```bash
$ cargo run --example stable-diffusion-interactive --features ort-cuda -- ~/path/to/stable-diffusion/
```

See [`examples/`](https://github.com/pykeio/diffusers/tree/main/examples) for more examples and [the docs](https://docs.rs/pyke-diffusers) for more detailed information.

### Converting models
pyke Diffusers currently supports Stable Diffusion v1, v2, and its derivatives.

To convert a model from a Hugging Face `diffusers` model:
1. Create and activate a virtual environment.
2. Install Python requirements:
    - install torch with CUDA: `python3 -m pip install torch --extra-index-url https://download.pytorch.org/whl/cu116`
    - install dependencies: `python3 -m pip install -r requirements.txt`
3. If you are converting a model directly from Hugging Face, log in to Hugging Face Hub with `huggingface-cli login` - this can be skipped if you have the model on disk
5. Convert your model with `scripts/hf2pyke.py`:
    - To convert a float32 model from HF (recommended for CPU inference): `python3 scripts/hf2pyke.py runwayml/stable-diffusion-v1-5 ~/pyke-diffusers-sd15/`
    - To convert a float32 model from disk: `python3 scripts/hf2pyke.py ~/stable-diffusion-v1-5/ ~/pyke-diffusers-sd15/`
    - To convert a float16 model from HF (recommended for GPU inference): `python3 scripts/hf2pyke.py --fp16 runwayml/stable-diffusion-v1-5@fp16 ~/pyke-diffusers-sd15-fp16/`
    - To convert a float16 model from disk: `python3 scripts/hf2pyke.py --fp16 ~/stable-diffusion-v1-5-fp16/ ~/pyke-diffusers-sd15-fp16/`

float16 models are faster on some GPUs and use less memory. `hf2pyke` supports a few options to improve performance or ORT execution provider compatibility. See `python3 scripts/hf2pyke.py --help`.

### ONNX Runtime binaries
When running the examples in this repo on Windows, you'll need to *copy the `onnxruntime*` dylibs from `target/debug/` to `target/debug/examples/`* on first run. You'll also need to copy the dylibs to `target/debug/deps/` if your project uses pyke Diffusers in a Cargo test.

### CUDA and other execution providers
CUDA is the only alternative execution provider available with no setup required. Simply enable pyke Diffusers' `ort-cuda` feature and enable `DiffusionDevice::CUDA`; see the docs or the [`stable-diffusion` example](https://github.com/pykeio/diffusers/blob/main/examples/stable-diffusion.rs) for more info. You may need to rebuild your project for `ort` to copy the libraries again.

For other EPs like DirectML or oneDNN, you'll need to build ONNX Runtime from source. See `ort`'s notes on [execution providers](https://github.com/pykeio/ort#execution-providers).

### Low memory usage
Lower resolution generations require less memory usage.

A `StableDiffusionMemoryOptimizedPipeline` exists for environments with low memory. This pipeline *removes the safety checker* and will only load models when they are required and unloads them immediately after. This will heavily impact performance and should only be used in extreme cases.

#### Quantization
In extremely constrained environments (e.g. <= 4GB RAM), it is also possible to produce a quantized int8 model. The int8 model's quality is heavily impacted, but faster and less memory intensive on CPUs.

To convert an int8 model:
```bash
$ python3 scripts/hf2pyke.py --quantize=ut ~/stable-diffusion-v1-5/ ~/pyke-diffusers-sd15-quantized/
```

`--quantize=ut` will quantize only the UNet and text encoder using uint8 mode for best quality and performance. You can choose to convert the other models using the following format:
- each model is assigned a letter: `u` for UNet, `v` for VAE, and `t` for text encoder.
- a lowercase letter means the model will be quantized to uint8
- an uppercase letter means the model will be quantized to int8

Typically, uint8 is higher quality and faster, but you can play around with the settings to see if quality or speed improves.

A combination of 256x256 image generation via `StableDiffusionMemoryOptimizedPipeline` with a uint8 UNet only requires **1.3 GB** of memory usage.

## [Roadmap](https://github.com/pykeio/diffusers/issues/22)
- [x] Import from original Stable Diffusion checkpoints
- [x] Graph fusion for better optimization
- [ ] Implement img2img, inpainting, and upscaling ([#2](https://github.com/pykeio/diffusers/issues/2))
- [ ] Textual inversion
- [x] VAE approximation
- [ ] CLIP layer skip
- [ ] Rewrite scheduler system ([#16](https://github.com/pykeio/diffusers/issues/16))
- [x] Acceleration for M1 Macs ([#14](https://github.com/pykeio/diffusers/issues/14))
- [ ] Web interface
- [ ] Explore other backends (pyke's DragonML, [tract](https://github.com/sonos/tract))
