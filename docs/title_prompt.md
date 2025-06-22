# title.go (in internal/llm/prompt)

## Overview

The `title.go` file, part of the `internal/llm/prompt` package, defines the system prompt for the "Title" agent. This agent's sole purpose is to generate a short, descriptive title for a new conversation based on the user's initial message. The prompt provides strict guidelines to the LLM to ensure the generated titles are suitable.

## Key Components

### Functions
- `TitlePrompt(_ models.ModelProvider) string`:
    - This function returns a static string that serves as the system prompt for the Title agent.
    - Like other specific agent prompt functions in this package (e.g., `SummarizerPrompt`, `TaskPrompt`), the `models.ModelProvider` argument is present in the signature but not used to customize the prompt. The title generation instructions are consistent across LLM providers.
    - The prompt gives the LLM clear instructions for title generation:
        - **Objective**: "generate a short title based on the first message a user begins a conversation with."
        - **Length Constraint**: "ensure it is not more than 50 characters long."
        - **Content**: "the title should be a summary of the user's message."
        - **Format**:
            - "it should be one line long."
            - "do not use quotes or colons."
            - "the entire text you return will be used as the title." (This implies no extra conversational text, just the title itself).
            - "never return anything that is more than one sentence (one line) long."

## Important Variables/Constants

This file does not define any exported package-level constants or variables beyond the `TitlePrompt` function. The prompt content is an unexported string literal.

## Usage Examples

The `TitlePrompt` function is invoked by `GetAgentPrompt` (in `prompt.go`) when the agent name is `config.AgentTitle`. This system prompt is then used to initialize the LLM provider (specifically, the `titleProvider` in `internal/llm/agent/agent.go`) responsible for generating titles.

```go
// Conceptual usage by GetAgentPrompt:
// import "github.com/opencode-ai/opencode/internal/llm/prompt"
// import "github.com/opencode-ai/opencode/internal/config"
// import "github.com/opencode-ai/opencode/internal/llm/models"

// agentType := config.AgentTitle
// providerType := models.ProviderOpenAI // or any other provider
//
// systemPromptForTitleAgent := prompt.GetAgentPrompt(agentType, providerType)
// // systemPromptForTitleAgent will contain the text defined in TitlePrompt()
```
The `internal/llm/agent/agent.go`'s `generateTitle` function uses an agent configured with this prompt to create a title from the user's first message in a new session.

## Dependencies and Interactions

- **Internal Dependencies:**
    - `github.com/opencode-ai/opencode/internal/llm/models`: For the `models.ModelProvider` type in the function signature, though not actively used for logic in this prompt.
- **External Libraries:** None.
- **Interactions:**
    - Provides a fixed, highly constrained system prompt for the Title agent.
    - This prompt is used by the `internal/llm/agent` package when an agent of type `config.AgentTitle` is invoked (specifically by the `titleProvider`).
    - The strict formatting and length constraints in the prompt are critical for ensuring that the generated titles are suitable for display in UI elements (like a session list) and for database storage.
