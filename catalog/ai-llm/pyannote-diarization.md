# pyannote.audio (speaker diarization)

- **category**: ai-llm
- **provider**: Hugging Face (hosting the gated `pyannote/speaker-diarization-3.1` and `pyannote/segmentation-3.0` models, run **locally** via the `pyannote.audio` Python library — not a hosted inference API)
- **reusable**: partial — the "Node spawns a Python subprocess running `pyannote.audio` with torch/torchaudio, exchanging JSON over stdout" pattern is reusable for any repo needing local diarization, but it requires provisioning a real Python environment and downloading gated model weights, unlike the marketplace's other pure-HTTP-API integrations.
- **docs**: https://huggingface.co/pyannote/speaker-diarization-3.1 and https://github.com/pyannote/pyannote-audio

## Overview
Speaker diarization ("who spoke when") is run **locally**, not via a hosted API — `pyannote.audio`'s pipeline downloads gated model weights from Hugging Face (requiring an accepted license + access token) and runs inference on-device with `torch`/`torchaudio`, using GPU/MPS acceleration if available. A Node.js (or other non-Python) app integrates it by shelling out to a small Python script and parsing its stdout as JSON.

## Playbook

### Prerequisites
- A Hugging Face account, with both gated model licenses accepted in the browser (`pyannote/segmentation-3.0` and `pyannote/speaker-diarization-3.1` — the pipeline pulls both).
- A Hugging Face access token (`read` scope is sufficient) generated from account settings.
- A Python 3.x environment where `pyannote.audio`, `torch`, `torchaudio`, and `soundfile` can be installed — this is a real Python dependency, not optional tooling, so plan for a virtualenv even in an otherwise all-JS/TS repo.
- Enough local compute to run inference in reasonable time (Apple Silicon MPS or an NVIDIA CUDA GPU meaningfully speeds this up over CPU-only).

### Setup steps
1. In a browser, visit both gated model pages on Hugging Face and click "Agree" to accept their license terms — inference will fail with an access error until this is done for **both** models, even if only the top-level pipeline model id is referenced in code.
2. Generate a Hugging Face access token and store it as an environment variable.
3. Create a Python virtualenv, `pip install` the pinned versions of `pyannote.audio`, `torch`, `torchaudio`, and `soundfile` — pin exact versions; `pyannote.audio`'s pipeline internals have had breaking changes across versions.
4. Write a small, single-purpose Python script that: loads the pipeline with the HF token, runs it on a given WAV path, and prints a JSON array of `{start, end, speaker}` turns to stdout (with all human-readable progress/debug logging sent to **stderr**, never stdout, so the caller's JSON parse doesn't choke on it).
5. From the host app, spawn that script as a child process per audio file, passing the HF token via environment (not as a CLI argument, to avoid it leaking into process-list/shell history), and parse stdout as JSON on clean exit.
6. Detect and select the best available device (MPS on Apple Silicon, else CUDA, else CPU) inside the Python script itself, logging which one was chosen to stderr for debuggability.

### Core pattern
```python
#!/usr/bin/env python3
"""Run pyannote speaker diarization on a WAV file.
Usage: diarize.py <wav_path>
Reads an HF access token from env. Writes JSON [{start, end, speaker}, ...]
to stdout; all progress/debug logs go to stderr.
"""
import json, os, sys

def main() -> int:
    if len(sys.argv) < 2:
        print("usage: diarize.py <wav_path>", file=sys.stderr)
        return 2

    wav_path = sys.argv[1]
    token = os.environ.get("HUGGINGFACE_TOKEN")
    if not token:
        print("HUGGINGFACE_TOKEN not set", file=sys.stderr)
        return 3

    print("[diarize] loading pipeline...", file=sys.stderr)
    from pyannote.audio import Pipeline
    import torch

    pipeline = Pipeline.from_pretrained(
        "pyannote/speaker-diarization-3.1", use_auth_token=token,
    )
    if torch.backends.mps.is_available():
        pipeline.to(torch.device("mps")); print("[diarize] using MPS", file=sys.stderr)
    elif torch.cuda.is_available():
        pipeline.to(torch.device("cuda")); print("[diarize] using CUDA", file=sys.stderr)
    else:
        print("[diarize] using CPU", file=sys.stderr)

    diarization = pipeline(wav_path)
    turns = [
        {"start": round(float(t.start), 3), "end": round(float(t.end), 3), "speaker": str(s)}
        for t, _, s in diarization.itertracks(yield_label=True)
    ]
    json.dump(turns, sys.stdout)
    return 0

if __name__ == "__main__":
    sys.exit(main())
```

```ts
// Node/TS caller — spawns the script, feeds the HF token via env, and
// parses stdout strictly as JSON (stderr is separate and never parsed).
import { spawn } from "node:child_process";

interface SpeakerTurn { start: number; end: number; speaker: string; }

export async function diarize(wavPath: string, hfToken: string, pythonBin = "python3"): Promise<SpeakerTurn[]> {
  return new Promise((resolve, reject) => {
    const proc = spawn(pythonBin, ["diarize.py", wavPath], {
      env: { ...process.env, HUGGINGFACE_TOKEN: hfToken },
    });
    let stdout = "", stderr = "";
    proc.stdout.on("data", (d) => (stdout += d.toString()));
    proc.stderr.on("data", (d) => (stderr += d.toString()));
    proc.on("error", reject);
    proc.on("close", (code) => {
      if (code !== 0) return reject(new Error(`diarize.py exited ${code}\n${stderr}`));
      try { resolve(JSON.parse(stdout)); }
      catch (err) { reject(new Error(`Failed to parse diarize.py output: ${(err as Error).message}\n${stdout}`)); }
    });
  });
}
```

Merging diarization turns with a separately-produced transcript (e.g. from Whisper) is a straightforward interval-overlap match: for each transcript segment, pick the speaker turn whose `[start, end)` overlaps it most.

### Env vars
- `HUGGINGFACE_TOKEN`
- `PYTHON_BIN` (path to the Python interpreter/venv to spawn, if not the default `python3` on `PATH`)
- `DEBUG_DIARIZATION` (verbose flag — gate whether the spawning process forwards the child's stderr progress logs to its own output)

### Gotchas
- Both gated model pages need their license accepted on Hugging Face — accepting only the top-level pipeline model's license and not its underlying segmentation model's license produces a confusing access-denied error that looks like a token problem but isn't.
- Keep the Python script's **stdout reserved exclusively for the final JSON payload** — any `print()` used for progress/debugging must go to stderr, or the caller's `JSON.parse` breaks on interleaved log lines.
- Pin exact versions of `pyannote.audio`/`torch`/`torchaudio` — this pipeline's internals and model-loading API have changed across versions in backward-incompatible ways.
- Pass the access token via the child process's environment, not as a command-line argument — CLI args are visible in process listings and shell history on most systems.
- This is a real local-compute dependency (Python + ML libraries + downloaded model weights), unlike a pure hosted-API integration — budget setup/CI time and disk space accordingly, and don't assume it'll "just work" in a container that hasn't provisioned Python.

### Playbook confidence: high

## Adoption
Used in **1** repo in this marketplace. Consumed as a middle stage of a multi-step audio-processing pipeline, attributing transcript segments to distinct speakers before a downstream summarization step.
