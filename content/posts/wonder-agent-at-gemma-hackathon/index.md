---
title: "Every question within reach: building Wonder Agent with Gemma, LiteRT, and NVIDIA"
date: 2026-11-12
draft: false
tags: ["engineering", "hackathon", "Gemma"]
---

A child sees a picture of a dog, learns its name, and asks why dogs bark. One word becomes an invitation to explore. **Wonder Agent is a language education app that helps children learn English words through pictures, voice, and interactive conversation.**

Our vision is to bring the marginal cost of education closer to zero, starting with each additional learning interaction. Our assumption is that access improves when a child's next question does not incur another cloud inference charge. Running Gemma and speech inference entirely on the device after setup helps us pursue that vision.

That assumption gave us an engineering problem: could we build a language learning experience that runs on a phone, works without a network connection after setup, and responds in a way a child can comfortably follow? Getting a model to run would be the first step. We also needed useful explanations, understandable illustrations, local speech, and clear controls for asking and interrupting.

Devices, electricity, downloads, and development still cost money. Local inference removes the per-request cloud inference bill; making the whole experience accessible remains a broader product challenge.

Wonder Agent began as Frank's solo Android project for the Google Gemma Hardware Hackathon at [AGI House](https://www.agihouse.org/) on October 24. It won third place for Best Use of LiteRT, second place in the Mobile AI track, and the Best Use of Pixel / TPU award. We are continuing that work as **Curilo Wonder**. This article covers the hackathon project and the engineering documented as it has developed.

## Why we started with words

{{< youtube 5o3rZJnITXU >}}

The motivation is personal. Frank's earlier work with the student NGO used technology to support education in rural China. After the hackathon, he teamed up with a software engineer friend to help launch it into a free education app.

We are both parents as well as builders, creating a learning app we want to use with our own children. Our priority is to make the time parents and children spend together more meaningful: exploring a picture, asking a question, and discovering something together. That shared experience guides our product decisions.

English vocabulary gives us a focused place to begin. A word provides a concrete topic, a picture gives it a visual anchor, and conversation lets the child choose what to explore next. Our broader ambition is to help children discover the world through their questions.

