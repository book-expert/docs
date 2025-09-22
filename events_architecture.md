# Events Package Architecture

## 1. Project Summary

The `events` package is a Go library that defines the standardized data structures (Go structs) used as message payloads for inter-service communication within the Book Expert microservices ecosystem. These structures are designed to be transmitted via NATS, ensuring a consistent and trackable data flow for the entire document processing pipeline.

## 2. Detailed Description

This package serves as the central contract for all event-driven communication between the various microservices. By defining explicit and versioned event structures, it ensures that producers and consumers of messages understand the expected data format, promoting system stability, maintainability, and ease of integration. Each event includes a mandatory `EventHeader` for consistent metadata, supporting multi-tenancy, distributed tracing, and workflow management.

The events defined here represent significant state changes or actions within the Book Expert workflow, from PDF upload to final audio generation.

### Key Features:
-   **Standardized Payloads**: Provides a consistent format for all messages exchanged between services.
-   **Multi-tenancy Support**: `EventHeader` includes `TenantID` and `UserID` for isolating data and operations per tenant/user.
-   **Workflow Tracking**: `WorkflowID` in the `EventHeader` allows for end-to-end tracing of a document's processing.
-   **Clear Communication Contract**: Acts as a single source of truth for event definitions, reducing integration errors.

## 3. Technology Stack

-   **Programming Language**: Go (version 1.25.1 as per `go.mod`)
-   **Core Libraries**: None beyond standard Go libraries, as it primarily defines data structures.

## 4. Architectural Components (Event Definitions)

The package primarily consists of Go struct definitions, each representing a specific event in the document processing workflow:

### `EventHeader`
-   **Purpose**: A mandatory metadata structure embedded in all NATS events. It provides essential context for multi-tenancy, security, and distributed tracing.
-   **Fields**:
    -   `Timestamp` (`time.Time`): The time the event was created.
    -   `WorkflowID` (`string`): A unique identifier for the entire document processing workflow.
    -   `UserID` (`string`): The ID of the user who initiated the workflow.
    -   `TenantID` (`string`): The ID of the tenant the workflow belongs to.
    -   `EventID` (`string`): A unique identifier for the specific event.

### `PDFCreatedEvent`
-   **Purpose**: Published by the API Gateway to initiate a new workflow, signaling that a new PDF has been uploaded.
-   **Fields**:
    -   `Header` (`EventHeader`)
    -   `PDFKey` (`string`): The unique identifier for the uploaded PDF in the object store.

### `PNGCreatedEvent`
-   **Purpose**: Published by the `pdf-to-png-service` for each successfully rendered and non-blank page of a PDF.
-   **Fields**:
    -   `Header` (`EventHeader`)
    -   `PNGKey` (`string`): The unique identifier for the generated PNG in the object store.
    -   `PageNumber` (`int`): The 1-based index of the page.
    -   `TotalPages` (`int`): The total number of pages in the original document.

### `TextProcessedEvent`
-   **Purpose**: Published by the `png-to-text-service` after performing OCR and optional augmentation on a single page.
-   **Fields**:
    -   `Header` (`EventHeader`)
    -   `PNGKey` (`string`): Identifier of the source PNG image.
    -   `TextKey` (`string`): Unique identifier for the processed text in the object store.
    -   `PageNumber` (`int`)
    -   `TotalPages` (`int`)
    -   `Voice` (`string`, `omitempty`): Voice to be used for TTS.
    -   `Seed` (`int`, `omitempty`): Seed for random number generator.
    -   `NGL` (`int`, `omitempty`): Number of layers to offload to GPU.
    -   `TopP` (`float64`, `omitempty`): Top-p sampling value.
    -   `RepetitionPenalty` (`float64`, `omitempty`): Repetition penalty.
    -   `Temperature` (`float64`, `omitempty`): Temperature for sampling.

### `AudioChunkCreatedEvent`
-   **Purpose**: Published by the `text-to-speech` service for each successfully generated audio segment.
-   **Fields**:
    -   `Header` (`EventHeader`)
    -   `AudioKey` (`string`): Unique identifier for the generated audio chunk in the object store.
    -   `PageNumber` (`int`)
    -   `TotalPages` (`int`)

### `WavFileCreatedEvent`
-   **Purpose**: Published by the `pcm-to-wav-service` for each successfully generated WAV file from PCM data.
-   **Fields**:
    -   `Header` (`EventHeader`)
    -   `WavKey` (`string`): Unique identifier for the generated WAV file in the object store.
    -   `PageNumber` (`int`)
    -   `TotalPages` (`int`)

### `FinalAudioCreatedEvent`
-   **Purpose**: Published by the `wav-aggregator-service` after successfully combining all WAV files for a workflow.
-   **Fields**:
    -   `Header` (`EventHeader`)
    -   `FinalAudioKey` (`string`): Unique identifier for the combined audio file in the object store.

## 5. NATS and JetStream Integration

This package is foundational for NATS and JetStream integration within the Book Expert project. While it does not contain any NATS client code itself (e.g., publishing or subscribing logic), it defines the *schema* for the messages that will be sent and received over NATS. Services will serialize instances of these structs (e.g., to JSON) before publishing them to NATS subjects and deserialize incoming NATS messages back into these structs.

This explicit definition ensures:
-   **Type Safety**: Go services can work with strongly typed event data.
-   **Consistency**: All services use the same event definitions.
-   **Evolvability**: Changes to event structures can be managed centrally.

## 6. Error Handling

As this package primarily defines data structures, it does not contain active error handling logic. Errors related to event processing (e.g., serialization/deserialization failures, invalid event data) would be handled by the individual microservices that produce or consume these events.

## 7. Testing

The `README.md` for this package explicitly states that it contains only data structures and therefore does not include specific tests. The correctness of these structures is implicitly verified by the compilation process and the unit/integration tests of the services that utilize them.