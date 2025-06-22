# message.go (in internal/tui/components/chat)

## Overview

The `message.go` file, part of the `internal/tui/components/chat` package, handles the rendering logic for individual messages within a chat conversation. It defines a `uiMessage` struct to hold rendered content and its properties. The core functionality revolves around `renderUserMessage` and `renderAssistantMessage` (which also handles nested tool call rendering via `renderToolMessage`). These functions take a `message.Message` and convert it into a styled string suitable for display in the TUI, applying themes, markdown formatting, and specific layouts for different message types and tool interactions.

## Key Components

- **`uiMessageType` enum**: Differentiates between `userMessageType`, `assistantMessageType`, and `toolMessageType` for styling or layout purposes.
- **`uiMessage` struct**:
    - `ID (string)`: Original message ID.
    - `messageType (uiMessageType)`
    - `position (int)`: Vertical position in the chat list (likely for scrolling).
    - `height (int)`: Calculated height of the rendered message.
    - `content (string)`: The final, styled string content of the message.
- **Rendering Functions**:
    - `renderUserMessage(...)`: Formats a user message, including any attachments.
    - `renderAssistantMessage(...)`: Formats an assistant message. This is complex as it handles:
        - Displaying main content or "thinking" content.
        - Displaying finish reason and model information.
        - Recursively rendering `ToolCall` parts using `renderToolMessage`.
        - Marking if the message is a session summary.
    - `renderToolMessage(...)`: Formats a `ToolCall` part, showing its name, parameters, and recursively rendering its `ToolResult` (if available) or a "Waiting for response..." status. It also handles nested tool calls from the "AgentTool".
    - `renderToolResponse(...)`: Formats the content of a `ToolResult` based on the tool that produced it (e.g., formatting diffs for "edit", code blocks for "view" or "write").
    - `renderToolParams(...)`: Formats the input parameters of a `ToolCall` for display.
    - `toMarkdown(...)`: Converts a string to Markdown and renders it.
    - `renderMessage(...)`: A general helper to wrap content with a border and apply basic styling.
- **Helper Functions**:
    - `findToolResponse(...)`: Locates a `ToolResult` corresponding to a `ToolCallID` within a list of messages.
    - `toolName(...)`, `getToolAction(...)`: Map internal tool names to display names and action descriptions.
    - `removeWorkingDirPrefix(...)`: Shortens file paths for display.
    - `truncateHeight(...)`: Limits the number of lines in a string.
    - `formatTimestampDiff(...)`: Formats time differences for display.

## Dependencies and Interactions

- Relies heavily on `message.Message` and its `ContentPart` types from `internal/message`.
- Uses `internal/config` for `WorkingDirectory()`.
- Uses `internal/diff` for formatting diffs from "edit" tool responses.
- Uses `internal/llm/agent` and `internal/llm/tools` for tool names.
- Uses `internal/llm/models` for model information.
- Uses `internal/tui/styles` and `internal/tui/theme` for all visual styling via `lipgloss`.
- `renderAssistantMessage` can call `messagesService.List` to fetch messages for nested task agent calls.

## Purpose

This file is crucial for translating the structured `message.Message` data into visually presentable, styled strings for the chat message list. It handles the complexities of displaying different content types, tool interactions, and assistant states (thinking, finished) in a user-friendly way within the TUI.
