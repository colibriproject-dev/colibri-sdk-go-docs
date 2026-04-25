# Documentation Style Guide - Colibri Project Go

This guide defines the writing and formatting standards for the `colibri-sdk-go` documentation.

## 1. Tone and Voice

*   **Tone**: Helpful, technical, and accessible.
*   **Voice**: 
    *   Use the **first person plural** ("We", "We configure", "Let's see") for tutorials and step-by-step guides, creating a sense of guidance.
    *   Use the **third person** ("The package provides", "The function executes") for technical descriptions and component definitions.
*   **Language**: English (EN).

## 2. Standardized Terminology

To maintain consistency, always use the terms below:

*   **Microservice** (avoid: microserviço, micro-serviço).
*   **Framework**.
*   **Endpoint**.
*   **Middleware**.
*   **Query**.
*   **Statement**.
*   **Cache**.
*   **Observability**.
*   **Routing**.
*   **Payload**.
*   **Header**.
*   **SDK**.

## 3. Markdown Formatting

*   **Titles**:
    *   `# Title` (only via frontmatter `title`).
    *   `## Main Section` (H2).
    *   `### Subsection` (H3).
*   **Code Blocks**:
    *   Always specify the language (e.g., `go`, `shell`, `dotenv`, `sql`).
    *   Use `showLineNumbers` for blocks with more than 5 lines.
*   **Technical Terms**: Use backticks for file names, functions, variables, and packages (e.g., `main.go`, `InitializeApp()`, `colibri-sdk-go`).
*   **Highlight**: Use **bold** for important terms or moderate emphasis.
*   **Links**: Links to external libraries should be made on the first mention.
*   **Separator**: End all files with `___`.

## 4. Feature Page Structure

Follow this preferred order:

1.  **Summary/Introduction**: What the feature is and what it's for.
2.  **Configuration (Environment Variables)**: What configurations are required.
3.  **Initialization**: How to enable the feature in `main.go`.
4.  **Main Components**: Technical breakdown.
5.  **Usage Examples**: Practical code.
6.  **Best Practices** (Optional).

---
