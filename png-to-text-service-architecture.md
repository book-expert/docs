# PNG to Text Service Architecture

## Project Summary

A service that processes PNG images from a NATS JetStream, extracts text using OCR, optionally augments the text using the Gemini API, and publishes the results back to NATS.

## Detailed Description

The `png-to-text-service` is an event-driven microservice designed to automate the extraction and augmentation of text from PNG images. It addresses the problem of converting image-based text into a machine-readable format, with an optional step to enhance the extracted text using advanced AI capabilities.

The service operates by:
1.  **Consuming PNG events:** It listens for new PNG image events on a NATS JetStream subject.
2.  **Retrieving images:** Upon receiving an event, it fetches the corresponding PNG image from a NATS Object Store.
3.  **Text Extraction (OCR):** It utilizes Tesseract OCR to extract raw text from the image.
4.  **Text Cleaning:** The raw OCR output is cleaned to remove artifacts and normalize formatting.
5.  **Text Augmentation (Optional):** If configured, the cleaned text is sent to the Google Gemini API for further processing, such as summarization, rephrasing, or keyword extraction, based on predefined prompts.
6.  **Publishing Results:** The final processed text is then stored in a NATS Object Store and a `TextProcessedEvent` is published to another NATS JetStream subject.

### High-Level Architecture

```mermaid
graph TD
    A[External System] -->|Publishes PNG Created Event| B(NATS JetStream - PNG Created Subject)
    B --> C{NATS Worker}
    C --> D[NATS Object Store - PNG Bucket]
    D --> C
    C --> E[Processing Pipeline]
    E --> F[OCR Processor (Tesseract)]
    E --> G[OCR Text Cleaner]
    F --> G
    G --> E
    E --> H[Gemini Processor (Google Gemini API)]
    H --> E
    E --> I[NATS Object Store - Text Bucket]
    I --> J(NATS JetStream - Text Processed Subject)
    J --> K[Downstream Consumers]
    C --x L(NATS JetStream - Dead Letter Subject)
```

## Technology Stack

-   **Programming Language:** Go (1.23+)
-   **Messaging & Object Storage:** NATS JetStream
-   **OCR Engine:** Tesseract (external CLI tool)
-   **AI Augmentation:** Google Gemini API
-   **Logging:** `github.com/book-expert/logger` (custom logger)
-   **Configuration:** Custom `config` package (`github.com/book-expert/configurator` for loading)
-   **Prompt Building:** `github.com/book-expert/prompt-builder`

## Getting Started

For detailed instructions on setting up the development environment and running the service, please refer to the main `README.md` file in the project root directory.

## Core Components

The service is structured into several internal packages, each with a distinct responsibility:

### `cmd/png-to-text-service` (Main Entry Point)

-   **Responsibility:** Application bootstrapping, configuration loading, component initialization (logger, NATS, OCR, Gemini, Pipeline, Worker), and graceful shutdown management.
-   **Key Functions:** `main()`, `run()`, `setupLogger()`, `loadConfig()`, `initOCRProcessor()`, `initGeminiProcessor()`, `initPipeline()`, `initNATSConnectionAndJetStream()`, `setupJetStream()`, `getPNGObjectStore()`, `getTextObjectStore()`, `initNATSWorker()`, `runWorker()`.

### `internal/config`

-   **Responsibility:** Loading, parsing, and providing access to application configuration from `project.toml` files and environment variables. It also applies default values and validates critical settings.
-   **Key Structures:** `Config`, `NATSConfig`, `Tesseract`, `Gemini`, `Augmentation`, `Prompts`, `Logging`, `PathsConfig`, `Project`.
-   **Key Functions:** `Load()`, `GetAPIKey()`, `EnsureDirectories()`, `GetLogFilePath()`, `validate()`, `applyDefaults()`.

### `internal/ocr`

-   **Responsibility:** Performing Optical Character Recognition (OCR) on image data using the external Tesseract CLI tool and cleaning the raw OCR output.
-   **Key Structures:** `Processor`, `TesseractConfig`, `Cleaner`.
-   **Key Functions:** `NewProcessor()`, `ProcessPNG()` (in `Processor`), `NewCleaner()`, `Clean()` (in `Cleaner`).

