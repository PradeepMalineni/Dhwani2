# Dhwani 1.0 — AGENTIC.md

## Mission
Build an offline-first, privacy-preserving, bidirectional sign-language communication application: sign → text → speech and speech → sign. The primary target is a battery-efficient Android mobile device; use C++17 for portable inference and motion processing. Internet access must never be necessary for the supported MVP workflows.

## Non-negotiable engineering principles
1. Offline by default: airplane-mode acceptance tests for all supported workflows. No network permissions required for the core app.
2. Privacy: camera/audio processed on device; no raw recording persistence or upload by default. Explicit consent for recording and sharing.
3. Accessibility: large controls, captions, audio output, clear confidence/uncertainty cues, no fabricated translation on low confidence.
4. Performance: instrument latency, peak RAM, app/model size, CPU/GPU/NPU utilization, thermal behavior and battery drain on actual target hardware. Optimize based on measurements, not guesses.
5. Reproducibility: version datasets, subject-level splits, labels, preprocessing, model artifacts and evaluation metrics.
6. Scope: choose ONE sign language and locale for MVP (provisional: Indian Sign Language and English speech). Confirm with domain experts and native signers.
7. Sign languages are full languages with their own grammar. Never assume word-for-word spoken-to-sign translation is linguistically correct.
8. Prefer portable C++ runtime; Android camera/audio/UI may use Kotlin/Java platform APIs with JNI bridges. Training can use Python and PyTorch.
9. Keep cloud/frontier models optional and development-only initially: research, annotation assistance, test generation, distillation. Never send user video/audio to a remote model without informed consent.

## MVP definition and acceptance criteria
MVP A: Given a curated video of one of 20–30 isolated signs, predict its label and play offline synthesized speech (or pre-bundled audio). Show confidence and an unknown/abstain result when uncertain. Provide an Android live-camera demonstration using the same inference core.
MVP B: Capture a short spoken utterance from a constrained vocabulary, transcribe locally, map supported phrases to validated sign-video clips, and play the corresponding sequence offline. Display a fallback message for unsupported sentences instead of claiming full translation.
Initial engineering targets (hypotheses, revise after baseline): 10–15 FPS end-to-end processing on a named mid-range Android reference phone; <200 ms p95 inference time per processed window (excluding the window accumulation delay); <150 MB peak incremental RAM for inference; <50 MB total bundled recognition model assets; zero network calls in airplane mode. Track sustained 20-minute battery/thermal behavior, but set battery thresholds after measuring baseline. Accuracy gate: ≥90% macro F1 on held-out signer evaluation for the limited isolated-sign vocabulary, with per-class reporting; do not declare success based on frame-random splits. If the target is not reached, report the measured gap honestly.

## Decisions to record before coding
- Target sign language and spoken language(s), community reviewers and accessibility requirements.
- Data license, consent, provenance, signer count, class balance and recording conditions.
- Reference Android device, minimum Android API level and device performance envelope.
- MVP vocabulary and approved sign variants.
- Visual output for speech → sign: approved real sign clips first; avatar later.
- Choice of landmark extraction: benchmark MediaPipe/other on-device solutions, including binary size, runtime, licensing and battery; do not assume Python desktop extraction is directly portable.

## Repository structure
```
dhwani/
  AGENTIC.md
  README.md
  docs/{product,architecture,decisions,experiments,ethics}/
  data/{README.md,schemas,manifests}/       # metadata only; no personal videos in git
  tools/{dataset_audit,annotation,preprocess,benchmark}/
  training/{configs,models,experiments,export}/
  cpp/{include,src,tests,benchmarks}/
  android/{app,benchmark}/
  assets/{sign_clips,labels,tts}/
  tests/{fixtures,integration,offline}/
  scripts/
  .github/workflows/
```

