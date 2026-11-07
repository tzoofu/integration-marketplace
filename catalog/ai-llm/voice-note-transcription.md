# Voice-note transcription (Gemini or local whisper.cpp)

- **category**: ai-llm
- **provider**: Google (Gemini API, `google.golang.org/genai`) and whisper.cpp (`whisper-cli`, local) — pluggable
- **reusable**: yes — a `Transcriber` interface with two engines, an ffmpeg front end and a durable job queue is app-agnostic; any app that receives short audio clips can drop it in.
- **docs**: https://ai.google.dev/gemini-api/docs/audio · https://github.com/ggml-org/whisper.cpp · https://pkg.go.dev/google.golang.org/genai

Transcribes incoming voice notes in the background with an engine picked by env: Gemini (audio sent inline, structured JSON back with text + detected language) or whisper.cpp running locally so audio never leaves the machine. Every engine first normalizes the clip with ffmpeg; a small worker pool drains a DB-backed queue with retries and stores the transcript next to the message.

## Playbook

### Prerequisites
- Go 1.22+ (uses the builtin `max`/`min`).
- `ffmpeg` on the host (both engines need it).
- Gemini engine: a Gemini API key and `go get google.golang.org/genai`.
- Whisper engine: whisper.cpp built locally (`whisper-cli` binary) plus a GGML model file (e.g. `ggml-base.bin` or `ggml-small.bin`, downloaded with whisper.cpp's `models/download-ggml-model.sh`).
- A database table for transcription jobs (any SQL store with `UPDATE … RETURNING`, e.g. SQLite 3.35+ or Postgres).

### Setup steps
1. Define a `Transcriber` interface (`Transcribe(ctx, ownerID, path) (Result, error)` + `Name()`), where `Result` holds `Text` and a BCP-47 `Language`.
2. Write an `FFmpeg` helper with `Check()` (runs `ffmpeg -version` at startup) and `Convert(in, "wav"|"flac")` that writes 16 kHz mono output into a fresh temp dir and returns a cleanup func.
3. Implement the Gemini engine: convert to FLAC, read the bytes, send them as an inline audio part plus a "transcribe verbatim, don't translate" instruction, ask for a JSON response with a schema `{text, language}` and temperature 0.
4. Implement the whisper engine: convert to 16-bit PCM WAV, run `whisper-cli -m <model> -f <wav> -l auto -oj -of <base> -np` under a timeout, read `<base>.json`, join the `transcription[].text` segments and take `result.language`.
5. At startup build a map of *available* engines: skip everything if ffmpeg is missing, add Gemini only if a key resolver exists, add whisper only if `exec.LookPath(whisperBin)` succeeds. Pick the default from env (`gemini | whisper | off`) and validate it in config.
6. Create a jobs table keyed by the message, with `status` (`queued|processing|done|failed`), `engine`, `language`, `text`, `attempts`, `error`, `next_attempt_at`, `completed_at`, and an index on `(status, next_attempt_at, created_at)`.
7. Start N worker goroutines (env, default 2). On boot, reset any `processing` rows back to `queued` (crash recovery). Each worker claims one due job atomically, runs the engine, then marks done (and indexes the text for search) or failed with a retry time.
8. Enqueue from the message-ingest path when a voice/audio message's media is downloaded; wake a worker via a 1-slot notify channel so new jobs start instantly, with a ticker as fallback.
9. Expose a "re-transcribe (optionally with engine X)" endpoint that re-enqueues the job with an explicit engine, and push a `transcript.done` event (SSE/websocket) so the UI updates live.

### Core pattern
```go
package transcribe

import (
	"bytes"
	"context"
	"encoding/json"
	"errors"
	"fmt"
	"os"
	"os/exec"
	"path/filepath"
	"strings"
	"time"

	"google.golang.org/genai"
)

type Result struct{ Text, Language string }

type Transcriber interface {
	Name() string
	Transcribe(ctx context.Context, ownerID int64, path string) (Result, error)
}

// ---- ffmpeg front end -------------------------------------------------------

type FFmpeg struct{ Bin string }

var ErrDecode = errors.New("could not decode the audio")

// Only local files and real audio containers: a "voice note" that is secretly an HLS/concat
// playlist must not make ffmpeg open other files or URLs.
func inputArgs(in string) []string {
	return []string{"-protocol_whitelist", "file,pipe",
		"-format_whitelist", "ogg,webm,matroska,mov,mp4,m4a,mp3,wav,aac,flac", "-i", in}
}

func (f *FFmpeg) Convert(ctx context.Context, in, format string) (string, func(), error) {
	codec := map[string]string{"wav": "pcm_s16le", "flac": "flac"}[format]
	if codec == "" {
		return "", func() {}, fmt.Errorf("unsupported format %q", format)
	}
	dir, err := os.MkdirTemp("", "audio-*")
	if err != nil {
		return "", func() {}, err
	}
	cleanup := func() { os.RemoveAll(dir) }
	out := filepath.Join(dir, "out."+format)
	ctx, cancel := context.WithTimeout(ctx, 60*time.Second)
	defer cancel()
	args := append([]string{"-hide_banner", "-loglevel", "error", "-y"}, inputArgs(in)...)
	args = append(args, "-ar", "16000", "-ac", "1", "-c:a", codec, out)
	var stderr bytes.Buffer
	cmd := exec.CommandContext(ctx, f.Bin, args...)
	cmd.Stderr = &stderr
	if err := cmd.Run(); err != nil {
		cleanup()
		// stderr can echo file paths: log it at debug only, return a generic error.
		return "", func() {}, ErrDecode
	}
	return out, cleanup, nil
}

// ---- Gemini engine ----------------------------------------------------------

type Gemini struct {
	Client *genai.Client // genai.NewClient(ctx, &genai.ClientConfig{APIKey: key, Backend: genai.BackendGeminiAPI})
	Model  string        // e.g. "gemini-2.5-flash"
	FF     *FFmpeg
}

func (g *Gemini) Name() string { return "gemini" }

func (g *Gemini) Transcribe(ctx context.Context, _ int64, path string) (Result, error) {
	flac, cleanup, err := g.FF.Convert(ctx, path, "flac") // Ogg/Opus is not a Gemini input type
	if err != nil {
		return Result{}, err
	}
	defer cleanup()
	data, err := os.ReadFile(flac)
	if err != nil {
		return Result{}, err
	}
	parts := []*genai.Part{
		genai.NewPartFromBytes(data, "audio/flac"),
		genai.NewPartFromText("Transcribe this voice message verbatim in its original language. Do not translate or summarize."),
	}
	resp, err := g.Client.Models.GenerateContent(ctx, g.Model,
		[]*genai.Content{genai.NewContentFromParts(parts, genai.RoleUser)},
		&genai.GenerateContentConfig{
			Temperature:      genai.Ptr[float32](0),
			ResponseMIMEType: "application/json",
			ResponseSchema: &genai.Schema{Type: genai.TypeObject, Required: []string{"text", "language"},
				Properties: map[string]*genai.Schema{
					"text":     {Type: genai.TypeString},
					"language": {Type: genai.TypeString, Description: "BCP-47 language code, e.g. es or en-US"},
				}},
		})
	if err != nil {
		return Result{}, err
	}
	var raw struct{ Text, Language string }
	if err := json.Unmarshal([]byte(resp.Text()), &raw); err != nil {
		return Result{}, err
	}
	return Result{Text: strings.TrimSpace(raw.Text), Language: raw.Language}, nil
}

// ---- whisper.cpp engine (local) ---------------------------------------------

type Whisper struct {
	Bin, Model string // "whisper-cli", "./models/ggml-base.bin"
	FF         *FFmpeg
}

func (w *Whisper) Name() string { return "whisper" }

func (w *Whisper) Transcribe(ctx context.Context, _ int64, path string) (Result, error) {
	wav, cleanup, err := w.FF.Convert(ctx, path, "wav")
	if err != nil {
		return Result{}, err
	}
	defer cleanup()
	base := filepath.Join(filepath.Dir(wav), "whisper") // writes base.json in the same temp dir
	ctx, cancel := context.WithTimeout(ctx, 5*time.Minute)
	defer cancel()
	var stderr bytes.Buffer
	cmd := exec.CommandContext(ctx, w.Bin, "-m", w.Model, "-f", wav, "-l", "auto", "-oj", "-of", base, "-np")
	cmd.Stderr = &stderr
	if err := cmd.Run(); err != nil {
		return Result{}, fmt.Errorf("whisper: %w", err)
	}
	raw, err := os.ReadFile(base + ".json")
	if err != nil {
		return Result{}, err
	}
	var out struct {
		Result        struct{ Language string `json:"language"` } `json:"result"`
		Transcription []struct{ Text string `json:"text"` }     `json:"transcription"`
	}
	if err := json.Unmarshal(raw, &out); err != nil {
		return Result{}, err
	}
	var b strings.Builder
	for _, s := range out.Transcription {
		b.WriteString(s.Text)
	}
	return Result{Text: strings.TrimSpace(b.String()), Language: out.Result.Language}, nil
}

// ---- engine registry --------------------------------------------------------

func Available(ctx context.Context, ff *FFmpeg, gemini *Gemini, whisperBin, whisperModel string) map[string]Transcriber {
	out := map[string]Transcriber{}
	if exec.CommandContext(ctx, ff.Bin, "-version").Run() != nil {
		return out // no ffmpeg: transcription disabled, log it loudly
	}
	if gemini != nil {
		out["gemini"] = gemini
	}
	if _, err := exec.LookPath(whisperBin); err == nil {
		out["whisper"] = &Whisper{Bin: whisperBin, Model: whisperModel, FF: ff}
	}
	return out
}
```

Worker pool over a DB queue:

```go
var backoff = []time.Duration{5 * time.Second, 30 * time.Second, 2 * time.Minute}
const maxAttempts = 3

// Claim one due job atomically (SQLite 3.35+/Postgres):
//   UPDATE transcription_jobs SET status='processing', attempts=attempts+1
//   WHERE id = (SELECT id FROM transcription_jobs
//               WHERE status='queued' AND next_attempt_at <= ? ORDER BY created_at LIMIT 1)
//   RETURNING id, message_id, coalesce(engine,''), attempts

type Worker struct {
	jobs          JobStore
	engines       map[string]Transcriber
	defaultEngine string
	n             int
	notify        chan struct{} // make(chan struct{}, 1)
}

func (w *Worker) Wake() {
	select { case w.notify <- struct{}{}: default: }
}

func (w *Worker) Start(ctx context.Context) {
	_ = w.jobs.RequeueProcessing(ctx) // crash recovery: processing -> queued
	for i := 0; i < max(w.n, 1); i++ {
		go w.loop(ctx)
	}
}

func (w *Worker) loop(ctx context.Context) {
	t := time.NewTicker(5 * time.Second)
	defer t.Stop()
	for {
		for w.processOne(ctx) && ctx.Err() == nil {
		}
		select {
		case <-ctx.Done():
			return
		case <-w.notify:
		case <-t.C:
		}
	}
}

func (w *Worker) processOne(ctx context.Context) bool {
	job, err := w.jobs.ClaimNext(ctx, time.Now().Unix())
	if err != nil {
		return false // ErrNotFound = queue empty
	}
	engine := job.Engine
	if engine == "" {
		engine = w.defaultEngine
	}
	tr := w.engines[engine]
	var res Result
	if tr == nil {
		err = ErrEngineUnavailable
	} else {
		res, err = tr.Transcribe(ctx, job.OwnerID, job.AudioPath)
	}
	saveCtx := context.WithoutCancel(ctx) // record the outcome even during shutdown
	if err != nil {
		retryAt := int64(0)
		if job.Attempts < maxAttempts && !isPermanent(err) {
			retryAt = time.Now().Add(backoff[job.Attempts-1]).Unix()
		}
		_ = w.jobs.Fail(saveCtx, job.ID, err.Error(), retryAt) // retryAt>0 -> back to queued
		return true
	}
	_ = w.jobs.Complete(saveCtx, job.ID, engine, res.Language, res.Text) // + index text for search
	// publish "transcript.done" to live clients here
	return true
}
```

### Env vars
- `APP_TRANSCRIBE_ENGINE` — `gemini` (default) | `whisper` | `off`
- `APP_TRANSCRIBE_WORKERS` — worker goroutines (default 2, must be >= 1)
- `APP_FFMPEG_BIN` — default `ffmpeg`
- `APP_WHISPER_BIN` — default `whisper-cli`
- `APP_WHISPER_MODEL` — path to the GGML model file
- `GEMINI_API_KEY` (or a per-user key resolved at call time; see Gotchas)

### Gotchas
- Gemini does not accept Ogg/Opus (the usual voice-note container), so always transcode first; FLAC 16 kHz mono keeps uploads small and lossless. whisper.cpp wants 16 kHz mono 16-bit PCM WAV — anything else fails or degrades.
- Treat incoming audio as untrusted input to ffmpeg: whitelist protocols (`file,pipe`) and demuxers. Without it a crafted "audio" file that is really a playlist can make ffmpeg read other local files or fetch URLs.
- Don't put ffmpeg/whisper stderr in user-facing errors; it echoes absolute paths. Return a generic "could not decode" error and log stderr at debug.
- Keep every conversion in its own `os.MkdirTemp` dir and `defer cleanup()` — whisper's `-of` writes `<base>.json` next to the WAV, so the same dir is cleaned in one call.
- Ask Gemini for the detected language in the same JSON response instead of a second call; instruct it explicitly *not* to translate or summarize, or it sometimes "helpfully" returns English.
- Use temperature 0 and the lowest thinking level the model supports (on models with configurable thinking) — transcription gains nothing from reasoning and it adds latency/cost.
- Pass `-l auto` to whisper for mixed-language inboxes; the detected language is in `result.language` of the `-oj` output, and text arrives as segments you must concatenate.
- Classify errors: missing engine, non-audio message, feature switched off, missing/invalid API key are *permanent* — don't burn retries on them. Network/5xx errors retry with backoff (5s, 30s, 2m, max 3 attempts).
- When a job is scheduled for retry, start a goroutine that sleeps until `retryAt` and then wakes the pool, otherwise retries wait for the next poll tick.
- On startup, reset rows stuck in `processing` back to `queued` — a crash mid-job otherwise leaves them orphaned forever.
- Use `context.WithoutCancel` when writing the job result so a shutdown signal doesn't drop a finished transcript.
- Per-tenant gating: check before enqueueing whether the owner turned transcription off or (for Gemini) has no usable key, and silently skip instead of queueing jobs that will fail. The whisper engine needs no key, so the "needs key" check depends on the engine.
- Transcripts are message content: never log the text above debug level.
- Build the engine map from what is actually installed and let the API's "re-transcribe with engine X" fail fast with "engine unavailable" rather than queueing.

### Playbook confidence: high

## Adoption
Used in **1** repo in this marketplace.
