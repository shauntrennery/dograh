# Speechify (Simba 3.2) — Live Dograh Integration Evidence

Evidence for [dograh-hq/dograh#654](https://github.com/dograh-hq/dograh/pull/654) per the
[AI Provider Integration PR guidelines](https://github.com/dograh-hq/dograh/blob/main/CONTRIBUTING.md#ai-provider-integration-pull-requests).

| | |
|---|---|
| Test date | 2026-08-18 |
| Dograh commit | `85691b98` (branch `feat/speechify-simba-tts`, rebased on `main` @ `ca89ca3c`) |
| Pipecat commit | `51b98b325` (branch `feat/speechify-tts`, dograh-hq/pipecat#56) |
| Provider endpoint | `POST https://api.speechify.ai/v1/audio/stream` (Bearer auth) |
| Provider account | Self-service Speechify developer account; API key from https://platform.speechify.ai (redacted throughout) |
| Deployment | Local OSS stack (`docker-compose-local.yaml` Postgres/Redis/MinIO + `scripts/start_services_dev.sh` + `ui npm run dev`) |

## Pipeline configuration (BYOK)

| Service | Provider | Model | Notes |
|---|---|---|---|
| Voice (TTS) | **Speechify** | **simba-3.2**, voice `beatrice_32`, language `en` | The provider under test |
| Transcriber (STT) | Local Models (Speaches) | `Systran/faster-distil-whisper-small.en` | Local container, port 8124 |
| LLM | OpenAI-compatible local endpoint | `gpt-4.1` (canned-response stub) | Local stub on port 8123 — the LLM is not the provider under test; it exists so the pipeline can run without a cloud LLM key |

## What was run

A browser test call ("Test Audio" panel) against a blank-canvas workflow (workflow id 1, run id 1)
on the local stack. The Speechify adapter added by this PR (`SpeechifyTTSService`, streaming PCM at
the 16000 Hz transport rate) synthesized every bot turn.

- `01-speechify-voice-form.png` — Speechify selected in the schema-driven Voice provider dropdown; model/voice/language fields with per-model filtering.
- `02-invalid-key-rejected.png` — saving with an invalid key is blocked: "tts: Invalid Speechify API key. The key was rejected by the Speechify API…" (server-side probe of `GET /v1/voices`, 401 → save rejected).
- `03-config-saved.png` — save succeeds with the real key ("Model configuration saved").
- `04-live-call-connected.png`, `05-live-transcript.png` — live WebRTC call; multi-turn transcript (local whisper transcribed the fake-device mic audio; every bot line was spoken by Simba 3.2).
- `run1-bot.wav` — bot-only audio track (pure Speechify output), 16000 Hz mono, 53.18 s. `run1.wav` is the mixed recording. Both uploaded by the run to MinIO (`recordings/1.wav`, `recordings/1/bot.wav`).
- `speechify-run1-log-excerpt.txt` — backend log for run 1: `Creating TTS service: provider=ServiceProviders.SPEECHIFY, model=simba-3.2`, per-utterance `Generating TTS [...]`, TTFB 0.571–0.876 s, TTFA, character-usage metrics. No credentials appear in logs.

The run's stored analytics stamp (`workflow_runs.initial_context.runtime_configuration`):

```json
{"stt_provider": "speaches", "stt_model": "Systran/faster-distil-whisper-small.en",
 "tts_provider": "speechify", "tts_model": "simba-3.2",
 "llm_provider": "openai", "llm_model": "gpt-4.1"}
```

## Voice × model live matrix (basis for the per-model voice options)

`POST /v1/audio/stream`, `output_format=pcm_16000`, 2026-08-18:

| voice \ model | simba-3.2 | simba-3.0 | simba-english | simba-multilingual |
|---|---|---|---|---|
| beatrice_32 | 200 | 200 | 200 | 200 |
| geffen_32 | 200 | 200 | 200 | 200 |
| alicia | **400** | 200 | 200 | 200 |
| alton | **400** | 200 | 200 | 200 |

400 body: `{"error":{"code":"bad_request","message":"the selected voice is not available for simba-3.2. Choose a voice that lists simba-3.2 in GET /v1/voices, or use simba-english, simba-3.0, or simba-multilingual."}}`

`GET /v1/voices` currently tags `beatrice_32` for simba-3.2 in the shared catalog; `geffen_32` is the
voice used in Speechify's own API examples and synthesizes with simba-3.2 (200 + audio above).

## Error behaviour

- **Invalid credential**: `GET /v1/voices` with a bad key returns 401 → the Dograh save is rejected with the message in `02-invalid-key-rejected.png`. A 403 is likewise surfaced as an authorization error (commit `85691b98`); connection failures are treated as inconclusive so a provider outage cannot lock users out of saving a valid key.
- **Invalid voice/model combination**: provider returns 400 (see matrix); at runtime the adapter yields an `ErrorFrame` carrying the provider's status and message.
- **Unsupported sample rate**: the adapter raises at startup instead of mislabeling resampled audio (pipecat commit `6d6bbb1`, kept in `51b98b325`).
