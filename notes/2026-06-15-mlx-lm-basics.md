# Getting Started with Apple's mlx-lm

Apple's [MLX](https://ml-explore.github.io/mlx/) framework lets you run and fine-tune large language models entirely on your Mac, taking full advantage of Apple Silicon's unified memory architecture. No GPU cloud bills. No data leaving your machine.

`mlx-lm` is the dedicated Python package that sits on top of MLX and handles all the LLM-specific plumbing: downloading models, tokenization, text generation, quantization, and even LoRA fine-tuning.

## Prerequisites

- A Mac with Apple Silicon (M1 or later)
- Python 3.9+
- A virtual environment is strongly recommended

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install mlx-lm
```

## Running Inference from the Terminal

The quickest way to try it out. Models are pulled automatically from the [MLX Community](https://huggingface.co/mlx-community) on Hugging Face and cached locally on first run.

```bash
# One-shot text generation
mlx_lm.generate --model mlx-community/Llama-3.2-3B-Instruct-4bit \
                 --prompt "Explain unified memory in one paragraph."

# Interactive chat session
mlx_lm.chat --model mlx-community/Llama-3.2-3B-Instruct-4bit
```

The `4bit` suffix means the model weights have been quantized to 4-bit precision — dramatically smaller on disk and lighter on RAM while staying surprisingly capable.

## Using the Python API

For anything beyond a quick test, drop into Python directly:

```python
from mlx_lm import load, generate

# Load model + tokenizer (downloads on first call)
model, tokenizer = load("mlx-community/Llama-3.2-3B-Instruct-4bit")

# Apply the model's chat template so the prompt is formatted correctly
messages = [{"role": "user", "content": "What is the capital of France?"}]
prompt = tokenizer.apply_chat_template(
    messages,
    tokenize=False,
    add_generation_prompt=True
)

# Generate a response
response = generate(model, tokenizer, prompt=prompt, max_tokens=256, verbose=True)
print(response)
```

`apply_chat_template` is important — without it you are sending raw text to a model that expects structured turn markers, which produces garbled output.

## Local OpenAI-Compatible Server

`mlx-lm` ships with a server that speaks the OpenAI REST API. Any tool that works with OpenAI (LangChain, Open WebUI, Cursor, etc.) can point at it instead.

```bash
mlx_lm.server --model mlx-community/Llama-3.2-3B-Instruct-4bit --port 8080
```

Then hit it like you would the real API:

```bash
curl http://localhost:8080/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "mlx-community/Llama-3.2-3B-Instruct-4bit",
    "messages": [{"role": "user", "content": "Hello!"}]
  }'
```

## Quantizing Your Own Models

If a model you want isn't in the MLX Community yet, you can convert and quantize it yourself:

```bash
mlx_lm.convert \
  --hf-path meta-llama/Meta-Llama-3-8B-Instruct \
  --mlx-path ./llama-3-8b-4bit \
  -q
```

The `-q` flag enables 4-bit quantization. The output directory contains the MLX-native weights ready to load with `mlx_lm.load()`.

## What to Try Next

- **LoRA fine-tuning** — `mlx_lm.lora` lets you fine-tune a model on your own dataset on-device in minutes.
- **Batch generation** — pass a list of prompts to `generate()` to process them efficiently in one shot.
- **Streaming** — set `stream=True` in the Python API for token-by-token output, useful for chat UIs.
