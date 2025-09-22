# Overall Architecture: BookExpert Microservices

The BookExpert system is a microservices-based application designed to process documents (primarily PDFs) and convert them into narrated audio. It leverages NATS (NATS.io) extensively for inter-service communication, eventing, and object storage (JetStream). The system is composed of several Go-based microservices, a C++ LLM inference engine, and a Python prompt builder, all orchestrated to form a document processing pipeline.

## Core Components & Data Flow:

1.  **UI Service (`ui-service`)**:
    *   **Purpose**: Provides a web-based user interface for uploading documents and configuring processing options (e.g., commentary, summary).
    *   **Technology**: HTML, Tailwind CSS, JavaScript.
    *   **Interaction**: Likely interacts with an API Gateway (not explicitly defined as a separate service, but implied by `PDFCreatedEvent` publisher).

2.  **API Gateway (Implied)**:
    *   **Purpose**: Acts as the entry point for external requests, receiving document uploads from the UI.
    *   **Interaction**: Publishes `PDFCreatedEvent` to NATS, initiating the document processing workflow.

3.  **PDF-to-PNG Service (`pdf-to-png-service`)**:
    *   **Purpose**: Converts PDF documents into a series of PNG images, one for each page. It can also detect and skip blank pages.
    *   **Technology**: Go, NATS, Ghostscript (`gs`).
    *   **Input**: Consumes `PDFCreatedEvent` from NATS. Downloads PDF from NATS object store (`pdf_files`).
    *   **Output**: Uploads generated PNGs to NATS object store (`png_images`). Publishes `PNGCreatedEvent` for each PNG to NATS.

4.  **PNG-to-Text Service (`png-to-text-service`)**:
    *   **Purpose**: Performs Optical Character Recognition (OCR) on PNG images to extract text. Optionally, it can augment the extracted text using an AI model (Google Gemini).
    *   **Technology**: Go, NATS, Tesseract OCR (`tesseract`), Google Gemini API, `prompt-builder` library.
    *   **Input**: Consumes `PNGCreatedEvent` from NATS. Downloads PNGs from NATS object store (`png_images`).
    *   **Output**: Uploads extracted text (and augmented text) to NATS object store (`text_files`). Publishes `TextProcessedEvent` to NATS.

5.  **TTS Service (`tts-service`)**:
    *   **Purpose**: Converts processed text into audio chunks (PCM format).
    *   **Technology**: Go, NATS, `chatllm` binary (LLM inference engine).
    *   **Input**: Consumes `TextProcessedEvent` from NATS. Downloads text from NATS object store (`text_files`).
    *   **Output**: Uploads generated PCM audio chunks to NATS object store (`audio_files`). Publishes `AudioChunkCreatedEvent` to NATS.

6.  **PCM-to-WAV Service (`pcm-to-wav-service`)**:
    *   **Purpose**: Converts raw PCM audio chunks into WAV format.
    *   **Technology**: Go, NATS, `sox` command-line tool.
    *   **Input**: Consumes `AudioChunkCreatedEvent` from NATS. Downloads PCM audio from NATS object store (`audio_files`).
    *   **Output**: Uploads WAV files to NATS object store (`wav_files`). Publishes `WavFileCreatedEvent` to NATS.

7.  **WAV Aggregator Service (`wav-aggregator-service`)**:
    *   **Purpose**: Collects all WAV files associated with a specific workflow and combines them into a single, final audio file.
    *   **Technology**: Go, NATS, `sox` command-line tool.
    *   **Input**: Consumes `WavFileCreatedEvent` from NATS. Downloads WAV files from NATS object store (`wav_files`). Uses NATS Key-Value store (`WavAggregatorKVBucket`) to track workflow progress.
    *   **Output**: Uploads the final aggregated WAV file to NATS object store (`final_audio_files`). Publishes `FinalAudioCreatedEvent` to NATS.

## Shared Libraries/Utilities:

