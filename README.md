# Documentation

This directory contains the foundational design principles, coding standards, and architectural documentation for the book-expert project.

## Core Principles

-   [**DESIGN_PRINCIPLES_GUIDE.md**](./DESIGN_PRINCIPLES_GUIDE.md): The single source of truth for our development philosophy, including Test-Driven Correctness, Automated Quality Enforcement, and Verifiable Truth.

## Language-Specific Standards

Each of the following documents provides a detailed, practical implementation of the core principles for a specific language.

-   [**Go**](./GO_CODING_STANDARD.md): A strict guide to modern Go, mandating `golangci-lint`, Test-Driven Development, and self-documenting code.
-   [**Python**](./PYTHON_CODING_STANDARD.md): A strict guide to modern Python, mandating a `ruff`/`black`/`mypy` toolchain, Test-Driven Correctness, and clear, self-documenting patterns.
-   [**Bash**](./BASH_CODING_STANDARD.md): A strict guide for writing safe, simple, and reliable shell scripts, mandating `shellcheck` and robust error handling.

## Architecture and Guides

-   [**NATS Architecture**](./NATS.GO.md): An overview of the NATS-based event-driven architecture and data flow between the microservices.
-   [**Prompt Engineering Guide**](./PROMPT_ENGINEERING.md): Strategies and best practices for designing effective prompts for Large Language Models (LLMs).
-   [**README Writing Guide**](./README_WRITING_GUIDE.md): The standard for all `README.md` files in this project.