In the app, a child browses categories and words, taps a speaker to hear a word, or opens its assistant. Gemma introduces the concept and invites a question. The child can speak or type, hear the answer, and follow the spoken words through highlighted text. You can see it in the [video demo](https://www.youtube.com/shorts/5o3rZJnITXU).

## A local pipeline for conversations

We use [LiteRT-LM](https://github.com/google-ai-edge/LiteRT-LM), Google's framework for running language models on edge devices, to run Gemma locally. Our Android integration supports Gemma 4 E2B and E4B artifacts, with backend selection governed by the model and device's capabilities.

The AI assistant separates the jobs behind a voice interaction:

1. **Capture and transcribe.** The app records a question into memory and sends the audio to local Gemma with a transcription instruction.
2. **Build the context.** A second request combines the transcript with the selected word, category, recent dialogue, and available illustration context.
3. **Generate and check.** Gemma produces a short reply, which passes local safety checks before the app displays or speaks it.
4. **Speak and highlight.** A separate offline text-to-speech engine produces the voice while the interface highlights the current words.

_(Please note: after the hackathon we'd combined Step 1 and 2 later for faster user interactions.)_

Gemma handles transcription, explanations, and image understanding. Separate TTS engines synthesize speech, and separate diffusion models power the image-generation experiments. Keeping those roles explicit helps us measure and improve each part.

The engine is reused between transcription and response requests, avoiding repeated initialization. Each request gets bounded context, with recent text dialogue supplying conversational memory. The reply prompt asks for no more than fifty spoken words: answer length is a product decision that also affects waiting time.

Model provisioning requires downloads; inference runs locally afterward. The app can also refresh vocabulary content and download voices, so offline inference should not be confused with an app that never uses the network.

## Measure the wait that matters

For a child, performance is the time between asking a question and receiving something useful. Tokens per second explains only part of that experience.

We built a Research Lab view with [two language-model workloads](https://github.com/frankdu/hack_android_base/blob/main/docs/BENCHMARKING.md). A fixed-token benchmark measures initialization, time to first token, and the rates at which the model processes its prompt and generates output. Repeatable input and output lengths help isolate runtime performance. A prompt playground exercises real prompts and separately measures engine loading, first visible text, and response completion. It also lets us observe engine reuse.

The distinction matters. A fast decoding rate can coexist with a long initialization delay. Native time to first token can differ from the time the app displays meaningful text. Neither measurement, by itself, tells us when a child hears the first spoken sentence.

Results retain model hashes, backend, runtime, device information, and environmental readings such as thermal status. Warm-ups stay out of summaries. Our default three measured runs are exploratory; they cannot establish reliable tail latency. We also keep backend failures visible instead of silently switching to CPU and labeling the result as an accelerator run.

[LiteRT](https://developers.google.com/edge/litert) provides conversion, optimization, and CPU, GPU, and NPU execution capabilities. In our app, acceleration remains specific to the artifact, runtime, and phone. Our [device validation](https://github.com/frankdu/hack_android_base/blob/main/docs/VALIDATION.md#photo-decoder-and-incomplete-json-fixes--october-20-2026) records Gemma E2B execution using Tensor G5 TPU text with GPU vision, but that does not establish TPU support for every model or for the diffusion pipelines. A compatible model file and a successful device run are both necessary.

## Generating 10,000 presentation images with NVIDIA

For the hackathon presentation, we generated **10,000 images**. The phone image-generation runs we were using took roughly **180–360 seconds per image**, or three to six minutes. At that pace, preparing the full collection on a phone would consume valuable hackathon time.

We used an **NVIDIA RTX 6000 Ada GPU workstation** to generate the batch, achieving approximately **1.5 images per second**. That throughput made preparing 10,000 images for the presentation practical within the hackathon workflow. NVIDIA accelerated the image preparation work, while Gemma and LiteRT powered the app's local learning interactions.

The workstation figure measures throughput across the presentation batch. These observations describe our hackathon workflows; a controlled hardware comparison would require matching the model, resolution, sampling settings, and timing boundaries. The LiteRT phone measurements below are separate experiments with their own documented configurations.

## Using Codex and the LiteRT skill on a remote workstation

The same RTX 6000 Ada workstation, running Ubuntu, also hosted our model preparation work. We used **Codex with the [LiteRT CLI agent skill](https://github.com/google-ai-edge/LiteRT-CLI/blob/main/.agents/skills/litert_cli/SKILL.md)** and had Codex connect remotely to download models and carry out the conversion and packaging workflow. The skill gives the coding agent instructions for using LiteRT's tools, including conversion, quantization, and benchmarking. Model-specific scripts handled the details of the image pipelines.

That workflow produced **five public LiteRT model packages**, published under [frankdu on Hugging Face](https://huggingface.co/frankdu/models). We created new conversions for SD-Turbo and SDXL-Turbo and reused existing LiteRT Community conversions for the other three, preserving their attribution. The published model cards describe each package's origin:

| Public model package                                                                                                                                                     | Preparation and provenance                                                              |
| ------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | --------------------------------------------------------------------------------------- |
| [SD-Turbo-Mobile-INT8-LiteRT](https://huggingface.co/frankdu/SD-Turbo-Mobile-INT8-LiteRT), and [SD-Turbo-LiteRT](https://huggingface.co/frankdu/SD-Turbo-LiteRT)         | New LiteRT conversion, including the model's sampling pipeline                          |
| [SDXL-Turbo-Mobile-INT8-LiteRT](https://huggingface.co/frankdu/SDXL-Turbo-Mobile-INT8-LiteRT), and [SDXL-Turbo-LiteRT](https://huggingface.co/frankdu/SDXL-Turbo-LiteRT) | New LiteRT conversion, including the model's sampling pipeline                          |
| [FLUX.2-klein-4B-LiteRT](https://huggingface.co/frankdu/FLUX.2-klein-4B-LiteRT)                                                                                          | Attributed LiteRT Community mirror, with reconstructed host constants and a CPU sampler |
| [Bonsai-Image-ternary-4B-LiteRT](https://huggingface.co/frankdu/Bonsai-Image-ternary-4B-LiteRT)                                                                          | Attributed LiteRT Community mirror with a runnable CPU example                          |
| [iris-3b-Turbo-LiteRT](https://huggingface.co/frankdu/iris-3b-Turbo-LiteRT)                                                                                              | Attributed LiteRT Community mirror with reference engines                               |

The packages include runnable examples, sample images, validation records, checksums, and source licenses. Each completed a desktop LiteRT CPU generation check. Publishing those artifacts gives other developers a starting point for reproducing the pipelines and testing their own devices. As of this writing, the models already incurred more than 3000 downloads.

## Making image models practical to download

Following the five-model exploration, our current Android LiteRT image catalog focuses on **SD-Turbo, SDXL-Turbo, and FLUX.2 Klein 4B**. The public releases preserve the broader work, and we also published compact SD-Turbo and SDXL-Turbo variants for phone use.

The first challenge was packaging. The original SD-Turbo and SDXL-Turbo LiteRT exports stored FP32 weights and required large downloads. Quantization reduces the precision used to store those weights. Using model-specific export scripts and LiteRT CLI quantization, we produced compact bundles with dynamic INT8 encoder and denoiser weights and FP16 image-decoder (VAE) weights. Activations remain floating point.

The [October 24 quantization audit](https://github.com/frankdu/hack_android_base/blob/main/docs/PIXEL_IMAGE_MODELS.md) records the change:

| Model      | Previous runtime download | Compact runtime download | Reduction |
| ---------- | ------------------------: | -----------------------: | --------: |
| SD-Turbo   |                   5.03 GB |                  1.32 GB |    73.75% |
| SDXL-Turbo |                  13.75 GB |                  3.53 GB |    74.35% |

These are decimal GB for the runtime files. Smaller downloads help provisioning, but file size does not establish peak memory, speed, or accelerator compatibility. We checked the quantized pipelines against their FP32 baselines on three fixed subjects. The subjects remained recognizable, while composition and detail could change; broader illustration-quality evaluation is still needed.

## The bug between a converted model and a working phone

One of the most useful lessons came from a tensor-buffer bug. Some exported text-encoder graphs exposed an input token tensor as a passthrough output. Our Android runner allocated a new output buffer, inadvertently replacing the input allocation with uninitialized data.

The consequences looked like model failures: incorrect conditioning in SD-Turbo and an out-of-bounds embedding lookup that aborted SDXL-Turbo's native process. The fix was to identify shared input/output tensors in the manifests, reuse their buffers, and release each shared allocation only once.

Both corrected pipelines then completed phone generation. This is the kind of work hidden behind a successful model conversion: tokenization, tensor ownership, graph loading, and cleanup all have to agree with the exported model.

Compatibility was another separate hurdle. The compact SD exports failed GPU graph compilation on our tested Tensor G5 phone, including an attempt to put only the denoiser on GPU. We mark those combinations unavailable. Image TPU execution also requires separately compiled, validated bundles. Quantization solved a download problem; other runtime problems remained.

## Phone measurements changed the product design

The compact models produced recognizable blue teapot images on a **Pixel 10 Pro Fold with Tensor G5**. Our [CPU smoke measurements](https://github.com/frankdu/hack_android_base/blob/main/docs/PIXEL_IMAGE_MODELS.md) were:

| Model                | Resolution / steps | Generation time | Peak worker RSS |
| -------------------- | ------------------ | --------------: | --------------: |
| SD-Turbo INT8/FP16   | 512 × 512 / 1      |         26.92 s |        1.75 GiB |
| FLUX.2 Klein 4B INT8 | 256 × 256 / 4      |         85.76 s |        2.09 GiB |
| SDXL-Turbo INT8/FP16 | 512 × 512 / 4      |         86.90 s |        1.75 GiB |

Each is a single sample using a fixed prompt and seed, a fresh isolated worker, and timing that includes graph loading and PNG output. Different resolutions and step counts prevent a controlled speed ranking. RSS measures resident process memory and does not capture every possible accelerator allocation.

For the product, the practical implication was clear: even a roughly 27-second generation takes too long to put in front of every conversational reply. We use prepared vocabulary illustrations and reuse visual context. If no stored illustration summary exists, Gemma can caption the image after foreground reply generation and save the result locally for later text prompts. A new question cancels unfinished captioning so it can take priority.

This makes the image useful across multiple turns without repeatedly processing it. It also keeps optional background work from blocking the child's next question.

## Responsiveness includes control

We explored sentence-level streaming as a way to begin speech before a full response finishes. The current learning app buffers and checks the complete reply before display or speech, then splits it into sentences for playback. Earlier speech delivery remains an optimization to evaluate alongside content checking; we have not established a measured time-to-first-speech improvement.

Meanwhile, clear interaction states make the experience easier to follow. The child taps **Talk** for each question. **Done** submits a recording; **Cancel** discards or stops the current work; **Stop speaking** interrupts playback. The microphone stays closed while the assistant speaks and does not reopen automatically afterward. Leaving the assistant stops audio and inference.

These choices matter as much as raw throughput. A child should understand when the app is listening and be able to stop it. Local prompting and reply checks are initial guardrails; age-appropriate response quality, voice recognition, and learning value still need evaluation with children on real devices.

## The next step

We plan to release **Curilo Wonder as a free education app for parents and children on both Android and iOS**. We are building it as a **not-for-profit project**, with the aim of helping families make the most of their time together. Keeping the app free reflects both that purpose and our broader vision of making learning more accessible.

The hackathon gave us encouragement and a working foundation. The next step is to connect engineering progress to the learning experience: measure time to first audible speech on representative phones, validate more accelerator configurations, improve context selection, and evaluate the conversations with children and their caregivers.

One detail from the build still makes us smile: Frank burned through 20+ GB on Google Fi and 7+ GB on AT&T Mobile. A friend's fiber connection rescued a few critical hours. Apparently, building offline AI takes quite a bit of downloading first. 😅

Thank you to Google and [AGI House](https://www.agihouse.org/) for the hackathon, to the LiteRT team for the tools, and to the Google LiteRT and NVIDIA teams for inviting us to share this project. Follow our progress at [curilo.app](https://curilo.app).

We want a child's next question to be an opportunity we can afford to answer. Wonder Agent is our first step toward making that possible on the device already in their hands.