*   **`configurator`**: A Go library responsible for loading application configurations from a TOML file specified by the `PROJECT_TOML` environment variable (which points to a URL). This enables centralized configuration management.
*   **`events`**: A Go package that defines the standardized NATS message payloads (`EventHeader`, `PDFCreatedEvent`, `PNGCreatedEvent`, `TextProcessedEvent`, `AudioChunkCreatedEvent`, `WavFileCreatedEvent`, `FinalAudioCreatedEvent`). This ensures consistent communication contracts between microservices.
*   **`logger`**: A Go library providing thread-safe, leveled logging to both stdout and files. Used across all Go services for consistent logging.
*   **`prompt-builder`**: A Go application/library for constructing structured prompts for AI models. Used by `png-to-text-service` for AI augmentation.

## External Dependencies/Tools:

*   **NATS Server with JetStream**: The central messaging and object storage backbone for the entire system.
*   **`chatllm.cpp`**: A C++ LLM inference engine (likely compiled into a binary named `chatllm`) used by the `tts-service` for text-to-speech synthesis. It supports various models, quantization, and GPU acceleration.
*   **Ghostscript (`gs`)**: Command-line tool used by `pdf-to-png-service` for PDF rendering.
*   **Tesseract OCR (`tesseract`)**: Command-line tool used by `png-to-text-service` for text extraction.
*   **`sox`**: Command-line tool used by `pcm-to-wav-service` and `wav-aggregator-service` for audio format conversion and aggregation.
*   **Google Gemini API**: Used by `png-to-text-service` for AI augmentation.

## Configuration Management:

All Go services use the `configurator` library to load their settings from a TOML file. The path to this TOML file is provided via the `PROJECT_TOML` environment variable, which is expected to be a URL. This allows for dynamic and centralized configuration.

## Error Handling:

Services in the BookExpert ecosystem use different NATS patterns, which dictates their error handling strategy:
-   **Durable Consumers**: Services that perform long-running or critical tasks, like the `wav-aggregator-service`, use durable consumers. They explicitly acknowledge messages on success (`Ack`) or failure (`Nak`/`Term`). This ensures that messages are not lost and can be retried or sent to a dead-letter queue.
-   **Request-Reply**: Simpler services like `pcm-to-wav-service` and `tts-service` operate on a request-reply basis. They listen for a message and send their result back as a direct reply. In this model, error handling is implicit: if a service fails to process a message, it simply doesn't send a reply, and the original requester is responsible for handling the timeout.

---

## Detailed Architecture: PDF-to-PNG Service

**1. Project Summary**

The `pdf-to-png-service` is a NATS-based microservice within the BookExpert ecosystem responsible for converting PDF documents into a series of PNG images, one for each page. It integrates with NATS for messaging and object storage, utilizes Ghostscript for rendering, and can optionally detect and remove blank pages.

**2. Core Capabilities**

*   **NATS Integration**: Consumes `PDFCreatedEvent` messages and publishes `PNGCreatedEvent` messages. Interacts with NATS object stores for PDF input and PNG output.
*   **Concurrent Processing**: Employs a worker pool to process PDF pages concurrently.
*   **High-Quality Rendering**: Renders PDF pages to PNG using Ghostscript, with configurable DPI.
*   **Intelligent Blank Page Detection**: Can detect and remove blank pages using a helper binary (`detect-blank`).
*   **Robust Error Handling**: Implements NATS `Ack`, `Nak`, and `Term` for reliable message processing.

**3. Technology Stack**

*   **Programming Language**: Go 1.25
*   **Messaging**: NATS (with JetStream)
*   **External Tools**:
    *   **Ghostscript (`gs`)**: Command-line tool for PDF rendering.
    *   **`detect-blank`**: A custom Go helper binary for blank page detection.
*   **Go Libraries**:
    *   `github.com/nats-io/nats.go`: NATS client library.
    *   `github.com/book-expert/configurator`: For loading service configuration.
    *   `github.com/book-expert/events`: For standardized NATS event structures.
    *   `github.com/book-expert/logger`: For structured logging.
    *   `github.com/cheggaaa/pb/v3`: For progress bars (likely for CLI/debugging, not core service logic).
    *   `github.com/google/uuid`: For generating unique identifiers.
    *   `github.com/stretchr/testify`: For testing utilities.

