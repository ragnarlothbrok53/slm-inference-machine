# slim-serve

**SLIM — Small Language Inference Machine.** An OpenAI-compatible streaming server for local
small language models (llama.cpp GGUF, MLX), with admission control, deadlines, cancellation and
Prometheus metrics.

[GitHub](https://github.com/ragnarlothbrok53/slm-inference-machine) ·
[Full README](https://github.com/ragnarlothbrok53/slm-inference-machine#readme) ·
[Architecture](https://github.com/ragnarlothbrok53/slm-inference-machine/blob/main/docs/architecture.md) ·
[Benchmarks](https://github.com/ragnarlothbrok53/slm-inference-machine/blob/main/benchmarks/results/README.md) ·
[Changelog](https://github.com/ragnarlothbrok53/slm-inference-machine/blob/main/CHANGELOG.md)

## Install

```bash
pip install "slim-serve[llama]" \
  --extra-index-url https://abetlen.github.io/llama-cpp-python/whl/cpu   # prebuilt CPU wheel
```

On Apple Silicon, `pip install "slim-serve[mlx]"` installs the MLX backend instead. Without the
extra index, pip builds llama-cpp-python from source, which needs a C++ compiler.

## Quick start

```bash
curl -L -o qwen.gguf \
  https://huggingface.co/Qwen/Qwen2.5-0.5B-Instruct-GGUF/resolve/main/qwen2.5-0.5b-instruct-q4_k_m.gguf
slim-serve serve --model qwen.gguf          # http://127.0.0.1:8000
```

No model? `SLM_ENGINE=fake slim-serve serve` serves deterministic fake tokens through the same
stack.

The OpenAI Python SDK works against it as is:

```python
from openai import OpenAI

client = OpenAI(base_url="http://127.0.0.1:8000/v1", api_key="unused")
for chunk in client.completions.create(model="any", prompt="Hello", max_tokens=32, stream=True):
    print(chunk.choices[0].text, end="", flush=True)
```

## What you get

- `POST /v1/completions`, streaming (SSE) and non-streaming, plus `GET /v1/models`.
- Inference on a dedicated thread per model replica, never on the event loop.
- A bounded queue that fails fast with `503 + Retry-After` instead of piling up latency.
- One deadline per request; timeouts and client disconnects stop generation within a token.
- `/healthz` (liveness) and `/readyz` (model loaded), so orchestrators route correctly.
- Prometheus metrics that separate queue wait, time to first token and decode speed.

It doesn't batch sequences: for raw GPU throughput, use vLLM or `llama-server`.
[Why, and where it breaks down](https://github.com/ragnarlothbrok53/slm-inference-machine#design-tradeoffs).

## Configuration

`SLM_*` environment variables, a `.env` file, or `serve` flags (which take precedence):

| Variable | Default | Meaning |
|---|---|---|
| `SLM_ENGINE` | `llama_cpp` | `llama_cpp`, `mlx` or `fake` |
| `SLM_MODEL_PATH` | `models/qwen2.5-0.5b-instruct-q4_k_m.gguf` | GGUF file, or MLX model dir / HF id |
| `SLM_REPLICAS` | `1` | Independent model copies, each with its own thread |
| `SLM_MAX_QUEUE_SIZE` | `32` | Requests that may wait before `503` |
| `SLM_REQUEST_TIMEOUT_S` | `120` | Deadline covering queue wait and generation |
| `SLM_HOST`, `SLM_PORT` | `127.0.0.1`, `8000` | Bind address |

[All settings](https://github.com/ragnarlothbrok53/slm-inference-machine#configuration) ·
`slim-serve info` prints the detected hardware and effective configuration.

## Learn more

- [Performance](https://github.com/ragnarlothbrok53/slm-inference-machine#performance--evaluation): measured queueing behaviour, serving overhead, prefill vs. KV cache, replicas
- [Failure handling](https://github.com/ragnarlothbrok53/slm-inference-machine#failure-handling) and [metrics](https://github.com/ragnarlothbrok53/slm-inference-machine#observability)
- [Design decisions](https://github.com/ragnarlothbrok53/slm-inference-machine/blob/main/docs/design-decisions.md)
- [SLOM](https://pypi.org/project/slom/): durable, tool-using agents built on top of slim-serve

MIT licensed.
