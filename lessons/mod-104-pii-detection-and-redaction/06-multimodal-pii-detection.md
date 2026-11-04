# Multimodal PII Detection

Text PII has had thirty years of tools; image / voice / document PII has had five, maybe ten. AI pipelines process all three. A screen-capture debugging tool that uploads screenshots to an LLM for "describe what went wrong" has leaked more PHI in production than most text pipelines. A call-centre analytics product that transcribes recordings uploads voice *and* transcript. An OCR'd medical PDF is a text pipeline — provided someone wrote the OCR step. This chapter is the reference for the four multimodal surfaces a mod-104-literate privacy engineer owns:

- Images — face / licence-plate / ID-document detection + blur.
- Audio — speaker-diarisation + PII-word redaction.
- OCR'd documents (PDFs, scanned forms) — OCR + the text pipeline.
- UI screen captures — a special case that composes all three.

Primary references:

- Presidio Image Redactor — [microsoft.github.io/presidio/image-redactor](https://microsoft.github.io/presidio/image-redactor/).
- OpenCV Haar / DNN face detectors — [docs.opencv.org/.../tutorial_py_face_detection](https://docs.opencv.org/4.x/d2/d99/tutorial_py_face_detection.html).
- MTCNN / RetinaFace — modern face detectors. [github.com/ipazc/mtcnn](https://github.com/ipazc/mtcnn), [github.com/serengil/retinaface](https://github.com/serengil/retinaface).
- YOLO / Detectron2 — generic object detection (useful for licence plates). [github.com/ultralytics/ultralytics](https://github.com/ultralytics/ultralytics), [github.com/facebookresearch/detectron2](https://github.com/facebookresearch/detectron2).
- OpenALPR — licence-plate recognition. [github.com/openalpr/openalpr](https://github.com/openalpr/openalpr).
- Tesseract — OCR. [github.com/tesseract-ocr/tesseract](https://github.com/tesseract-ocr/tesseract).
- `pyannote.audio` — speaker diarisation. [github.com/pyannote/pyannote-audio](https://github.com/pyannote/pyannote-audio).
- NIST SP 500-290 — biometric performance evaluation (useful for face-detector eval). [nist.gov/publications/nist-sp-500-290](https://www.nist.gov/publications/nist-sp-500-290).
- Google Cloud DLP image redaction — [cloud.google.com/sensitive-data-protection/docs/redacting-sensitive-data-images](https://cloud.google.com/sensitive-data-protection/docs/redacting-sensitive-data-images).
- GDPR Rec. 51 + Art. 9 — biometric data as special category.

## Why multimodal needs its own module

Three ways multimodal differs from text:

- **The detector is a vision / audio model, not a regex.** Faces are not regex-identifiable. Voice is not regex-identifiable. The detector is an ML model, which means the eval set is a labelled image / audio set, not a labelled text set.
- **The operator is domain-specific.** "Replace" on a face is `blur + box`; "replace" on an audio clip is `silence + bleep`; "replace" on an OCR'd span is a pixel overlay. Each needs a per-modality operator.
- **The sensitive categories are broader.** A face is biometric data under GDPR Art. 9 — special category. A voiceprint is biometric data. Even a hand (uniquely identifying) can be. The lawful-basis bar is higher; the DPIA bar is higher.

Multimodal is where privacy engineering meets computer-vision / speech engineering. Accept early: you are co-owning this surface with a vision / speech engineer.

## Image PII — faces

### The detector

Three usable face detectors, roughly in order of maturity / speed:

- **Haar cascade** (OpenCV's classic). Fast, CPU, famously high-FP under tilt / occlusion. Fine for the "redact every obvious face in a passport photo on an internal doc" job.
- **Single-Shot MultiBox Detector (SSD)** + a face-trained model. Better than Haar; still CPU-feasible.
- **MTCNN / RetinaFace.** Transformer-era accuracy. RetinaFace in particular is the common production choice; MTCNN remains a widely used reference.
- **YOLO-face** / **Mediapipe FaceDetection.** YOLO variants fine-tuned on faces; Mediapipe is Google's mobile-first offering.

The detector returns a list of bounding boxes `[(x1, y1, x2, y2, score)]`. Minimum pipeline shape:

```python
from retinaface import RetinaFace
import cv2

img = cv2.imread("meeting_screenshot.png")
faces = RetinaFace.detect_faces(img)   # dict of face_id -> {facial_area, score, landmarks}

for _, f in faces.items():
    x1, y1, x2, y2 = f["facial_area"]
    region = img[y1:y2, x1:x2]
    blurred = cv2.GaussianBlur(region, (51, 51), 0)
    img[y1:y2, x1:x2] = blurred

cv2.imwrite("meeting_screenshot_redacted.png", img)
```

Three operator choices and their reversibility:

- **Gaussian blur.** Large-kernel blur destroys enough detail to defeat casual recognition. Ineffective against targeted CV attacks (super-resolution / blur-inversion, some of which can partially recover faces from mild blur). *Irreversible under attacker budget.*
- **Pixelation / mosaic.** Equivalent to blur for most threats; aesthetically worse.
- **Solid colour box.** Fully destroys the underlying pixels. Irreversible. Visually harsh.
- **Face replacement** (DeepPrivacy, CIAGAN). Replaces with a synthetic face that preserves pose / lighting; the underlying identity is destroyed but the image remains visually realistic. Useful for synthetic training data; heavy to run.

**Pick an operator by threat model.** A training-corpus image headed to a model-provider's log should be solid-boxed; a UI screenshot for a bug report can be blurred. The DPIA records the choice.

### The eval set

- **Public labelled sets.** WIDER FACE for detection ([shuoyang1213.me/WIDERFACE/](http://shuoyang1213.me/WIDERFACE/)); FDDB ([vis-www.cs.umass.edu/fddb/](http://vis-www.cs.umass.edu/fddb/)); FaceForensics for the deepfake-generation-aware case.
- **Sampled real traffic.** Draw from your actual image traffic; label with bounding boxes.
- **Per-demographic slice.** NIST's FRVT (Face Recognition Vendor Test) documents face-detector bias across demographics; detector recall varies by skin tone / age / mask-wearing. Slice the eval per demographic and report per-slice recall. See [pages.nist.gov/frvt](https://pages.nist.gov/frvt/).

A face detector with 95% overall recall but 70% recall on a specific demographic slice is a bias finding, not a tuning finding. The DPIA cites the per-slice numbers.

### Failure modes

- **Occlusion / angle.** Side profiles, partially occluded faces, masks. RetinaFace handles better than older detectors; still imperfect.
- **Small faces.** Faces at <20px tend to miss. Pad the eval with small faces explicitly.
- **Group photos.** Overlapping bounding boxes; non-maximum suppression can drop valid ones. Re-run with a lower NMS threshold if you see this.
- **Faces in reflections / monitors-in-the-background.** A reflection in a window; a face on a laptop screen inside the photo. Detectors often miss these; attacker-sophistication question.

## Image PII — ID documents and licence plates

Treat ID documents and licence plates as their own detection surface:

- **Licence plates.** YOLO fine-tuned on a plate dataset; OpenALPR as a managed detector. Operator: solid-box + blur. The eval set is a labelled plate dataset.
- **ID documents (passports, driving licences, ID cards).** Specific recognisers (face box + holographic patterns). Several managed vendors (Microblink, Veriff, Jumio, Private AI) ship ID-document detectors; fine-tuned YOLO is the self-host path.
- **Screen bezels / monitors in frame.** A laptop screen in a photograph is a secondary surface — OCR the screen region; run the text pipeline.
- **Handwritten signatures.** Treated as biometric; blur or box.

Record each detector as a named recogniser with a measured per-class recall.

## Audio PII — the speech surface

### What counts as audio PII

Two shapes:

- **Content PII.** The spoken content contains PII — "my social is one two three four five six seven eight nine". Transcription + text pipeline handles this.
- **Voiceprint PII.** The speaker's voice is itself a biometric that identifies the individual (Art. 9 special-category). A redacted transcript that leaves the raw audio in place has only partially de-identified.

The AI data path's audio choke-points:

- **Transcription ingest** — a call-centre recording is transcribed before storage.
- **Agent / voice-assistant input** — a user utterance is streamed to an STT model.
- **Audio embeddings** — an audio clip is embedded for similarity search (voice-search, audio-QA).

### Transcription + text pipeline

The simplest shape: transcribe (Whisper / Azure STT / AWS Transcribe / Google Speech-to-Text), run the text pipeline on the transcript, and re-emit the transcript with tokens.

```python
from faster_whisper import WhisperModel

model = WhisperModel("large-v3", device="cuda", compute_type="float16")
segments, info = model.transcribe("call_recording.wav", word_timestamps=True)

transcript_text = " ".join(seg.text for seg in segments)
pii_results = analyzer.analyze(text=transcript_text, language="en")
redacted = anonymizer.anonymize(text=transcript_text, analyzer_results=pii_results)
```

This covers *content* PII.

### Voiceprint and speaker diarisation

To redact the raw audio, you need to know *where in the waveform* the PII content is, then apply an audio operator (bleep / silence / substitute). Steps:

1. **Speaker diarisation** — partition the audio into "who speaks when" segments. `pyannote.audio` is the current open-source reference.
2. **Word-level timestamps** — Whisper's `word_timestamps=True` outputs per-word start / end times in the waveform.
3. **PII span → waveform span** — map the text spans the analyzer found back to the word timestamps, producing a list of `(start_ms, end_ms)` audio spans.
4. **Audio operator** — mute, bleep (overlay a tone), or substitute (replace the audio with TTS of the redacted-text) each span.

Minimum shape:

```python
pii_audio_spans = []
for segment in segments:
    for word in segment.words:
        for res in analyzer.analyze(text=word.word, language="en"):
            pii_audio_spans.append((word.start, word.end))

audio = AudioSegment.from_wav("call_recording.wav")
for (start_s, end_s) in pii_audio_spans:
    silence = AudioSegment.silent(duration=int((end_s - start_s) * 1000))
    audio = (
        audio[:int(start_s * 1000)]
        + silence
        + audio[int(end_s * 1000):]
    )
audio.export("call_recording_redacted.wav", format="wav")
```

### Voice-conversion / voiceprint-anonymisation

Where the *voice itself* is sensitive, the operator is heavier:

- **Voice conversion** — re-synthesise the speech in a different voice (e.g. a neutral TTS) while preserving linguistic content. Research-level, not a commodity.
- **Vocal pitch / formant shifting** — simple perturbation that defeats casual recognition; does not defeat sophisticated voiceprint matching.
- **Transcript-only release** — ship the redacted transcript; do not ship the audio. Simplest and strictest.

Document the operator choice in the DPIA. The VoicePrivacy Challenge ([voiceprivacychallenge.org](https://www.voiceprivacychallenge.org/)) is the research reference for evaluation.

### Audio eval set

- **Labelled speech corpora with PII annotations.** Publicly scarce; often sourced via vendor ground-truth or internal labelling.
- **Switchboard / Fisher** for diarisation evaluation.
- **Real sampled traffic with consented recordings** for the dominant case.

Measure per-entity recall on the transcript *and* alignment-error on the audio-span mapping. A 95%-recall transcript detector that places the audio span 300ms too late is a 300ms leak.

## OCR'd documents — PDFs, scanned forms, images of text

The PDF / scanned-form case is often the easiest unlock: run OCR, run the text pipeline, apply redaction *on the image*.

### Presidio Image Redactor

Presidio ships `ImageRedactorEngine` that wraps Tesseract + analyzer:

```python
from presidio_image_redactor import ImageRedactorEngine
from PIL import Image

engine = ImageRedactorEngine()
image = Image.open("scanned_form.png")
redacted_image = engine.redact(image=image, fill="black")
redacted_image.save("scanned_form_redacted.png")
```

Steps under the hood:

1. OCR extracts text with per-word bounding boxes.
2. The Presidio analyzer finds PII spans in the OCR output.
3. The redactor draws boxes over the bounding boxes of the PII words.

Three failure modes with this shape:

- **OCR errors cascade.** A low-quality scan where Tesseract misreads "SSN 123-45-6789" as "SSN I23-4S-6789" may not match the `US_SSN` recogniser's pattern, leading to FN. Mitigation: a loose pattern + context-word recogniser catches OCR-like strings; QA the OCR quality on the eval set.
- **Multi-line entities.** An address spanning three lines of OCR output — each line's text may or may not be analyzable as a span. The analyzer sees the lines concatenated; the bounding-box list is per-word. Boxes need to cover all of the words of a matched span, not just the one that triggered.
- **Non-text PII.** Faces / signatures / barcodes in the PDF. Redact separately with the image pipeline.

### Multi-page PDFs

For multi-page PDFs, process page-by-page and preserve the PDF structure (PyMuPDF / pdfplumber for text extraction; `pdf2image` for rendering to images). For *searchable* PDFs with embedded text, prefer the text-layer extraction + overlay-box-on-rendered-image approach — it is both faster and more accurate than re-OCR.

### Managed alternatives

- Google Cloud DLP's `redactImage` + `inspect` endpoints handle this natively.
- AWS Textract + Comprehend PII in a pipeline.
- Azure AI Document Intelligence + Language PII.
- Private AI's document pipeline.

Build-vs-buy per chapter 05.

## UI screen captures — the composite case

Debugging tools / support tools that ingest UI screenshots are a special multimodal case because they contain:

- OCR'd text (menus, buttons, user-filled fields).
- Faces (video-conference tile; avatars).
- Document thumbnails (uploaded PII docs rendered as preview tiles).
- Decorative elements that *look* like PII (lorem-ipsum SSN-shaped strings).

A production-grade pipeline:

1. **OCR the whole screenshot** → text + per-region bounding boxes.
2. **Face detection pass.** Blur / box any face bounding boxes.
3. **ID-doc / plate detection pass.** Specialised detectors for these shapes.
4. **Text PII pass** on the OCR output; redact boxes on the corresponding bounding boxes.
5. **URL / query-string redaction.** URLs in captured address bars commonly contain tokens or PII in the query string; a dedicated recogniser for URL query-param-values catches these.
6. **"Region of suspicion" alert.** Where a region is likely a credential (e.g. a password field) but the OCR fails, over-redact — box the region conservatively.

The one-shot Presidio Image Redactor is a reasonable starting point; a UI-screen-capture pipeline usually ends up with its own composition of the above.

## The compliance layer

GDPR, HIPAA, and biometric-law obligations differ sharply across multimodal:

- **Face data is biometric data.** GDPR Art. 9 — special category; one of the Art. 9(2) lawful bases required. CCPA / BIPA / various US state biometric privacy laws.
- **Voice data is biometric data** when used for identification (speaker-ID). Content transcription without speaker-identity use is usually not biometric in the strict regulatory sense; check the jurisdiction.
- **HIPAA Safe Harbor § 164.514(b)(2)(i)** lists "full-face photographs and any comparable images" as one of the 18 identifiers. Chapter 08 details.
- **Illinois BIPA, Texas CUBI, Washington "My Health My Data Act"** impose explicit opt-in consent requirements for biometric capture + processing.
- **EU AI Act** categorises real-time remote biometric identification systems as "high-risk" and remote biometric-categorisation (gender / race inference from face) as "prohibited" in most public contexts.

Multimodal PII pipelines are often dual-use: the same face-detector that protects individuals by redacting their faces *can* be used to recognise them. Document both uses in the DPIA; prohibit the identification use by access-control if it is not the authorised purpose.

## Latency and cost accounting

Multimodal detectors are an order of magnitude more expensive than text:

- A GPU-RetinaFace pass on a 1080p image: ~100ms per frame on a modern GPU; slower CPU.
- Whisper large-v3 on a 10-minute audio file: tens of seconds; large memory.
- OCR of a 10-page PDF: seconds per page on CPU.

Three implications:

- **Batch, batch, batch.** Multimodal rarely runs on the synchronous prompt-side path. It runs at ingest / retrieval / after-the-fact log scrubbing.
- **Autoscale on queue depth, not request rate.** The variance per request is high; a batch autoscaler is appropriate.
- **Cost is a first-class matrix column.** The chapter-05 build-vs-buy matrix gains rows for multimodal units of work; the hybrid answer is often "Presidio text + managed multimodal".

## A reference pipeline

A production AI product with multimodal inputs ends up with a pipeline shape like:

```text
            ┌───────────────┐
            │  input bytes  │
            └───────┬───────┘
                    │ sniff
       ┌────────────┼────────────┐
       ▼            ▼            ▼
   image path    audio path    text path
       │            │            │
   face det.    diarise      recogniser
   plate det.   STT          catalogue
   OCR → text  → text        analyzer
       └──────┬─────┘            │
              ▼                  │
          token map              │
              │                  │
              └──────┬───────────┘
                     ▼
            anonymisation operator
                     │
                     ▼
             egress / log / index
```

Each detector has its own labelled eval, own CI gate, own residual-risk entry in the DPIA. The composition is the pipeline.

## Summary — what you can do after this chapter

- Build a face-detection + blur / box pipeline with a measured per-demographic recall, and pick an operator (blur vs. box vs. replace) by threat model.
- Build an audio-PII pipeline that does transcription + text-pipeline + waveform-span-redaction; understand the voiceprint vs. content distinction.
- Build an OCR + text-pipeline pipeline for scanned PDFs / documents; understand the cascading-FN failure mode.
- Compose the three for UI screen captures.
- Map the detectors to the relevant special-category obligations (GDPR Art. 9; HIPAA; BIPA; EU AI Act).
- Budget multimodal detectors at ingest / retrieval, not at prompt-side.

The next chapter switches modalities again — from unstructured to tabular. Where this chapter and chapters 02-04 remove PII at the span level, chapter 07 asks the harder question: when a release is *already* de-identified at the field level, what statistical risk remains, and how do you measure it with k-anonymity, l-diversity, and t-closeness?
