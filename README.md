# Running a Local AI on a Jetson Orin Nano 8 GB

After trying different AI tools over the past few years — including Ollama and LM Studio on my main workstation — I wanted to set up a small local AI lab on a dedicated device, separate from my daily driver.

The reason is simple: I work primarily in cybersecurity, and I'd rather not send my queries to third-party servers where someone might end up in the news or on social media.

After doing a lot of reading, I explored different affordable alternatives. I remembered I had an old Jetson Nano 4 GB collecting dust that I'd originally bought for "emulators." Yes, that's the kind of person I am.

But that 4 GB Nano is completely useless for AI. One of the main challenges with AI is that you absolutely need RAM and GPU power. Buying a powerful graphics card and a new machine wasn't the budget-friendly route.

Then I remembered NVIDIA had released a new mini PC model — the [Jetson Orin Nano](https://amzn.to/4e5nhNK) — so I picked one up along with [a case](https://amzn.to/4cRMY2d) and a small M.2 drive, and waited for it to arrive.

What seemed like an afternoon project turned into weeks of trying different tools until I found one that actually worked on this hardware.

This article documents that journey.

Don't expect to run large models. Keep in mind we're talking about a device with 8 GB of RAM — though with a surprisingly capable processor.

![Jetson Orin Nano](https://www.alexmilla.net/wp-content/uploads/2026/04/Jetson-Nano-Orin.jpeg)

## Specs

```
NVIDIA Jetson Orin Nano 8 GB

CPU: ARM Cortex-A78AE, 6 cores, 64-bit, 1.5 GHz
GPU: NVIDIA Ampere, 1024 CUDA cores, 32 Tensor cores
RAM: 8 GB LPDDR5 (unified memory, shared between CPU and GPU)
Memory bandwidth: 102 GB/s (Super version)
AI performance: up to 67 TOPS (Super version, with software update)
Storage: supports NVMe (not included)
Connectivity: 2x MIPI CSI (cameras), USB 3.2, Gigabit Ethernet, M.2 Key E & Key M, 40-pin GPIO
Power consumption: 7W to 25W configurable
Dimensions: 100 x 79 x 21 mm
Price: ~$249 USD (Developer Kit)
GPU architecture: Ampere, compute capability 8.7
Software support: JetPack (L4T), CUDA, cuDNN, TensorRT, DeepStream, Isaac, Riva
```

## Starting Point: Ollama

My first choice was Ollama. I already had it installed and running on my desktop PC. It's convenient — simple CLI, Modelfile support for custom profiles, and you can download both local and cloud models with a single command.

The problem wasn't Ollama itself, though the software would occasionally hang or consume more resources than expected. At some point it became frustrating, so I started looking for more flexible alternatives.

Ollama works like a black box: you download a model, chat with it, and that's about it. I wanted more control over inference, native API access, and — given my limited RAM — the ability to squeeze every bit of performance from the GPU.

On top of that, the newest 2026 models (like Qwen 3.5, Qwen 3.6, or Gemma 4) take a while to reach Ollama. Some don't work at all due to incompatibilities with multimodal vision files.

With GGUF files from Hugging Face, you get access to everything from day one.

## Failed Attempt: Unsloth Studio

I saw that Unsloth had launched a fairly complete GUI called Unsloth Studio — local model chat, integrated Hugging Face model search, code execution, auto-healing tool calling, Model Arena for comparing models... It looked great.

The problem: **it doesn't work on the Jetson Orin Nano.** Unsloth Studio supports Mac, Windows, and x86 Linux with desktop NVIDIA GPUs, but the Jetson uses an ARM architecture (aarch64) with a different GPU stack (JetPack/L4T). There's no way to install it.

Ruled out.

## What Worked: llama.cpp Compiled with CUDA

In the end, the most straightforward approach — and the one that gave the best results — was compiling llama.cpp from source with CUDA support. No middlemen, no abstraction layers. The pure inference engine connected directly to the Orin's GPU.

![llama.cpp](https://www.alexmilla.net/wp-content/uploads/2026/04/LLAMA-IA.png)

The compilation was clean, no surprises. Within minutes the binaries were ready. The best thing about llama.cpp is that it includes a built-in web server (llama-server) that exposes a basic chat GUI and an OpenAI-compatible API. You don't need to install anything else to get started.

I downloaded the GGUF models directly from Hugging Face, launched the server, and opened the browser from my PC. Chat running at 9 tokens per second with GPU acceleration. For a machine under $350 that fits in the palm of your hand, more than decent.

## Choosing Models: Less Is More

With 8 GB of RAM shared between CPU and GPU, there's no room for fantasies. After subtracting what the OS consumes, you're left with about 5-6 GB for the model. That puts you in the 2-4B parameter range.

And here's the surprise: 4B models in 2026 are not the toys they were a year ago. Qwen3.5-4B, for example, has a reasoning mode, supports 201 languages, and performs surprisingly well for everyday queries. It's no Claude or GPT-4, but for analyzing a log, translating a paragraph, or explaining a technical concept, it does the job.

I ended up with three models installed, each for a different type of task:

- **Qwen3.5-4B** — My default model. General reasoning, technical queries, basic cybersecurity.
- **Gemma 4 E2B** — When I need to write or translate content. It writes with a more natural tone than other models its size.
- **Phi-4 Mini** — The fast one. For short tasks where I need speed over depth.

Only one model fits in memory at a time. That's one of the downsides, but switching takes just a few seconds.

## The Final Touch: Auto-Start

The last thing I configured was a systemd service so the chat server starts automatically when the Orin boots. No need to SSH in or remember any commands. I turn on the machine, open the browser from any device on my local network, and the AI is right there waiting.

If the service crashes, systemd relaunches it automatically.

## What's Missing and What's Coming

The llama-server GUI is functional but basic.

From here on, what I understand — and what I've learned over these months — is how AI is actually used in practice. Working with it for specific tasks without getting into expensive, heavy model training.

Building my own applications that connect to this small local AI through its API. Leveraging that power for Blue Team workflows, which is my day-to-day, combining it with the tools I already use and saving time.

Obviously, all of this will be complemented with paid AI services. I won't lie to you, readers. As I tell my colleagues: while AI is cheap, you should leverage it to build your own complementary tools — the ones you're missing or that are too expensive. Build yourself a foundation to be one step ahead.

## Is It Worth It?

If you're looking for an assistant that competes with ChatGPT or Claude in response quality, NO. A 4B model doesn't reach that level. But if what you want is privacy, independence from external services, and an assistant that works without internet or subscriptions — where you send a request and can wait a few minutes for it to finish — then yes.

The Jetson Orin Nano is silent, low power, fits anywhere, and once configured it just works. For anyone working with sensitive data or simply wanting full control over their tools, it has a value that goes beyond raw performance.

I've published the full step-by-step technical guide (in English) for anyone who wants to replicate the setup from scratch starting from a freshly installed OS.

👉 **[Setup Guide](GUIDE.md)**

---

**Original blog post (Spanish):** [alexmilla.net](https://www.alexmilla.net/montando-una-ia-local-en-una-jetson-orin-nano-de-8-gb/)
