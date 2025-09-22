# Configurator Service Architecture

## 1. Project Summary

The `configurator` service is a Go library responsible for fetching application configuration from a remote URL, specified by the `PROJECT_TOML` environment variable, and unmarshaling it into a type-safe Go struct. It acts as a centralized configuration client for other services within the Book Expert project.

## 2. Detailed Description

This library provides a robust and simple mechanism for services to retrieve their operational parameters. By relying on a URL-based configuration source, it enables dynamic configuration updates without requiring service redeployments. The library handles the HTTP request, including timeouts, and parses the TOML content into a provided Go struct, ensuring type safety and ease of use.

### Key Features:
-   **URL-based Configuration**: Fetches TOML configuration from any accessible HTTP endpoint.
-   **Environment Variable Driven**: The configuration source URL is determined by the `PROJECT_TOML` environment variable, promoting flexible deployment.
-   **Type-Safe Unmarshaling**: Utilizes `github.com/pelletier/go-toml/v2` for reliable TOML parsing into Go structs.
-   **Timeout Mechanism**: Includes a configurable timeout for HTTP requests to prevent indefinite blocking.
-   **Comprehensive Error Handling**: Provides explicit error types for common failure scenarios (e.g., missing environment variable, unexpected HTTP status).

## 3. Technology Stack

-   **Programming Language**: Go (version 1.25.1 as per `go.mod`)
-   **Core Libraries**:
    -   `github.com/pelletier/go-toml/v2`: For TOML parsing and unmarshaling.
    -   `github.com/book-expert/logger`: For structured logging within the library.
-   **Testing Framework**:
    -   `github.com/stretchr/testify`: For assertion and testing utilities.

## 4. Architectural Components

The `configurator` package consists of the following main components:

### `Load` Function
-   **Purpose**: The primary entry point for consumers of the library. It orchestrates the fetching and unmarshaling process.
-   **Inputs**: Takes an empty interface `target` (to unmarshal the TOML into) and a `*logger.Logger` instance for logging.
-   **Process**:
    1.  Reads the `PROJECT_TOML` environment variable to get the configuration URL.
    2.  Calls `fetchURL` to retrieve the TOML content.
    3.  Calls `unmarshalTOML` to parse the content into the `target` struct.
-   **Outputs**: Returns an `error` if any step fails.

### `fetchURL` Function
-   **Purpose**: Handles the HTTP request to fetch the TOML file from the specified URL.
-   **Inputs**: The configuration URL (string) and a `*logger.Logger`.
-   **Process**:
    1.  Creates an `http.Request` with a `context.WithTimeout` (defaulting to `DefaultURLTimeout` of 10 seconds).
    2.  Executes the HTTP GET request using `http.DefaultClient.Do`.
    3.  Defers closing the response body.
    4.  Calls `processResponse` to handle the HTTP response.
-   **Outputs**: Returns the raw TOML content as `[]byte` or an `error`.

### `processResponse` Function
-   **Purpose**: Validates the HTTP response status and reads the response body.
-   **Inputs**: An `*http.Response` object.
-   **Process**:
    1.  Checks if `resp.StatusCode` is `http.StatusOK`. If not, returns `ErrUnexpectedHTTPStatus`.
    2.  Reads the entire `resp.Body` using `io.ReadAll`.
-   **Outputs**: Returns the response body as `[]byte` or an `error`.

### `unmarshalTOML` Function
-   **Purpose**: Parses the raw TOML data into the provided Go struct.
-   **Inputs**: Raw TOML data as `[]byte` and the `target` interface.
-   **Process**: Uses `toml.Unmarshal` to perform the parsing.
-   **Outputs**: Returns an `error` if parsing fails.

## 5. NATS and JetStream Integration

The `configurator` package **does not directly integrate with NATS or JetStream**. Its responsibility is solely to provide a mechanism for fetching and parsing configuration. Any service consuming this configuration might then use the retrieved settings to establish connections to NATS, configure JetStream streams, or define other messaging patterns. The `configurator` acts as a foundational utility, independent of the messaging layer.

## 6. Error Handling

The package defines specific error variables for clarity:
-   `ErrUnexpectedHTTPStatus`: Returned when the HTTP request to fetch the TOML file does not return a `200 OK` status.
-   `ErrProjectTomlNotSet`: Returned when the `PROJECT_TOML` environment variable is not set.

All internal errors are wrapped using `fmt.Errorf` to provide a clear chain of causation and context, adhering to Go's idiomatic error handling practices.

## 7. Testing

The `configurator_test.go` file contains comprehensive unit tests that cover:
-   Successful configuration loading from a mock HTTP server.
-   Handling of the `PROJECT_TOML` environment variable not being set.
-   Behavior with an invalid URL.
-   Handling of non-200 HTTP responses.
-   Error handling for invalid TOML content.

The tests utilize `httptest` to create a local HTTP server for controlled testing of network interactions and `github.com/stretchr/testify` for robust assertions.
