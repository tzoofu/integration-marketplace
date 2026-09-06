# Google Gemini (meeting summarization)

- **category**: ai-llm
- **provider**: Google (Gemini, called via the Vercel AI SDK — `ai` + `@ai-sdk/google`)
- **reusable**: yes — the `generateText`-via-AI-SDK call pattern is provider-agnostic and could be lifted into a shared `@marketplace/ai-sdk-client` package; only the prompt/schema content itself is app-specific.
- **docs**: https://ai.google.dev/gemini-api/docs and https://sdk.vercel.ai/docs

## Overview
Rather than calling Google's Gemini SDK directly, this pattern goes through the Vercel AI SDK's provider-agnostic `generateText` call, using `@ai-sdk/google` as the concrete provider adapter. That abstraction is the main reason this is worth generalizing: swapping providers (Gemini → OpenAI → Anthropic) later is a one-line change to which adapter is constructed, not a rewrite of the calling code.

## Playbook

### Prerequisites
- A Google AI Studio / Gemini API key.
- `ai` and `@ai-sdk/google` installed (or the equivalent adapter package for whichever provider is actually in use — `@ai-sdk/openai`, `@ai-sdk/anthropic`, etc., all share the same `generateText` call shape).

### Setup steps
1. Generate an API key for the Gemini API and store it as an environment variable.
2. Install the Vercel AI SDK core package (`ai`) plus the Google provider adapter (`@ai-sdk/google`).
3. Make the model id configurable via an environment variable rather than hardcoding it, so the model can be bumped (e.g. a new Gemini version) without a code change or redeploy.
4. Build the prompt/system-message content as pure string templates fed by already-structured data (e.g. a formatted transcript) — keep the *parsing* of that input separate from the *prompt construction*, so the prompt template stays easy to iterate on.
5. Instruct the model explicitly not to fabricate information it wasn't given, and to use a fixed, predictable output structure (fixed section headers in a fixed order) if the output will be parsed or displayed structurally downstream.

### Core pattern
```ts
import { generateText } from "ai";
import { createGoogleGenerativeAI } from "@ai-sdk/google";

const SYSTEM_PROMPT = `You are an assistant that produces structured summaries.
Always respond in the requested language and in Markdown.
Use exactly these headings, in this order:
## Title
## Summary
## Key topics
## Decisions
## Action items
## Key quotes

Guidelines:
- Do not invent information. If something is unclear, omit it.
- If speakers are labeled (SPEAKER_00, SPEAKER_01, ...), preserve those labels in quotes.`;

export async function summarize(apiKey: string, modelId: string, content: string): Promise<string> {
  const google = createGoogleGenerativeAI({ apiKey });

  const { text } = await generateText({
    model: google(modelId),
    system: SYSTEM_PROMPT,
    prompt: content,
  });

  return text;
}

// Swapping providers later is just a different adapter + model id — the
// calling code (summarize's signature and generateText call shape) does not
// change:
//   import { createOpenAI } from "@ai-sdk/openai";
//   const openai = createOpenAI({ apiKey });
//   const { text } = await generateText({ model: openai("gpt-4o"), ... });
```

### Env vars
- `GOOGLE_GENERATIVE_AI_API_KEY`
- `SUMMARY_MODEL` (model id, e.g. a `gemini-*` version string — kept configurable so it can be bumped without a code change)

### Gotchas
- If a model-id env var might be stored with a provider prefix (e.g. `"google/gemini-2.5-pro"`) for use in a different tool/UI elsewhere, strip that prefix before passing it to the SDK's provider-specific model constructor — the constructor expects a bare model id, not a prefixed one.
- Having a second provider's adapter package installed "for later" but never actually invoked is easy to lose track of — call out clearly in code/docs which provider is the *live* one if more than one adapter is present in the dependency tree, to avoid a future maintainer assuming both are active.
- Explicit "do not invent information" and "omit if unclear" instructions in the system prompt measurably reduce hallucinated content in long-document summarization — worth keeping even though it seems obvious.
- Fixing the output's heading structure (exact headings, exact order) in the system prompt is what makes the result safely parseable/renderable downstream — without it, a model will vary section naming/ordering run to run.

### Playbook confidence: high

## Adoption
Used in **1** repo in this marketplace. Consumed as the final stage of a multi-step audio-processing pipeline, generating a structured, non-English Markdown summary (topics, decisions, action items, quotes) from a diarized, merged transcript.
