# NATS Architecture and Pipeline Documentation

## 1. System Architecture Overview

This project utilizes a resilient, event-driven microservices architecture built on NATS and JetStream. Services are decoupled and communicate asynchronously by passing messages, which contain references to data rather than the data itself. This allows for a scalable and robust document processing pipeline.

The high-level data flow is as follows:

```
[PDF Upload] -> (pdf.created) -> [pdf-to-png-service] -> (png.created) -> [png-to-text-service] -> (text.processed) -> [tts-service] -> (audio.chunk.created) -> [pcm-to-wav-service] -> (wav.created) -> [wav-aggregator-service] -> (final.audio.created) -> [Final Audio Files]
```

## 2. Core Concepts

### NATS & JetStream

NATS provides the core messaging backbone. We specifically use **JetStream**, the persistence layer of NATS, for all inter-service communication. This guarantees that even if a consuming service is offline, messages will be retained and processed once the service comes back online (at-least-once delivery).

### JetStream Object Store

To handle large binary files (PDFs, PNGs, audio), we use the **JetStream Object Store**. Instead of publishing large payloads directly into a NATS message, which is inefficient, we follow this pattern:

1.  A service uploads a large file (e.g., a PDF) to a designated Object Store bucket.
2.  The service then publishes a small, lightweight event message to a NATS subject.
3.  This event message contains a **key** (e.g., `pdf_key`) that uniquely identifies the file in the Object Store.
4.  The consuming service receives this event, extracts the key, and uses it to retrieve the file directly from the Object Store.

This pattern keeps the messaging layer fast and efficient while allowing for the transfer of arbitrarily large files.

## 3. Service Interaction Pipeline (End-to-End Flow)

### 3.1. `pdf-to-png-service`

-   **Input:** Consumes a `PDFCreatedEvent` from the `pdf.created` subject on the `PDF_JOBS` stream.
-   **Action:**
    1.  Extracts the `PDFKey` from the event.
    2.  Downloads the source PDF from the `PDF_FILES` object store bucket.
    3.  Converts each page of the PDF into a separate PNG image.
    4.  Uploads each generated PNG to the `PNG_FILES` object store bucket, receiving a unique key for each one.
-   **Output:** For each successfully created PNG, it publishes a `PNGCreatedEvent` (containing the new `PNGKey`) to the `png.created` subject.

### 3.2. `png-to-text-service`

-   **Input:** Consumes `PNGCreatedEvent` messages from the `png.created` subject on the `PNG_PROCESSING` stream.
-   **Action:**
    1.  Extracts the `PNGKey` from the event.
    2.  Downloads the corresponding PNG image from the `PNG_FILES` object store.
    3.  Performs Optical Character Recognition (OCR) on the image to extract text.
    4.  Optionally augments the text with AI-generated commentary.
    5.  Uploads the resulting text to the `TEXT_FILES` object store, receiving a unique `TextKey`.
-   **Output:** Publishes a `TextProcessedEvent` (containing the `TextKey` and original page metadata) to the `text.processed` subject.

### 3.3. `tts-service`

-   **Input:** Consumes `TextProcessedEvent` messages from the `text.processed` subject on the `TTS_JOBS` stream.
-   **Action:**
    1.  Extracts the `TextKey` from the event.
    2.  Downloads the text file from the `TEXT_FILES` object store.
    3.  Generates a raw PCM audio representation of the text using the `chatllm` binary.
    4.  Uploads the final audio segment to the `AUDIO_FILES` object store, receiving a unique `AudioKey`.
-   **Output:** Publishes an `AudioChunkCreatedEvent` (containing the `AudioKey`) to the `audio.chunk.created` subject.

### 3.4. `pcm-to-wav-service`

-   **Input:** Consumes `AudioChunkCreatedEvent` messages from the `audio.chunk.created` subject on the `AUDIO_PROCESSING` stream.
-   **Action:**
    1.  Extracts the `AudioKey` from the event.
    2.  Downloads the PCM audio file from the `AUDIO_FILES` object store.
    3.  Converts the PCM file to a WAV file using `sox`.
    4.  Uploads the final WAV file to the `WAV_FILES` object store, receiving a unique `WavKey`.
    -   **Output:** Publishes a `WavFileCreatedEvent` (containing the `WavKey`) to the `wav.created` subject.

### 3.5. `wav-aggregator-service`

-   **Input:** Consumes `WavFileCreatedEvent` messages from the `wav.created` subject.
-   **Action:**
    1.  Persists workflow state in a NATS Key-Value store.
    2.  When all WAV files for a workflow are received, it downloads them from the `WAV_FILES` object store.
    3.  Combines them into a single WAV file using `sox`.
    4.  Uploads the final WAV file to the `FINAL_AUDIO_FILES` object store.
-   **Output:** Publishes a `FinalAudioCreatedEvent` (containing the `FinalAudioKey`) to the `final.audio.created` subject.
## 4. Configuration & Event Reference

### 4.1. Event Payloads (`events` package)

All events share a common `EventHeader`. The key data fields for each event are:

