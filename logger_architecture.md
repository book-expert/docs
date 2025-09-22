# Logger Package Architecture

## 1. Project Summary

The `logger` package provides a simple, file-based logging utility for Go applications within the Book Expert ecosystem. It allows services to write structured log messages to a specified file with support for different log levels (Info, Warn, Error, Debug).

## 2. Detailed Description

This package offers a straightforward and efficient way to centralize application logs to a file. It is designed to be easy to integrate into any Go service, providing basic logging functionalities without external dependencies beyond standard Go libraries. The logger ensures thread-safe writes to the log file and includes standard log flags for timestamping and file/line information.

### Key Features:
-   **File-based Logging**: All log messages are written to a designated file.
-   **Multiple Log Levels**: Supports `Info`, `Warn`, `Error`, and `Debug` levels for granular control over log output.
-   **Thread-Safe**: Uses a mutex to protect concurrent writes to the log file.
-   **Standard Log Formatting**: Includes timestamps and source file information in log entries.
-   **Simple API**: Easy to initialize and use across different parts of an application.

## 3. Technology Stack

-   **Programming Language**: Go (version 1.25.1 as per `go.mod`)
-   **Core Libraries**: Standard Go `log` package, `os`, `sync`.

## 4. Architectural Components

The `logger` package consists of the following main components:

### `Logger` Struct
-   **Purpose**: Represents the logging instance, holding the underlying `log.Logger` and a mutex for synchronization.
-   **Fields**:
    -   `*log.Logger`: The standard Go logger instance.
    -   `*os.File`: The file to which logs are written.
    -   `sync.Mutex`: A mutex to ensure thread-safe writes.

### `New` Function
-   **Purpose**: Constructor for creating a new `Logger` instance.
-   **Inputs**: `logDir` (string) for the directory where the log file will be created, and `logFileName` (string) for the name of the log file.
-   **Process**:
    1.  Creates the log directory if it doesn't exist.
    2.  Opens/creates the log file in append mode.
    3.  Initializes a `log.Logger` with the file as output and `log.LstdFlags`.
-   **Outputs**: Returns a new `*Logger` instance or an `error`.

### Logging Methods (`Info`, `Warn`, `Error`, `Debug`)
-   **Purpose**: Provide an interface for writing log messages at different severity levels.
-   **Inputs**: Format string and variadic arguments.
-   **Process**:
    1.  Acquires a lock using the mutex.
    2.  Uses the underlying `log.Logger` to write the formatted message to the file.
    3.  Releases the lock.

### `Close` Method
-   **Purpose**: Closes the underlying log file, releasing system resources.
-   **Inputs**: None.
-   **Process**: Closes the `*os.File` associated with the logger.
-   **Outputs**: Returns an `error` if closing the file fails.

### `cmd/logger/main.go`
-   **Purpose**: A simple command-line application demonstrating the usage of the `logger` package. It initializes a logger and writes various log messages.

## 5. NATS and JetStream Integration

The `logger` package **does not directly integrate with NATS or JetStream**. It is a standalone utility designed to provide local file-based logging for individual services. Services that utilize this logger might themselves be integrated with NATS/JetStream, but the logging mechanism itself is decoupled from the messaging system.

## 6. Error Handling

The `New` function handles errors related to directory creation and file opening. The logging methods themselves do not return errors; instead, they rely on the underlying `log.Logger` to handle write failures (which typically print to `stderr` if the file is inaccessible). The `Close` method returns an error if the file cannot be closed properly.

## 7. Testing

The `logger_test.go` file contains unit tests that verify:
-   The successful creation of a new logger and log file.
-   The ability to write messages at different log levels to the file.
-   The content of the log file matches the expected output.
-   The `Close` method correctly closes the file.

The tests use `t.TempDir()` to create temporary directories for log files, ensuring test isolation and cleanup.