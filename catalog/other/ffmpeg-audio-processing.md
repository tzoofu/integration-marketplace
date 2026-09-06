# ffmpeg / ffprobe (audio processing)

- **category**: other
- **provider**: FFmpeg project (invoked as external system processes, not an npm/pip dependency)
- **reusable**: yes — the normalize/probe/silence-detect/chunk pattern around system `ffmpeg`/`ffprobe` binaries is generic and reusable for any repo that needs to prep audio before sending it to a transcription API (e.g. Whisper) with a payload-size limit.
- **docs**: https://ffmpeg.org/documentation.html

## Overview
FFmpeg and ffprobe are shelled out to as child processes (no SDK/library binding) to normalize arbitrary uploaded audio into a canonical 16kHz mono PCM WAV, probe its duration, detect silence gaps, and slice it into sub-25MB chunks aligned to silence boundaries so each chunk stays under an API's upload size limit.

## Playbook

### Prerequisites
- `ffmpeg` and `ffprobe` binaries installed and on `PATH` in every environment this runs in (dev machine, CI, and the production/serverless runtime — note this often means a custom Docker layer or buildpack, since these binaries are not installed by default on most PaaS/serverless images).
- Node.js `child_process` (or equivalent process-spawning API in another language/runtime).

### Setup steps
1. Install ffmpeg (e.g. `apt-get install -y ffmpeg` in the container image, or the platform's ffmpeg buildpack/layer). Verify with `ffmpeg -version` and `ffprobe -version` at startup or in CI.
2. Write a thin `run(cmd, args)` process wrapper that spawns the binary, buffers stdout/stderr, and rejects on non-zero exit code.
3. Build a `probeDuration` function on top of `ffprobe` to get total audio length in seconds.
4. Build a `normalizeToWav` function on top of `ffmpeg` to downmix/resample any input format to 16kHz mono 16-bit PCM WAV (the format most speech APIs expect).
5. Build a `detectSilences` function using ffmpeg's `silencedetect` audio filter, parsing `silence_start:`/`silence_end:` markers out of stderr (ffmpeg writes filter output to stderr, not stdout).
6. Build a chunker that: computes a target max chunk duration from a byte-size budget and the known bytes/sec of the normalized format, and if the file exceeds that duration, picks cut points near (but not exactly at) each target boundary by snapping to the nearest detected silence center within a search window, then slices with a `sliceAudio` function (`-ss`/`-t` + same normalize flags).

### Core pattern
```ts
import { spawn } from "node:child_process";
import { promises as fs } from "node:fs";
import path from "node:path";

// 1. Generic process wrapper for any ffmpeg/ffprobe invocation
function run(cmd: string, args: string[]): Promise<{ stdout: string; stderr: string }> {
  return new Promise((resolve, reject) => {
    const proc = spawn(cmd, args);
    let stdout = "";
    let stderr = "";
    proc.stdout.on("data", (d) => (stdout += d.toString()));
    proc.stderr.on("data", (d) => (stderr += d.toString()));
    proc.on("error", reject);
    proc.on("close", (code) => {
      if (code === 0) resolve({ stdout, stderr });
      else reject(new Error(`${cmd} ${args.join(" ")} exited ${code}\n${stderr}`));
    });
  });
}

// 2. Probe duration (seconds) via ffprobe
export async function probeDurationSec(audioPath: string): Promise<number> {
  const { stdout } = await run("ffprobe", [
    "-v", "error",
    "-show_entries", "format=duration",
    "-of", "default=noprint_wrappers=1:nokey=1",
    audioPath,
  ]);
  const dur = parseFloat(stdout.trim());
  if (Number.isNaN(dur)) throw new Error(`Could not probe duration of ${audioPath}`);
  return dur;
}

// 3. Normalize any input to 16kHz mono PCM WAV
export async function normalizeToWav(inputPath: string, outputDir: string): Promise<string> {
  await fs.mkdir(outputDir, { recursive: true });
  const outputPath = path.join(outputDir, `${path.basename(inputPath, path.extname(inputPath))}.wav`);
  await run("ffmpeg", [
    "-y", "-i", inputPath,
    "-ac", "1", "-ar", "16000", "-c:a", "pcm_s16le",
    outputPath,
  ]);
  return outputPath;
}

// 4. Detect silence gaps via the silencedetect filter (output lands on stderr)
export async function detectSilences(
  audioPath: string,
  thresholdDb = -35,
  minSilenceSec = 0.5
): Promise<Array<{ start: number; end: number }>> {
  const { stderr } = await run("ffmpeg", [
    "-i", audioPath,
    "-af", `silencedetect=noise=${thresholdDb}dB:d=${minSilenceSec}`,
    "-f", "null", "-",
  ]);

  const silences: Array<{ start: number; end: number }> = [];
  const starts: number[] = [];
  const ends: number[] = [];
  let m: RegExpExecArray | null;
  const startRegex = /silence_start:\s*(\d+\.?\d*)/g;
  const endRegex = /silence_end:\s*(\d+\.?\d*)/g;
  while ((m = startRegex.exec(stderr)) !== null) starts.push(parseFloat(m[1]));
  while ((m = endRegex.exec(stderr)) !== null) ends.push(parseFloat(m[1]));
  for (let i = 0; i < Math.min(starts.length, ends.length); i++) {
    silences.push({ start: starts[i], end: ends[i] });
  }
  return silences;
}

// 5. Slice a time range out of the audio, re-applying the target format
export async function sliceAudio(
  inputPath: string,
  outputPath: string,
  startSec: number,
  durationSec: number
): Promise<void> {
  await run("ffmpeg", [
    "-y",
    "-ss", startSec.toString(),
    "-t", durationSec.toString(),
    "-i", inputPath,
    "-ac", "1", "-ar", "16000", "-c:a", "pcm_s16le",
    outputPath,
  ]);
}

// 6. Chunk a normalized WAV into sub-size-limit pieces, snapped to silence
const TARGET_CHUNK_BYTES = 23 * 1024 * 1024; // stay under an API's ~25MB limit
const BYTES_PER_SEC_16K_MONO_WAV = 16000 * 2;
const SOFT_MAX_CHUNK_SEC = Math.floor(TARGET_CHUNK_BYTES / BYTES_PER_SEC_16K_MONO_WAV);

export async function chunkAudio(wavPath: string, chunkDir: string) {
  await fs.mkdir(chunkDir, { recursive: true });
  const totalDuration = await probeDurationSec(wavPath);

  if (totalDuration <= SOFT_MAX_CHUNK_SEC) {
    return [{ index: 0, path: wavPath, startSec: 0, durationSec: totalDuration }];
  }

  const silences = await detectSilences(wavPath, -35, 0.5);
  const silenceCenters = silences.map((s) => (s.start + s.end) / 2);

  const boundaries: number[] = [0];
  let nextTarget = SOFT_MAX_CHUNK_SEC;
  while (nextTarget < totalDuration) {
    // prefer a nearby silence center over a hard cut at the target
    const candidate =
      silenceCenters.find((c) => c > nextTarget - 60 && c < nextTarget + 30) ?? nextTarget;
    if (candidate > boundaries[boundaries.length - 1] + 10) boundaries.push(candidate);
    nextTarget = candidate + SOFT_MAX_CHUNK_SEC;
  }
  boundaries.push(totalDuration);

  const chunks = [];
  for (let i = 0; i < boundaries.length - 1; i++) {
    const start = boundaries[i];
    const dur = boundaries[i + 1] - start;
    if (dur < 1) continue;
    const chunkPath = path.join(chunkDir, `chunk_${String(i).padStart(3, "0")}.wav`);
    await sliceAudio(wavPath, chunkPath, start, dur);
    chunks.push({ index: i, path: chunkPath, startSec: start, durationSec: dur });
  }
  return chunks;
}
```

### Env vars
None — this pattern has no configuration; behavior is driven by constants (sample rate, channel count, silence threshold/duration, target chunk byte size) that should be tuned per downstream API's limits.

### Gotchas
- `ffmpeg`'s `silencedetect` filter writes its `silence_start:`/`silence_end:` markers to **stderr**, not stdout — a naive wrapper that only captures stdout will see nothing.
- Chunk boundary selection deliberately searches a window around the target time (`nextTarget - 60` to `nextTarget + 30` seconds in the reference implementation) rather than requiring an exact match, since silence rarely falls exactly on the byte-budget boundary; falling back to a hard cut at `nextTarget` when no silence is found nearby avoids infinite loops on continuous audio.
- The byte-budget chunk size is derived from the *known, fixed* output format (16kHz mono 16-bit PCM = 32,000 bytes/sec) — this only works because normalization happens before chunking; chunking based on the original (variable, compressed) input format's bitrate would be inaccurate.
- ffmpeg/ffprobe must be present in the runtime `PATH`; this is easy to miss in containerized or serverless deployments that don't include them by default, causing a spawn `ENOENT` at runtime rather than a build-time failure.
- `sliceAudio` re-applies the normalization flags (`-ac 1 -ar 16000 -c:a pcm_s16le`) on every slice rather than assuming the sliced output inherits the input's format — needed because ffmpeg's `-ss`/`-t` slicing re-encodes rather than simply copying bytes.

### Playbook confidence: high

## Adoption
Used in **1** repo in this marketplace.
