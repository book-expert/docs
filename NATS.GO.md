# NATS Architecture and Pipeline Documentation

## 1. System Architecture Overview

This project utilizes a resilient, event-driven microservices architecture built on NATS and JetStream. Services are decoupled and communicate asynchronously by passing messages, which contain references to data rather than the data itself. This allows for a scalable and robust document processing pipeline.

The high-level data flow is as follows:

```
[PDF Upload] -> (pdf.created) -> [pdf-to-png-service] -> (png.created) -> [png-to-text-service] -> (text.processed) -> [tts-service] -> (audio.chunk.created) -> [Audio Files]
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

## 4. Configuration & Event Reference

### 4.1. Event Payloads (`events` package)

All events share a common `EventHeader`. The key data fields for each event are:

-   `PDFCreatedEvent`: Contains `PDFKey` (string).
-   `PNGCreatedEvent`: Contains `PNGKey` (string), `PageNumber` (int), `TotalPages` (int).
-   `TextProcessedEvent`: Contains `TextKey` (string), `PNGKey` (string), `PageNumber` (int), `TotalPages` (int), and TTS parameters: `Voice` (string), `Seed` (int), `NGL` (int), `TopP` (float64), `RepetitionPenalty` (float64), `Temperature` (float64).
-   `AudioChunkCreatedEvent`: Contains `AudioKey` (string), `PageNumber` (int), `TotalPages` (int).

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
```

### 4.3. NATS Server (`nats-server.service`)

The NATS server is run as a `systemd` service. The unit file is configured to enable JetStream via the `-js` flag in its `ExecStart` command:
`ExecStart=/usr/sbin/nats-server -js`

## 5. NATS Go Client Best Practices

When developing a service, use the following patterns based on the latest `nats.go` library.

**1. Get JetStream Context:**
```go
// nc is your nats.Conn
js, err := jetstream.New(nc)
if err != nil {
    log.Fatal("Failed to get JetStream context: ", err)
}
```

**2. Get or Create an Object Store:**
```go
// The bucket name comes from your service's config, loaded from project.toml
bucketName := cfg.NATS.PDFObjectStoreBucket
ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
defer cancel()

store, err := js.ObjectStore(ctx, bucketName)
if err != nil {
    cfg := jetstream.ObjectStoreConfig{
        Bucket: bucketName,
        // ... other options
    }
    store, err = js.CreateObjectStore(ctx, cfg)
    if err != nil {
        return fmt.Errorf("failed to create object store '%s': %w", bucketName, err)
    }
}
```

**3. Download an Object:**
```go
// The key comes from the event message (e.g., event.PDFKey)
object, err := pdfStore.Get(ctx, key)
if err != nil {
    return fmt.Errorf("failed to get object '%s': %w", key, err)
}
defer object.Close()

// object implements io.Reader, so you can stream it to a file or memory.
bytes, err := io.ReadAll(object)
```

**4. Upload an Object:**
```go
// dataReader can be an os.File, bytes.Reader, etc.
info, err := pngStore.Put(ctx, &nats.ObjectMeta{
    Name: "unique-key-for-png.png",
    Description: "Page 1 of source file XYZ.pdf",
}, dataReader)
if err != nil {
    return fmt.Errorf("failed to put object: %w", err)
}

// info.Name contains the key to publish in the next event.
log.Printf("Successfully put object %s", info.Name)
```

## 6. Submitting Jobs with the `nats` CLI

The `nats` command-line interface (CLI) is a powerful tool for interacting with a NATS server. It can be used to submit jobs to the pipeline, inspect streams and object stores, and more.

### 6.1. Installation

The `nats` CLI can be installed using `go install`:

```bash
go install github.com/nats-io/natscli/nats@latest
```

### 6.2. Submitting a Job

To submit a job to the pipeline, you need to:

1.  **Upload the PDF to the `PDF_FILES` object store.**
2.  **Publish a `PDFCreatedEvent` message to the `pdf.created` subject.**

Here's how to do it:

**1. Create the Object Store (if it doesn't exist):**

```bash
nats --server nats://127.0.0.1:4222 object add PDF_FILES
```

**2. Upload the PDF:**

```bash
nats --server nats://127.0.0.1:4222 object put PDF_FILES --name <pdf_filename> <path_to_pdf_file>
```

**3. Publish the Event:**

```bash
nats --server nats://127.0.0.1:4222 pub pdf.created '{"header":{"timestamp":"<timestamp>","workflow_id":"<workflow_id>","user_id":"user-123","tenant_id":"tenant-456","event_id":"<event_id>"},"pdf_key":"<pdf_filename>"}'
```

Replace `<pdf_filename>`, `<path_to_pdf_file>`, `<timestamp>`, `<workflow_id>`, and `<event_id>` with the appropriate values.

## 7. Troubleshooting

### `pdf-to-png-service`: "not a directory" error

This error occurs when the `pdfrender` package, which is designed to work with a directory of PDF files, is given a path to a single file instead.

The `pdf-to-png-service` downloads the PDF from the object store to a temporary directory. The `InputPath` option for the `pdfrender` package must be set to the path of this temporary directory, not the path of the PDF file itself.

This can be fixed by changing the following line in `pdf-to-png-service/cmd/main.go`:

```go
// Incorrect:
opts := &pdfrender.Options{
    InputPath:              j.localPDFPath,
    // ...
}

// Correct:
opts := &pdfrender.Options{
    InputPath:              j.outputDir,
    // ...
}
```