**4. Internal Components and Data Flow**

The service's main execution flow is managed by `cmd/main.go`, which orchestrates the interaction between NATS and the core PDF rendering logic located in `internal/pdfrender`.

*   **`cmd/main.go` (Service Orchestration)**:
    *   **Initialization**:
        *   Sets up a signal handler for graceful shutdown (`SIGINT`, `SIGTERM`).
        *   Initializes a bootstrap logger (temporary, to log configuration loading).
        *   Loads service configuration using the `configurator` library, fetching a TOML file from a URL specified by `PROJECT_TOML` environment variable.
        *   Initializes the main application logger based on the loaded configuration (`cfg.Paths.BaseLogsDir`).
        *   Connects to the NATS server and obtains a JetStream context.
        *   Ensures the necessary NATS streams (`pdf_stream_name`, `png_stream_name`) and object stores (`pdf_object_store_bucket`, `png_object_store_bucket`) are created or exist.
        *   Creates a NATS consumer (`pdf_consumer_name`) to listen for `pdf.created` events.
    *   **Message Processing Loop (`processMessages`)**:
        *   Continuously fetches messages from the NATS consumer in batches.
        *   For each message (`jetstream.Msg`), it unmarshals the `PDFCreatedEvent`.
        *   Creates a `job` struct to encapsulate the context for processing a single PDF.
        *   Calls `job.run()` to execute the processing logic for the PDF.
    *   **`job` struct and methods**: Represents a single PDF conversion task.
        *   `run(ctx context.Context)`: The main execution method for a job.
            *   Logs receipt of the job.
            *   Sends an `InProgress` status to NATS.
            *   Calls `setupWorkDir()` to create a temporary directory for the PDF and generated PNGs. `cleanupWorkDir()` is deferred to ensure cleanup.
            *   Calls `executeProcessingSteps()` to perform the core conversion.
            *   Handles errors from `executeProcessingSteps()` by calling `nak()` (negative acknowledgment) or `term()` (terminate message processing) on the NATS message, depending on the error type.
            *   On success, calls `ack()` (acknowledgment) on the NATS message.
        *   `downloadPDF(ctx context.Context)`: Downloads the PDF file from the `pdf_object_store_bucket` to the local temporary directory.
        *   `processPDF(ctx context.Context)`:
            *   Determines the executable directory to locate the `detect-blank` binary.
            *   Initializes a `pdfrender.Processor` with configured options (DPI, worker count, blank detection thresholds).
            *   Calls `processor.Process()` to start the PDF-to-PNG conversion.
            *   Returns the path to the directory containing the final PNGs.
        *   `publishPNGs(ctx context.Context, pngsFinalDir string)`:
            *   Reads the generated PNG files from the temporary output directory.
            *   Filters for `.png` files.
            *   For each PNG file:
                *   Constructs a unique object name (e.g., `tenantID/workflowID/page_0001.png`).
                *   Uploads the PNG to the `png_object_store_bucket`.
                *   Publishes a `PNGCreatedEvent` to the `png.created` subject, including `PNGKey`, `PageNumber`, and `TotalPages`.

