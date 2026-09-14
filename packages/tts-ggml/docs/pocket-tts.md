# Pocket TTS

Pocket is an English, CPU-only TTS engine implemented in the Fabric speech
library. It streams PCM16 audio through the existing addon, inference plugin
and public SDK APIs. Python is used only to convert weights and benchmark the
upstream reference; it is not needed for synthesis in Fabric.

## Model bundle

Convert the supported Kyutai English 2026-04 checkpoint using
`engines/tts/scripts/convert-pocket-to-gguf.py` in qvac-fabric-speech.cpp.
The bundle contains `flow-lm.gguf`, `mimi.gguf`, `frontend.json` and `voice.gguf`.
The native repository's `engines/tts/docs/pocket-tts.md` documents conversion,
asset revisions, validation, CLI usage and the benchmark scripts.

The public checkpoint without voice cloning works with a prepared voice
embedding. Reference-WAV conditioning requires weights with a functioning
Mimi encoder; the public checkpoint with zero encoder weights is rejected
for that operation. Do not advertise voice cloning from those weights.

## Addon usage

```js
const TTSGgml = require('@qvac/tts-ggml')
const model = new TTSGgml({
  engine: 'pocket',
  files: { modelDir: '/absolute/path/to/pocket-bundle' },
  config: { language: 'en', useGPU: false },
  threads: 1,
  seed: 1234,
  temperature: 0.3,
  steps: 1
})
try {
  await model.load()
  const response = await model.run({ input: 'Hello from Pocket TTS in Fabric.' })
  for await (const chunk of response.iterate()) {
    // chunk.outputArray: Int16Array; chunk.sampleRate defaults to 24000.
    // Feed samples to an audio device or collect them in a WAV file.
  }
} finally {
  await model.destroy()
}
```

`npm run example:pocket -- /absolute/bundle "Hello from Fabric." /tmp/pocket.wav`
saves a playable WAV. The addon also accepts explicit `files.pocketFlowModel`,
`files.pocketMimiModel`, `files.pocketFrontend` and `files.pocketVoice` paths.

`run()` streams native chunks without sentence splitting, including when
`streamOutput: true` is requested. `runStream()` explicitly splits sentences;
`runStreaming()` accepts incremental text. `cancel()` invalidates pending text
and waits for native completion before another request is admitted.
A `run()` AbortSignal also stops native work. Invalid reload options or failed
replacement activation preserve the previously loaded model. Overlapping
lifecycle operations are rejected; `unload()` permits a later load/reload,
while `destroy()` is terminal.

## Public SDK usage

```js
import { loadModel, textToSpeech, unloadModel, close } from '@qvac/sdk'
const root = '/absolute/path/to/pocket-bundle'
const modelId = await loadModel({
  modelType: 'tts-ggml',
  modelSrc: `${root}/flow-lm.gguf`,
  modelConfig: {
    ttsEngine: 'pocket',
    language: 'en',
    useGPU: false,
    mimiModelSrc: `${root}/mimi.gguf`,
    frontendSrc: `${root}/frontend.json`,
    voiceSrc: `${root}/voice.gguf`,
    threads: 1,
    seed: 1234,
    outputSampleRate: 24000
  }
})
try {
  const response = textToSpeech({ modelId, text: 'Hello from Fabric.', stream: false })
  const pcm = await response.buffer
  await response.done
  // pcm contains PCM16 samples at the configured output sample rate.
} finally {
  await unloadModel({ modelId, autoClose: false })
  await close()
}
```

Companion sources use the standard model-source resolver, including local paths
and descriptors. For reference conditioning supply `referenceAudioSrc` instead
of `voiceSrc`. The schema requires exactly one. Each model admits one active
TTS request; queued requests wait until cancellation and native callbacks drain.

## Supported controls

English only; CPU only; `threads` defaults to one per worker (two workers).
`seed` accepts unsigned 32-bit integers. `temperature`, `steps`, `nCtx`,
`maxTokens`, `noiseClamp`, `eosThreshold`, `framesAfterEos` and
`outputSampleRate` are validated before native loading. Output resampling
supports 8–192 kHz. Other engines' sampling, emotional, speed, GPU and
LavaSR settings are rejected instead of silently ignored.

