# PiTranslate — Project Plan

**Status:** Planning  
**Target platform:** Raspberry Pi (ARM64, initially Pi 4)  
**Runtime:** Node.js, bare metal  
**Primary purpose:** Offline translation of caption text for media services

## 1. Purpose

PiTranslate is a small, local translation service for captions produced by speech-recognition systems such as Whisper. It provides an HTTP API that other media applications can use to translate caption segments without sending their content to an external translation provider.

PiTranslate is a dedicated machine-translation service, not a general-purpose LLM application. The initial implementation should favour compact translation models and predictable CPU inference.

## 2. Agreed language policy

Only these three source languages are accepted:

- Finnish (`fi`)
- English (`en`)
- Swedish (`sv`)

The initial configurable target-language set is:

- Finnish (`fi`)
- English (`en`)
- Swedish (`sv`)
- Russian (`ru`)
- Ukrainian (`uk`)
- Spanish (`es`)

A request to translate a language into itself must be rejected or explicitly treated as a no-op according to the API contract. Russian, Ukrainian, Spanish, and other languages are not supported as source languages in the initial scope. Do not silently add automatic source-language detection or arbitrary source languages.

The language list must be configuration-driven so that target languages can be added later without redesigning the service.

## 3. Technology direction

### Application

- Node.js application, using JavaScript or TypeScript (choose during implementation).
- Bare-metal installation on Raspberry Pi OS 64-bit.
- No Docker or Podman requirement.
- No Python application or Python runtime dependency.
- Managed by `systemd`.
- HTTP API for local applications on the trusted network.

### Translation runtime

The preferred first technology to evaluate is **Hugging Face Transformers.js**, using ONNX Runtime for local inference.

This is a technology choice to validate, not an assumption that every desired model is already available in a compatible format. Before committing to a model, verify:

1. The model supports the required language direction.
2. The model can be loaded by the selected Transformers.js version.
3. Its ONNX assets and operators work on ARM64 Linux.
4. CPU inference works on the target Raspberry Pi.
5. Its licence permits the intended organisational use and redistribution, if applicable.
6. Translation quality is acceptable on real caption material.

Investigate compact OPUS-MT/Marian models as initial candidates where compatible. Do not assume that a model published for Python Transformers can be used unchanged in Transformers.js. Model conversion or a compatible ONNX export may be necessary.

Keep the inference adapter behind an internal interface so the model runtime can be replaced if the preferred stack proves unsuitable. Do not introduce Python merely to preserve a particular model choice without first evaluating Node-compatible alternatives.

## 4. Deployment and filesystem layout

Use a conventional bare-metal service installation, for example:

```text
/opt/pitranslate/
  app/
  current/
  releases/              # optional, if the deployment convention uses releases
  models/                # downloaded model assets; persistent across app updates
  config/
    languages.json
  logs/                  # only if not using journald
```

The exact layout should follow the existing deployment conventions used by the other Node applications on the host.

- Run under a dedicated unprivileged service account.
- Keep model assets separate from application code and package updates.
- Use a Node.js version supported by the selected inference dependencies.
- Use `systemd` for startup, restart, and service supervision.
- Log to stdout/stderr so journald can collect service logs.
- Do not expose the API publicly by default.

Model downloads and installation are an explicit provisioning/update operation. Normal translation requests must work offline and must never trigger an implicit download.

## 5. Proposed architecture

```text
Caption producer (e.g. Whisper)
          |
          | HTTP request: source language, target language, text
          v
      PiTranslate
          |
          +-- Request validation
          +-- Language policy / pair registry
          +-- Translation adapter
          |       |
          |       +-- Selected local model
          |       +-- CPU inference
          |
          +-- Response with translated text
          |
          v
Caption consumer / media pipeline
```

Keep the service independent of Whisper, MediaMTX, S3, and any particular caption format. Its core responsibility is translating text. Caption timestamp and cue management should remain in the calling media application unless a later requirement justifies a dedicated caption endpoint.

## 6. API proposal

Use a small versioned HTTP API. Exact framework and schemas can be selected during implementation.

### Health

`GET /health`

Returns service status and whether the configured models are loaded and ready. Do not expose filesystem paths or sensitive runtime details.

### Capabilities

`GET /v1/languages`

Returns supported source languages, configured target languages, and installed translation directions. Distinguish configured languages from directions for which a usable model is actually installed.

### Translate

`POST /v1/translate`

Example request:

```json
{
  "source": "fi",
  "target": "en",
  "text": "Tervetuloa jumalanpalvelukseen."
}
```

Example response:

```json
{
  "source": "fi",
  "target": "en",
  "translatedText": "Welcome to the service.",
  "model": "configured-model-id"
}
```

The API should validate source and target codes, reject unsupported directions, impose a configurable text-size limit, and return useful machine-readable errors. Do not return a successful translation when no model is installed for the requested direction.

A batch endpoint may be added if needed for caption cues, but first establish reliable single-request behaviour.

## 7. Caption-specific translation behaviour

Caption text is not always a complete sentence. The service or its caller should support translating context windows containing adjacent cues where this improves quality.

Important requirements:

