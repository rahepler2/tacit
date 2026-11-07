# 17 · Speech service

A Python service that provides streaming speech-to-text, end-of-turn detection and text-to-speech, fully on-prem.

## 1. Interface

WebSocket `/v1/stream` (internal only; the API relays browser audio).

Client (API) → service:
| type | payload |
|---|---|
| `config` | `{session_id, language, vocabulary: string[], vad: {min_silence_ms, threshold}, diarize: bool}` |
| *(binary)* | PCM16 mono 16 kHz frames |
| `tts` | `{request_id, text, voice, language, rate?}` |
| `tts.cancel` | `{request_id}` |
| `flush` | `{}` (force finalization) |

Service → client:
| type | payload |
|---|---|
| `partial` | `{text, start_ms}` |
| `final` | `{text, start_ms, end_ms, confidence, words?: [{w, s, e, p}], speaker?}` |
| `eot` | `{at_ms}` (end-of-turn) |
| `speech_start` | `{at_ms}` (for barge-in) |
| `tts.audio` (binary, prefixed with request_id header frame) / `tts.done` | |
| `error` | `{code, message}` |

## 2. Requirements

- **SPH-001** STT: faster-whisper (model size configurable; GPU if available) with streaming via VAD-segmented chunking. Partials at least every 500 ms while speech is active.
- **SPH-002** **Vocabulary biasing:** the `vocabulary` list (equipment tags, system names, people, acronyms from the entity registry and glossary, ≤ 200 terms) is used as initial prompt and hotwords. The API builds it per session from the campaign's top entities.
- **SPH-003** End-of-turn: Silero VAD with configurable minimum silence (default 900 ms), plus a short semantic guard (do not end turn if the final text ends with a conjunction or filler such as "and", "so", "um"; extend by 600 ms once).
- **SPH-004** Language pinned to the session's interview language. Auto-detect only if `language = "auto"`.
- **SPH-005** TTS: Piper (or a configured on-prem engine) with ≥ 1 stock voice per supported language. Streaming synthesis: first audio ≤ 400 ms after request for ≤ 60 words on reference hardware.
- **SPH-006** `tts.cancel` stops synthesis and playback streaming within 100 ms.
- **SPH-007** Diarization (group sessions): optional, using an on-prem diarization model. Outputs speaker labels; the API maps them to participants with facilitator confirmation.
- **SPH-008** No voiceprints or speaker embeddings persisted beyond the session's processing lifetime (REG-024). Diarization embeddings are held in memory only.
- **SPH-009** Reference WER targets measured on the synthetic-plus-pilot audio set (19): overall ≤ 12 %, entity-token accuracy ≥ 90 % with vocabulary biasing.
- **SPH-010** Health and metrics endpoints; GPU memory and queue depth metrics exported via OpenTelemetry.
