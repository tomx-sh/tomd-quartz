---
publish: true
created: 2026-07-31
modified: 2026-08-07T09:59:00.353Z
---

# Running AI models locally

Short notes on hardware, models, and the settings that make local AI more responsive.

> [!note]
> Model names and results reflect testing in July 2026. Local AI changes quickly, so treat them as a shortlist rather than permanent recommendations.

## Quick model picks

| Hardware                  | Models to try                                                      |
| ------------------------- | ------------------------------------------------------------------ |
| Raspberry Pi, 8 GB        | `qwen3:4b` (best tested result), `qwen3:1.7b`, `gemma4:e2b-it-qat` |
| Raspberry Pi, 16 GB       | `qwen3:8b`, `gemma4:e4b-it-qat`, `gemma4:12b-it-qat`               |
| M1 Pro MacBook Pro, 32 GB | `gemma4:12b-mlx`                                                   |
| M4 MacBook Air, 32 GB     | `gemma4:e4b-mlx`                                                   |
| HP EliteDesk G4 AMD 8GB   | `gemma4:e2b-it-qat`                                                |

For a dedicated workstation, AMD Threadripper Pro on WRX90 is one high-end CPU/platform option. Sipeed's [K3](https://sipeed.com/k3) is another piece of dedicated hardware worth investigating.

## Apple Silicon

### Ollama

Ollama currently gives the best tested results on a MacBook Pro with:

- `qwen3.6:35b-a3b-coding-nvfp4` for coding;
- `gemma4:26b-a4b-it-qat` as a strong quantized option;
- `gemma4:31b-it-qat` when memory permits;
- smaller Qwen or Gemma variants when fast responses matter more than capability.

Gemma 4 QAT variants reduce memory usage while preserving much of the model quality. Available sizes include `e2b`, `e4b`, `12b`, `26b-a4b`, and `31b`.

### MLX and oMLX

Use **MLX 4-bit** models rather than GGUF files with oMLX. Candidates include:

- `Qwen3-Coder-30B-A3B-Instruct-4bit` for coding and agents;
- a Qwen 35B A3B 4-bit model for general reasoning;
- `Qwen3.5-9B-4bit` as a faster fallback.

In practice, oMLX produced less satisfactory results than Ollama in these tests.

## Making OpenCode responsive with Ollama

Tests on a fanless MacBook Air M4 with 24 GB of unified memory found that slow visible responses did **not** come primarily from decoding speed.

The largest sources of delay were:

- hidden reasoning before the visible answer;
- automatic conversation-title generation on the first turn;
- OpenCode's large system prompt and tool definitions;
- prompt-cache invalidation when requests competed with each other.

The changes with the greatest impact were:

- disable automatic title generation;
- disable reasoning by default and enable it only for difficult tasks;
- keep a single model loaded;
- allow only one parallel Ollama request;
- cap the working context instead of using the model's theoretical maximum;
- enable automatic compaction and pruning for long conversations.

Suggested OpenCode settings:

```json
{
  "agent": {
    "title": { "disable": true }
  },
  "compaction": {
    "auto": true,
    "prune": true,
    "reserved": 4096
  }
}
```

Suggested Ollama environment:

```text
OLLAMA_KEEP_ALIVE=30m
OLLAMA_MAX_LOADED_MODELS=1
OLLAMA_NUM_PARALLEL=1
```

A practical default budget on a 24 GB Mac is:

- context: 32,768 tokens;
- maximum output: 4,096 tokens;
- reasoning: off by default, with a selectable high-reasoning variant.

In the measured OpenCode setup, disabling title generation reduced visible time to first token from roughly **5–10 seconds** to **under one second** on repeated Qwen 2B requests. The improvement came from stable prompt-cache reuse, not faster token decoding.

## Useful checks

```bash
# Show resident models
ollama ps

# Inspect Ollama's metadata for a model
curl -s http://127.0.0.1:11434/api/show \
  -d '{"model":"qwen3.5:2b-mlx"}' | python3 -m json.tool

# Inspect OpenCode's merged configuration
opencode debug config

# Watch cache and timing information
tail -f ~/.ollama/logs/server.log
```

## Other local components

- Speech: [`blaizy/mlx-audio`](https://github.com/blaizy/mlx-audio) and Nemotron 3.5 ASR
- Headless browsing: [`h4ckf0r0day/obscura`](https://github.com/h4ckf0r0day/obscura)
- Fine-tuning: [Gemma Trainer walkthrough](https://dev.to/googleai/master-local-fine-tuning-with-gemma-trainer-3ipp)

## Takeaways

- Start with a 4B model on an 8 GB Raspberry Pi and an 8B–12B model on a 16 GB Pi.
- On Apple Silicon, test Ollama before investing time in alternative runtimes.
- For agent use, prompt processing and cache reuse can matter more than raw decode speed.
- Keep reasoning optional: small models can spend their entire output budget on hidden thinking.
- Use conservative context limits; advertised maximum context is rarely the best everyday setting.