## Canonical data contract
A sample manifest row must include: sample_id, sign_language, gloss_or_label, signer_pseudonym, session_id, capture_device, fps, width, height, duration_ms, video_uri, annotation_source, consent_status, license, split and quality_flags. Keep video files outside version control.
Landmark tensor: `[T, J, C]`, where J and C are versioned schema constants, with timestamps, landmark validity/confidence masks, camera mirroring/orientation and coordinate system. Do not silently mix normalized image coordinates and metric/world coordinates.
Derived features: relative positions anchored to stable body landmarks, bone vectors/distances, velocity and acceleration computed with actual delta time, motion magnitude/direction, optional handshape and face cues where licensed and consented. Use smoothing and masking; never divide by zero or infer missing landmarks as real zeros.

## Workstream 1 — Dataset and baseline
1. Inventory all existing videos, label availability, duplicates, signers, resolutions, consent and licensing. Produce `docs/experiments/dataset-audit.md` and `data/manifests/dataset.csv`.
2. Select 20–30 signs only after confirming at least several signers and adequate independent examples per class. If data is insufficient, plan additional capture before promising model quality.
3. Define signer-disjoint train/validation/test splits; group near-duplicates and same recording sessions. Freeze the test set.
4. Produce a reproducible landmark extraction CLI with clear schema and failure logging. Validate on representative lighting, camera orientations and hand occlusions.
5. Train simple baselines: landmark feature statistics + Random Forest or small MLP; then compact TCN/GRU. Include a no-motion or position-only ablation.
6. Compare optional microtransition features (position, relative geometry, velocity, acceleration, direction, duration/persistence) using ablations. Avoid derivative noise amplification and report when higher derivatives fail to help.
7. Report macro F1, per-class precision/recall, confusion matrix, signer-generalization, unknown rejection and calibration. Select the smallest model meeting the quality gate.
8. Export candidate to ONNX or TFLite/LiteRT as supported by the chosen mobile runtime; validate numerical parity on held-out fixtures.

## Workstream 2 — Portable C++ engine
1. Define stable APIs: `Frame`, `LandmarkSet`, `MotionWindow`, `Prediction`, `ModelInfo`, `Recognizer`.
2. Implement timestamp-aware ring buffer, normalization, feature extraction, valid-mask handling and window segmentation.
3. Provide pluggable runtime adapter for ONNX Runtime Mobile or LiteRT/TFLite C API, benchmark binary size and CPU delegates before choosing.
4. Implement temporal smoothing, minimum confidence, hysteresis and an `UNKNOWN` class/abstention mechanism. Avoid speaking the same sign repeatedly across adjacent windows.
5. Expose JNI methods to initialize model, ingest landmarks, infer, reset and release resources.
6. Unit test against golden fixtures and compare Python/C++ feature parity within explicit tolerances.
7. Build with CMake and Android NDK; enable sanitizer tests on desktop and optimize Android release builds.

## Workstream 3 — Android sign → speech
1. Minimal accessible Android UI with camera permission, start/stop, preview, recognized text, confidence indicator and speech playback.
2. Use CameraX or platform camera API, perform landmark detection locally, pass structured landmarks into C++ via JNI.
3. Decouple camera capture, landmark extraction and inference with bounded queues and frame dropping to prevent backlog.
4. Speak accepted predictions with an offline-capable Android TTS voice; test that the voice is installed and works in airplane mode. Provide bundled pre-recorded audio fallback for MVP vocabulary.
5. Add opt-in diagnostic overlay and local benchmark logs; do not save images by default.
6. Validate against previously unseen signers and different environments, with explicit consent.

## Workstream 4 — Android speech → sign
1. Select a small, validated phrase set and have qualified signers review mappings and variants.
2. Benchmark an offline ASR engine (e.g. whisper.cpp with an appropriately sized model or Vosk) on the reference phone for word error rate, memory, latency and battery.
3. Implement local phrase/intent mapping with deterministic handling of unsupported phrases. Do not assume English word order is sign-language grammar.
4. Bundle consented, licensed sign-video clips; display matching clips and captions. Define transitions between clips and get user feedback on intelligibility.
5. Measure phrase-level comprehension with sign-language users. Defer generative avatars and open-ended translation until linguistic and evaluation foundations exist.