*   **`internal/pdfrender` package**: Contains the core logic for PDF processing.
    *   **`Processor` struct**: Encapsulates configuration (`Options`), logger, and a `CommandExecutor`.
    *   **`NewProcessor(opts *Options, log *logger.Logger)`**: Constructor that applies default options.
    *   **`Process(ctx context.Context)`**: Main method of the `Processor`.
        *   Validates configuration.
        *   Calls `prepareTools()` to ensure `detect-blank` binary is available (compiles it if missing).
        *   Discovers PDF files in the input path (though in this service, the PDF path comes from NATS).
        *   Calls `processOnePDF()` for each PDF (in the context of the NATS worker, this is called once per received PDF).
    *   **`processOnePDF(ctx context.Context, pdfPath string)`**:
        *   Gets the total page count of the PDF using `pdfinfo` (via `processor.executor.Run`).
        *   Creates a structured output directory for the PNGs using `setupOutputDirectory`.
        *   Initializes a `pageProcessor` to manage concurrent page rendering.
        *   Calls `pageProcessor.processPages()` to render all pages.
    *   **`pageProcessor` struct and methods**: Manages concurrent rendering of pages for a single PDF.
        *   `processPages(ctx context.Context, pdfPath string, pageCount int)`:
            *   Creates a channel (`jobs`) for `pageJob` structs.
            *   Starts a pool of worker goroutines (`pageWorker`).
            *   Sends a `pageJob` for each page to the `jobs` channel.
            *   Uses `pb.New` for a progress bar (visible in console output).
            *   Waits for all workers to complete.
        *   `pageWorker(ctx context.Context, waitGroup *sync.WaitGroup, jobs <-chan pageJob)`:
            *   Worker goroutine that pulls `pageJob`s from the channel.
            *   Calls `processSinglePage()` for each job.
        *   `processSinglePage(ctx context.Context, job pageJob)`:
            *   Calls `renderPage()` to convert a single PDF page to PNG using Ghostscript.
            *   Calls `handleBlankDetection()` to check if the generated PNG is blank and removes it if so.
    *   **`commands.go`**: Handles execution of external commands (`pdfinfo`, `ghostscript`, `detect-blank`).
        *   `CommandExecutor` interface and `defaultExecutor` implementation for mocking.
        *   `getPDFPages`: Uses `pdfinfo` to get page count.
        *   `renderPage`: Uses `ghostscript` to convert a PDF page to PNG.
        *   `handleBlankDetection`: Uses `isPngBlank` to check for blank pages.
        *   `isPngBlank`: Executes the `detect-blank` helper binary.
        *   `ensureDetectBlankBinary`: Checks if `detect-blank` exists and builds it from source if not.
    *   **`files.go`**: Utility functions for file system operations.
        *   `DiscoverPDFs`: Finds PDF files (not directly used in the NATS worker flow, but useful for local testing/CLI).
        *   `setupOutputDirectory`: Creates output directories for PNGs.

*   **`cmd/detect-blank/main.go` (Helper Binary)**:
    *   **Purpose**: A standalone command-line tool compiled from Go source that determines if a given PNG image is "blank" (mostly white) based on configurable fuzziness and non-white pixel thresholds.
    *   **Usage**: Takes `filepath`, `fuzz_percent`, and `non_white_threshold` as arguments.
    *   **Exit Codes**: `0` for blank, `1` for not blank, `2` for error.
    *   **Mechanism**: Decodes the PNG, iterates through pixels, and counts non-white pixels based on the provided thresholds.

**5. Configuration**

The `pdf-to-png-service` is configured via a TOML file loaded by the `configurator` library. Key configuration parameters include:

```toml
[nats]
url = "nats://localhost:4222"
pdf_stream_name = "pdfs"
pdf_consumer_name = "pdf_processor"
pdf_created_subject = "pdf.created"
pdf_object_store_bucket = "pdf_files"
png_stream_name = "pngs"
png_created_subject = "png.created"
png_object_store_bucket = "png_images"

[paths]
base_logs_dir = "/var/log/pdf-to-png-service"
```

Additionally, the `pdfrender.Options` struct allows configuration of:
*   `DPI`: Resolution for PNG rendering (default 300).
*   `Workers`: Number of concurrent goroutines for page processing (default 4).
*   `BlankFuzzPercent`: Tolerance for white pixel deviation (default 5).
*   `BlankNonWhiteThreshold`: Minimum ratio of non-white pixels to consider content (default 0.005).

**6. Error Handling**

The service employs robust error handling, particularly around NATS message processing:
*   **`jobError`**: Custom error type to associate an error with a specific NATS message handling action (`Nak` or `Term`).
*   **`ack()`, `nak(reason error)`, `term(reason error)`**: Methods on the `job` struct to acknowledge, negatively acknowledge, or terminate a NATS message, ensuring proper message lifecycle management and retry mechanisms (if configured in NATS JetStream).
*   Detailed logging is used to capture errors and warnings.