-   `PDFCreatedEvent`: Contains `PDFKey` (string).
-   `PNGCreatedEvent`: Contains `PNGKey` (string), `PageNumber` (int), `TotalPages` (int).
-   `TextProcessedEvent`: Contains `TextKey` (string), `PNGKey` (string), `PageNumber` (int), `TotalPages` (int), and TTS parameters: `Voice` (string), `Seed` (int), `NGL` (int), `TopP` (float64), `RepetitionPenalty` (float64), `Temperature` (float64).
-   `AudioChunkCreatedEvent`: Contains `AudioKey` (string), `PageNumber` (int), `TotalPages` (int).
-   `WavFileCreatedEvent`: Contains `WavKey` (string), `PageNumber` (int), `TotalPages` (int).

### 4.2. Configuration (`project.toml`)

The central `project.toml` file defines all NATS-related configuration under the `[nats]` section. This is the single source of truth for all stream, subject, and bucket names.

```toml
[nats]
url = "nats://127.0.0.1:4222"

# PDF Processing
pdf_stream_name = "PDF_JOBS"
pdf_consumer_name = "pdf-workers"
pdf_created_subject = "pdf.created"
pdf_object_store_bucket = "PDF_FILES"

# PNG Processing
png_stream_name = "PNG_PROCESSING"
png_consumer_name = "png-text-workers"
png_created_subject = "png.created"
png_object_store_bucket = "PNG_FILES"

# Text Processing
text_stream_name = "TTS_JOBS"
text_object_store_bucket = "TEXT_FILES"
text_processed_subject = "text.processed"

# TTS Processing
tts_stream_name = "TTS_JOBS"
tts_consumer_name = "tts-workers"
audio_chunk_created_subject = "audio.chunk.created"
audio_object_store_bucket = "AUDIO_FILES"

# PCM to WAV Processing
audio_processing_stream_name = "AUDIO_PROCESSING"
audio_processing_consumer_name = "pcm-to-wav-workers"
wav_created_subject = "wav.created"

# WAV Aggregator Service
wav_consumer_name = "wav-aggregator-workers"
final_audio_created_subject = "final.audio.created"
final_audio_object_store_bucket = "FINAL_AUDIO_FILES"
wav_aggregator_kv_bucket = "WAV_AGGREGATOR_STATE"

```

```toml
[wav_aggregator_service]
worker_pool_size = 10
```

## 5. Identified Issues and Recommendations

This section details issues identified during the review of the `wav-aggregator-service` and the solutions implemented or recommended for future work.

### 5.1. Workflow State Management

*   **Issue:** The initial implementation stored workflow progress in memory (`map[string]*WorkflowState`), leading to data loss on service restarts and making the service non-resilient.
*   **Solution Implemented:** Refactored to use a **NATS Key-Value store** (`nats.KeyValue`) to persist `WorkflowState`. This ensures state is durable across restarts, making the service stateless and resilient.
*   **Recommendation:** All services managing multi-step workflows should leverage NATS Key-Value stores for persistent state management.

### 5.2. Concurrency Control

*   **Issue:** The service could spawn an unlimited number of concurrent `sox` processes, potentially leading to resource exhaustion and instability under high load.
*   **Solution Implemented:** Implemented a **channel-based worker pool (semaphore)** to limit the number of concurrent aggregations. The pool size is configurable via `worker_pool_size` in `project.toml`.
*   **Recommendation:** Services performing resource-intensive operations should implement configurable concurrency limits.

### 5.3. Message Acknowledgement and Error Handling

*   **Issue:** Messages were acknowledged prematurely (before aggregation completion), leading to lost messages if aggregation failed. There was no built-in retry mechanism for transient errors.
*   **Solution Implemented:** Message acknowledgement (`msg.Ack()`) is now performed **only after successful aggregation and workflow state deletion**. If any step fails, the message is not acknowledged, allowing NATS to redeliver it for retry.
*   **Recommendation:** Implement a more sophisticated retry mechanism (e.g., exponential backoff, dead-letter queues for persistent failures) for critical operations.

### 5.4. Testing and Code Quality

*   **Issue:** The `wav-aggregator-service` initially lacked comprehensive unit and integration tests, making refactoring and verification challenging.
*   **Solution Implemented:** Added **unit tests** for the `NatsWorker`'s state management, concurrency control, and message acknowledgement logic. Mocks were used to isolate dependencies.
*   **Recommendation:** Adhere strictly to the **Test-Driven Correctness (TDD)** principle. All new features and bug fixes must be accompanied by comprehensive unit and integration tests, aiming for high code coverage.

### 5.5. Context Propagation

*   **Issue:** The `context.Context` was not consistently propagated through the aggregation logic, limiting the ability to implement timeouts or cancellations for long-running operations.
*   **Solution Implemented:** Ensured `context.Context` is passed from the `Run` method down to the `aggregate` function and used in the `sox` command execution.
*   **Recommendation:** Always propagate `context.Context` for all operations that may involve I/O, long-running tasks, or external calls to enable proper cancellation and timeout.

### 5.6. Minor Logic Flaws

*   **Issue:** A minor bug was identified in the `aggregate` function's loop for iterating through `workflow.Events`.
*   **Solution Implemented:** The loop logic was reviewed and confirmed to be correct for 1-indexed page numbers.
*   **Recommendation:** Conduct thorough code reviews and unit testing to catch such subtle logic errors early in the development cycle.