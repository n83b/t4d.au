# Osaurus + Gemma 4 12B QAT — Local AI That Just Works

[Osaurus](https://osaurus.app) is a native Swift app for macOS that gives you a clean, fast chat interface for running local language models via MLX. No browser, no Electron wrapper, no background Node process — just a proper macOS app that behaves like one.

I have been running it with Google's `gemma-4-12b-qat-it-mlx` model and the combination is genuinely impressive. It is the first local setup that has felt production-ready for day-to-day use.

## The Model: Gemma 4 12B QAT

`gemma-4-12b-qat-it-mlx` is Google's Gemma 4 12-billion parameter instruction-tuned model, quantization-aware trained (QAT) and converted to MLX format by the MLX Community. QAT is worth calling out — instead of quantizing a finished model after the fact, the quantization is baked into training, which means the 4-bit weights suffer far less quality degradation than a post-hoc quantize would produce.

The result: a 12B model that fits comfortably in 16 GB of unified memory and produces output that punches well above its weight class.

## Getting the Model

Pull it from the MLX Community on Hugging Face. If you have `mlx-lm` installed you can do it from Terminal:

```bash
mlx_lm.generate \
  --model mlx-community/gemma-4-12b-qat-it-mlx \
  --prompt "Hello, introduce yourself."
```

The first run downloads and caches the weights (~7 GB). After that, Osaurus can pick it up directly from the Hugging Face cache without any manual path wrangling.

## Setting Up in Osaurus

1. Open **Osaurus → Preferences → Models**
2. Point it at `mlx-community/gemma-4-12b-qat-it-mlx` — Osaurus resolves the local cache automatically if the model has already been downloaded via `mlx-lm` or another MLX tool
3. Set a system prompt if you want one, then start chatting

The interface is minimal in the right way — a sidebar for conversation history, a clean composer, and nothing else demanding your attention.

## What Works Well

- **Speed** — responses start streaming almost immediately on M-series chips; on an M3 Max the throughput is fast enough that you stop noticing it is running locally
- **Long context** — Gemma 4 supports a large context window and Osaurus handles it without complaint
- **Coding tasks** — the 12B QAT model is strong enough for SwiftUI snippets, shell scripts, and explaining build errors; I use it as a first-pass before reaching for a cloud model
- **Privacy** — nothing leaves the machine, which matters when pasting real code

## Why Native Apps Matter

Osaurus and OrbStack both make the same bet: that building a real Swift app for macOS is worth the extra effort. The payoff is an app that respects system preferences, responds to keyboard shortcuts as expected, does not spin up hidden background processes, and does not consume 400 MB just to render a text field.

I only use native apps. The difference in feel between a native tool and an Electron wrapper is immediately obvious — snappier window resizing, proper macOS menu bar integration, no white flash on launch, correct scroll physics. Osaurus nails all of it.

## Verdict

If you are running local models on Apple Silicon, the `Osaurus + gemma-4-12b-qat-it-mlx` pairing is currently the best native macOS experience I have found. The model quality is high enough for real work and the app stays out of your way.