## Workstream 5 — Performance and offline verification
- Establish cold-start and warm-start metrics, camera-to-caption latency, ASR latency, CPU/GPU/NPU, model size, RAM, FPS, thermal throttling and energy use over a sustained 20-minute test.
- Compare sampling rates, frame sizes, landmark detectors, quantization (FP32/FP16/INT8), thread counts, window sizes and runtime delegates. Measure accuracy regressions after each optimization.
- Run all user flows in airplane mode, with no sign-in, and verify no network dependencies or unbundled model downloads.
- Test device rotation, low light, camera loss, microphone denial, background/foreground transitions and app recovery.

## Execution sequence — agent-sized tasks
### Sprint 0: Discovery and contracts
S0.1 Create repo scaffold, ADR template and CI checks.
S0.2 Audit existing videos and produce reproducible manifest.
S0.3 Freeze MVP vocabulary, language, signer split and quality gates.
S0.4 Select Android reference device and document budgets.
**Exit:** Data audit, frozen split, approved MVP scope, buildable skeleton.

### Sprint 1: Training baseline
S1.1 Implement landmark extraction and schema validation.
S1.2 Implement timestamp-aware feature pipeline.
S1.3 Train position-only and compact temporal baselines.
S1.4 Evaluate signer-disjoint test set and publish metrics.
**Exit:** Reproducible model with honest evaluation, artifact checksum and prediction CLI.

### Sprint 2: C++ parity and Android integration
S2.1 Implement portable C++ preprocessing and unit tests.
S2.2 Integrate mobile inference runtime and benchmark.
S2.3 Implement Android camera → landmarks → JNI → prediction.
S2.4 Implement confidence gating and offline TTS.
**Exit:** Offline sign → speech demo on physical Android device.

### Sprint 3: Reverse direction
S3.1 Benchmark offline ASR alternatives.
S3.2 Create signer-validated sign-clip phrase pack.
S3.3 Implement speech → local text → supported phrase → clip playback.
S3.4 Test comprehension and unsupported-input behavior.
**Exit:** Offline constrained-vocabulary speech → sign demo.

### Sprint 4: Hardening and release candidate
S4.1 Run sustained device benchmarks and quantization comparisons.
S4.2 Run signer-diverse, accessibility and robustness tests.
S4.3 Complete consent, license, privacy and release checklist.
S4.4 Produce reproducible APK, model cards and demo script.
**Exit:** MVP release candidate with documented limits and measured performance.

## Agent operating rules
- Work on one numbered task at a time. Before editing, inspect the repository and read related ADRs.
- For each task, provide: goal, assumptions, files to change, acceptance tests, risks and rollback path.
- Make small, reviewable commits. Never silently alter data schemas, label mappings or split assignments.
- Execute tests and include actual command output; never claim tests ran when they did not.
- Keep a running `docs/experiments/experiment-log.md` with config, data version, seed, metrics and artifacts.
- When blocked by unavailable data, device or user decision, create a clear decision request rather than inventing results.
- Do not introduce cloud services, telemetry, heavy dependencies or a large foundation model without a measured benefit and explicit approval.
- Update README and ADRs for decisions that change architecture or behavior.

## Agent first instruction
Read `AGENTIC.md`. Begin with S0.1 and S0.2 only. Inspect the current repository and list available video datasets without modifying source data. Create the scaffold and a dataset audit script that reports counts, metadata, label completeness, duplicates and provenance. Do not train a model until the user approves the audit, vocabulary and signer-disjoint split.

## Future architecture (not MVP commitments)
Continuous sign segmentation, sign-language grammar translation, signer adaptation, on-device language models, optional cloud augmentation, sign avatars, external camera/wearable interfaces, wall projection, low-power accelerator hardware and multilingual support. Treat each as a separate validated research milestone.