Fabric defaults to one sampling step, matching the native CLI and upstream
reference, including SDK calls that omit `steps`. An explicit `steps: 4` (or
addon `numInferenceSteps: 4`) offers a quality option at additional compute
cost; use `--steps 4` for an equivalent native CLI comparison.

In a listening comparison of six prompts from 0.88 to 70 seconds, the listener
reported an artifact on “speech” only in the one-step render of “Hello! We can
generate speech with Fabric.” The listener also heard the artifact in the
upstream PyTorch one-step reproduction with matching random inputs, while the
four-step render sounded clean. The faster native build and earlier build
produced nearly identical PCM, and matched-noise PyTorch synthesis closely
matched Fabric. These findings support an upstream sampling artifact for this
take, not a Fabric-specific one-step implementation bug. Four steps mitigated
this example; they are not a guarantee against all artifacts or a port fix.

On an Apple M2, the paired renders took about 20–25% longer at four steps,
while remaining faster than playback (the 70-second passage took 16.0 seconds
versus 13.0 seconds with one step, excluding model loading). These were single
renders after warmup, rather than repeated benchmark measurements.

### Why choose four steps for audio quality?

Pocket's flow sampler transforms noise into the next audio latent. One step
predicts a single update over the full interval from 0 to 1. Four steps use
four quarter-interval updates, evaluating the flow network again on the
updated latent each time. This gives the model intermediate refinement
opportunities instead of relying on one full-interval prediction. It is a
mechanistic reason to try additional steps when a take has an artifact, not
proof that the reported sound was caused by a particular numerical error.

The recommendation here is supported by the controlled listening result:
the same text, voice, seed and temperature produced an audible artifact with
one step in both Fabric and upstream, while the four-step take sounded clean
to the listener. Use `steps: 4` when avoiding this artifact matters more than
minimum generation latency. Keep `steps: 1` for the upstream performance
default. The other eleven files in the listening set had no reported artifact;
this is evidence of an improvement for one take, not a general quality score.

Only the flow sampling stage repeats; text/voice conditioning and Mimi audio
decoding are not each run four times. That is why the observed total latency
increase was about 20–25%, rather than fourfold:

| Passage | One-step generation | Four-step generation |
| --- | ---: | ---: |
| Original sentence (2.64 s audio) | 0.46 s | 0.56 s |
| Extended story (69.92 s audio) | 13.03 s | 15.96 s |

## Validation

Build native prebuilds using the package's pinned vcpkg registry, then build
TypeScript in `@qvac/tts-ggml`, `@qvac/inference` and `@qvac/sdk`.
Set `QVAC_POCKET_MODEL_DIR` to the converted bundle for real-model tests:

- In the addon: `npm run test:pocket`; set `QVAC_POCKET_AUDIO_OUTPUT` to save WAV.
- In inference: `npm run test:pocket` (schema, lifecycle and real-plugin tests).
- In SDK: `npm run test:pocket:node` (public APIs over a real Bare worker/socket).

The SDK test requires model assets rather than skipping. Its owned temporary
configuration, cache and worker are bounded by a process supervisor. The
inference and addon real-model tests skip when their model variable is absent;
pass it explicitly when validating synthesis.

The merged native source passes nine Pocket CTests (124 enabled CPU CTests
in total). The September 14 macOS package passes 295 addon unit tests / 928
assertions, 64 Node package/build tests, and the real addon integration test
with 124 assertions, including the 100-frame EOS-tail regression. Focused
inference tests and four public SDK transport tests generate real audio.
The same pinned package builds for iOS Simulator; the Bare Kit worklet passes
65 assertions and produces valid 24 kHz PCM (Bare 1.29.4, iOS 18.6), including
the EOS-tail regression. Assertion totals can vary with streamed chunk sizes.
Adversarial review covered native math/asset validation in the speech repository
and addon request/lifecycle races, cancellation, reload and test cleanup here.
Audio checks cover sample validity and clipping; these are not subjective
naturalness ratings.

