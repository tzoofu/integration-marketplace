# OpenAI Whisper (transcription)

- **category**: ai-llm
- **provider**: OpenAI
- **reusable**: yes — a thin API wrapper around the Whisper transcription endpoint; a shared `@marketplace/openai-whisper` client would be a straightforward extraction if another repo needs speech-to-text.
- **docs**: https://platform.openai.com/docs/guides/speech-to-text

## Overview
OpenAI's `audio.transcriptions` endpoint (model `whisper-1`) converts an audio file to text, optionally with per-segment timestamps (`verbose_json` response format). For files longer than the API's per-request duration/size limits, audio needs to be normalized and split into chunks client-side first, with each chunk transcribed separately and stitched back into one continuous segment timeline.

## Playbook

### Prerequisites
- An OpenAI account with API access and billing enabled.
- `ffmpeg` available on the host (or an equivalent audio-processing library) for normalizing arbitrary input formats to a Whisper-friendly WAV/format and for chunking long recordings.
- The official `openai` Node/Python SDK (or any HTTP client capable of `multipart/form-data` uploads).

### Setup steps
1. Generate an API key in the OpenAI dashboard and store it as an environment variable — never hardcode it or check it into source control.
2. Normalize incoming audio to a format/sample-rate Whisper handles cleanly (e.g. mono WAV) using `ffmpeg`, if the source audio isn't already in a supported shape.
3. If a recording may exceed the API's practical single-request limits (long meetings, multi-hour files), split it into fixed-length chunks with a small helper that also records each chunk's start offset in the original timeline — segment timestamps returned per chunk are chunk-relative and must be shifted back by that offset.
4. Call the transcription endpoint once per chunk, in order, accumulating segments into one flat list.
5. Optionally pass the tail of the previous chunk's transcript back in as a `prompt` hint for the next chunk — it measurably improves continuity (spelling of names, terminology) across chunk boundaries for long recordings.
6. Store `temperature: 0` for deterministic, repeatable output unless intentionally sampling.

### Core pattern
```ts
import { createReadStream } from "node:fs";
import OpenAI from "openai";

interface VerboseSegment { start: number; end: number; text: string; }
interface VerboseResponse { text: string; segments?: VerboseSegment[]; }

interface Chunk { path: string; startSec: number; }

export async function transcribeChunks(
  apiKey: string,
  chunks: Chunk[],
  opts: { language?: string } = {}
): Promise<{ start: number; end: number; text: string }[]> {
  const client = new OpenAI({ apiKey });
  const allSegments: { start: number; end: number; text: string }[] = [];
  let priorContext = "";

  for (const chunk of chunks) {
    const response = (await client.audio.transcriptions.create({
      file: createReadStream(chunk.path),
      model: "whisper-1",
      language: opts.language,
      response_format: "verbose_json",
      prompt: priorContext || undefined,
      temperature: 0,
    })) as unknown as VerboseResponse;

    for (const s of response.segments ?? []) {
      allSegments.push({ start: s.start + chunk.startSec, end: s.end + chunk.startSec, text: s.text.trim() });
    }
    // Feed a short tail of context forward to help continuity across the
    // chunk boundary (names, jargon) — not required, but noticeably improves
    // accuracy on long multi-chunk recordings.
    priorContext = response.text.trim().split(/\s+/).slice(-50).join(" ");
  }

  return allSegments;
}
```

### Env vars
- `OPENAI_API_KEY`

### Gotchas
- `whisper-1` alone does **not** identify *who* said what — it only transcribes speech to text with timing. If speaker attribution is needed, that's a separate diarization step (see [pyannote-diarization](pyannote-diarization.md)) whose output gets merged with Whisper's segments afterward by matching timestamp ranges.
- Segment timestamps from a chunked file are relative to the start of *that chunk*, not the original recording — always add back the chunk's offset before using them, or downstream code silently produces wrong timings for every chunk after the first.
- Passing the previous chunk's trailing text as `prompt` measurably reduces terminology/name drift across chunk boundaries in long recordings — worth doing even though it's optional.
- Normalize the input format up front (e.g. to WAV via `ffmpeg`) rather than trusting arbitrary source formats to work — this avoids provider-side format-support edge cases becoming a runtime surprise mid-pipeline.

### Playbook confidence: high

## Adoption
Used in **1** repo in this marketplace. Consumed as the first stage of a multi-step audio-processing pipeline, transcribing chunked non-English audio with verbose JSON output so downstream stages can work with per-segment timestamps.