### `internal/augment`

-   **Responsibility:** Augmenting or enhancing extracted text using the Google Gemini API. This involves preparing image data, constructing prompts, making API calls, and handling retries.
-   **Key Structures:** `GeminiProcessor`, `GeminiConfig`, `AugmentationOptions`.
-   **Key Functions:** `NewGeminiProcessor()`, `AugmentTextWithOptions()`.

### `internal/pipeline`

-   **Responsibility:** Orchestrating the end-to-end process of converting a PNG image to augmented text. It integrates the `ocr` and `augment` components, managing temporary files and the flow of data between steps.
-   **Key Interfaces:** `OCRProcessor`, `Augmenter`.
-   **Key Structures:** `Pipeline`.
-   **Key Functions:** `New()`, `Process()`.

### `internal/worker`

-   **Responsibility:** The NATS consumer. It connects to NATS JetStream, subscribes to PNG creation events, retrieves images from the object store, invokes the processing pipeline, stores the processed text in the text object store, and publishes `TextProcessedEvent` messages. It also handles message acknowledgment, error logging, and dead-lettering.
-   **Key Interface:** `Pipeline`.
-   **Key Structures:** `NatsWorker`.
-   **Key Functions:** `New()`, `Run()`, `handleMsg()`, `processAndPublishText()`.

## Data Flow

1.  An external system publishes a `PNGCreatedEvent` message to the `PNG_CREATED_SUBJECT` on NATS JetStream, indicating a new PNG image is available for processing. The image itself is stored in the `PNG_OBJECT_STORE_BUCKET`.
2.  The `NatsWorker` subscribes to the `PNG_CREATED_SUBJECT`. Upon receiving a message, it retrieves the PNG image data from the `PNG_OBJECT_STORE_BUCKET`.
3.  The `NatsWorker` passes the raw PNG image data and an `objectID` to the `pipeline.Pipeline`'s `Process` method.
4.  The `pipeline.Pipeline` first creates a temporary PNG file from the raw data.
5.  The `pipeline.Pipeline` then uses the `ocr.Processor` to extract raw text from the temporary PNG file.
6.  The raw OCR text is passed to the `ocr.Cleaner` to remove artifacts and normalize the text.
7.  If augmentation is enabled and the cleaned text meets the minimum length requirement, the `pipeline.Pipeline` sends the cleaned text, the temporary image path, and configured augmentation options to the `augment.GeminiProcessor`. The `GeminiProcessor` interacts with the Google Gemini API to augment the text.
8.  The `pipeline.Pipeline` returns the final (cleaned or augmented) text.
9.  The `NatsWorker` generates a unique `textKey` for the processed text.
10. The `NatsWorker` stores this processed text in the `TEXT_OBJECT_STORE_BUCKET` using the generated `textKey`.
11. Finally, the `NatsWorker` constructs and publishes a `TextProcessedEvent` (containing `PNGKey`, `TextKey`, and other metadata) to the `TEXT_PROCESSED_SUBJECT` on NATS JetStream, signaling that the text is ready for downstream consumers.
12. In case of processing failures, the `NatsWorker` publishes the original message to a `DEAD_LETTER_SUBJECT` for further investigation and acknowledges the original message.

## Error Handling

The service employs explicit error handling throughout, wrapping errors with context using `fmt.Errorf("...: %w", err)` to provide clear traceability. Critical errors lead to service termination, while transient errors in message processing are handled by publishing to a dead-letter queue and acknowledging the original message to prevent reprocessing loops.

## Concurrency

The `NatsWorker` runs its message processing loop in a goroutine, allowing it to fetch and process messages concurrently. The service is designed for graceful shutdown, ensuring that ongoing processing tasks are completed before the application exits, by listening for OS signals and canceling the processing context.

## License

Distributed under the MIT License. See the LICENSE file for more information.