The full current inference TypeScript build still reports two existing type
errors in unchanged `safe-fetch.ts`. Focused Pocket compilation uses
`noEmitOnError` and passes; the whole SDK compiles. Physical iOS / Android audio
and mobile SDK transport validation remain outside the measured coverage.
The unrelated Supertonic fit-test fix was merged separately in
[speech PR #242](https://github.com/tetherto/qvac-fabric-speech.cpp/pull/242)
and is included in the merged native source.

## Dependency pins and measured performance

The consumer pins registry baseline and reference to
`bdfafc83cb0ba12b54c3eecd09b6936911f4d9e0`, selecting speech-cpp
2026-09-14#1 and ggml-speech 2026-09-14. This includes the follow-up fix for
Linux/Android dynamic CPU backends: Pocket's metadata-only memory planner now
resolves `ggml_graph_plan` through the selected backend registry. The original
direct import left an unresolved symbol in the addon. Synthesis arithmetic is
unchanged. The source fixes are
[ggml #92](https://github.com/tetherto/qvac-ext-ggml/pull/92) and
[speech #251](https://github.com/tetherto/qvac-fabric-speech.cpp/pull/251).
The explicit registry reference is needed until the follow-up version database
merges; the original native PR #240 and registry PR #364 are already merged.

Static/dynamic CPU planner regression tests and Pocket graph-memory, fit and
engine/audio tests pass locally with the follow-up sources. The broader addon
validation above used the preceding September 14 pin; Linux/Android prebuild
validation for this fix is tracked on Fabric PR #4396.

The npm addon release is a separate dependency: published `@qvac/tts-ggml@0.9.0`
does not contain Pocket. The workspace builds and audio tests above use the
checkout's wrapper and locally built native prebuilds. Package-local Bun installs
in SDK Pod CI resolve the published addon and currently fail on `ENGINE_POCKET`
and `pocketFlowModel`. Before shipping the SDK integration, release the Pocket
addon with matching platform prebuilds and raise the inference/SDK dependency
floors to that version. The current `^0.9.0` ranges do not establish Pocket support.


The recorded September 10 native benchmark linked the exact libraries installed
by the previous speech-cpp 2026-09-10#1 registry pin (`470e678f` source) and
normal addon build. It has not been rerun against the September 14 package;
current validation above covers builds and functional audio checks. It uses Apple M2 CPU, one thread per worker,
two workers, **one sampling step**, one warmup and three measured runs per
prompt, matching Fabric's default. Medians:

| Prompt | Upstream generation | Fabric generation | Upstream / Fabric audio length |
| --- | ---: | ---: | ---: |
| Short | 0.94 s | 0.98 s | 5.52 / 5.52 s |
| Numbers | 1.23 s | 1.42 s | 7.36 / 7.44 s |
| Punctuation | 1.18 s | 1.24 s | 6.88 / 6.72 s |
| Accented names | 1.23 s | 1.38 s | 7.20 / 7.36 s |
| Long | 6.16 s | 6.55 s | 36.32 / 36.48 s |

Fabric RTF is 0.178–0.191 (about 5.2–5.6× real time), versus upstream
0.167–0.172. Generation time is 5–16% above upstream across these prompts.
First audio arrives in 74–116 ms in Fabric versus 53–69 ms upstream.
Loading takes 0.23–0.31 seconds during the matrix; an earlier first launch
measured 9.61 seconds, so warmed launch timings do not establish cold-cache
startup performance. The previous manual development build was slower; these
final numbers supersede its approximately 2× slowdown.
These are native steady-state measurements; public SDK startup/IPC latency
is additional. Seeds do not produce equivalent randomness across the two
implementations at temperature 0.3. All ten WAVs pass signal checks. ASR
recovers short and punctuation prompts exactly, shares the long prompt's
single assistance/assistants mismatch, and finds a contraction difference in
the accented-name prompt. The earlier zero-temperature follow-up resolves
that contraction and gives matching transcriptions. No subjective listening
score or equal-naturalness claim is made.


To reproduce the packaged iOS worklet after building the simulator prebuild,
run `python3 scripts/run-pocket-ios-worklet.py --help`. The runner needs a
booted arm64 simulator, the bundle, Bare Kit, bare-pack >= 2 and bare-link >= 3.
It resolves pnpm package paths canonically, uses the actual TTS framework,
and leaves a fresh output directory containing logs, result JSON and WAV.
It neither boots/shuts down a simulator nor edits an app checkout. Functional
coverage runs with brittle's optional Node-only coverage reporter deferred.