- Preserve the original caption text; translations are additional tracks, not replacements.
- Preserve cue ordering and timestamps in the media pipeline.
- Avoid translating every short fragment independently when adjacent context is available.
- Do not merge or split cues in a way that loses timing alignment.
- Keep paragraph/context grouping separate from the core translation API initially.
- Test names, religious vocabulary, biblical references, spoken Finnish, and sentence fragments.
- Do not claim that machine translation is a verified transcript or authoritative translation.

A future caption-aware endpoint could accept cue IDs, timestamps, and text, and return translations associated with the same cue IDs. It is not required for the first proof of concept.

## 8. Performance and resource goals

The first deployment target is a Raspberry Pi 4 with a 64-bit operating system. Confirm the actual RAM size and OS before finalising model choices.

Initial goals:

- CPU-only operation.
- No GPU requirement.
- Modest memory footprint suitable for a Pi hosting other services.
- Load each configured model once and reuse it.
- Avoid loading every language-pair model simultaneously if memory is constrained.
- Measure cold-start time, resident memory, translation latency, and throughput.
- Test realistic caption-sized inputs as well as longer context windows.

Do not set hard performance promises before benchmarking. If the Pi 4 cannot meet the required latency, consider a Pi 5 or a separate host only after measuring the actual workload.

## 9. Model and language-pair validation

There are up to 15 useful directions among the six initial languages (six languages, excluding same-language pairs, but only the three agreed source languages are permitted).

The required direction matrix is:

| Source | Potential targets |
|---|---|
| Finnish | English, Swedish, Russian, Ukrainian, Spanish |
| English | Finnish, Swedish, Russian, Ukrainian, Spanish |
| Swedish | Finnish, English, Russian, Ukrainian, Spanish |

For each direction, record:

- Model identifier and revision.
- Direct or pivot translation, if applicable.
- Model and tokenizer format.
- Licence and attribution requirements.
- Download size.
- Runtime compatibility on ARM64.
- Peak memory and latency on the Pi.
- Quality notes from representative caption samples.

Prefer direct language-pair models. If a direction is unavailable, do not silently pivot through another language. A pivot strategy, if ever adopted, must be explicit and evaluated for quality loss.

## 10. Security and privacy

- Bind to localhost or a trusted private interface by default.
- Do not expose the service directly to the public internet.
- No external translation API calls.
- No telemetry or remote model lookup during normal operation.
- Do not log caption contents by default.
- Apply request-size limits and sensible timeouts.
- Run as an unprivileged user.
- Keep model downloads and updates separate from runtime requests.

If other hosts need access, use the existing private network and access-control conventions.

## 11. Implementation stages

### Stage 1 — Feasibility proof

- Confirm target Pi model, OS architecture, and Node.js version.
- Build a minimal Node.js test using Transformers.js and ONNX Runtime.
- Load one candidate model for Finnish-to-English.
- Verify offline inference on ARM64.
- Measure memory, startup time, and translation latency.
- Test real caption samples.

**Exit criterion:** A repeatable local translation works on the target Pi without Python or external services.

### Stage 2 — Model and language-pair survey

- Evaluate candidates for Finnish, English, and Swedish source languages.
- Identify available direct target directions for Russian, Ukrainian, and Spanish.
- Verify licensing and ONNX compatibility.
- Produce a tested model registry and document unsupported directions.

**Exit criterion:** Every advertised direction has a tested model, or is clearly marked unavailable.

### Stage 3 — Service implementation

- Implement the HTTP API and validation.
- Implement the model adapter and model registry.
- Load models lazily or according to a measured memory policy.
- Add health and capabilities endpoints.
- Add structured logs and graceful shutdown.
- Add automated tests for validation, errors, and model integration.

### Stage 4 — Bare-metal deployment

- Add installation and update instructions.
- Add a systemd unit and dedicated service account.
- Add explicit model provisioning instructions.
- Ensure application updates do not delete model assets.
- Verify restart, offline operation, and recovery after power loss.

### Stage 5 — Media pipeline integration

- Connect a caption producer/consumer through HTTP.
- Translate context-aware groups of caption text.
- Keep original and translated caption tracks separate.
- Test live and recorded-service workflows.
- Measure end-to-end delay and resource impact.

## 12. Out of scope for the initial release

- General-purpose LLM inference.
- Speech recognition or speech synthesis.
- Arbitrary source-language detection.
- Translation from Russian, Ukrainian, Spanish, or other non-approved source languages.
- Public multi-tenant translation API.
- Caption rendering, video compositing, or MediaMTX responsibilities.
- Automatic cloud fallback.
- Automatic model downloads during requests.

## 13. Open decisions

1. Exact Raspberry Pi model and RAM capacity.
2. Raspberry Pi OS version and Node.js version.
3. JavaScript versus TypeScript.
4. HTTP framework and port.
5. Exact model/runtime combination after ARM64 proof of concept.
6. Whether all models can remain loaded or must be loaded on demand.
7. Whether the service needs an API key on the private network.
8. Whether caption context grouping belongs in PiTranslate or in the calling media service.

## 14. Guiding principles

- Local-first and offline-capable.
- Node.js application and bare-metal deployment.
- Small dedicated translation models, not an LLM server.
- Explicitly supported language directions.
- Model compatibility and licence verified before adoption.
- Measure on the actual Raspberry Pi before promising performance.
- Keep the translation service independent of the media pipeline.
