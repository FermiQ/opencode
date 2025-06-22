# summarizer.go (in internal/llm/prompt)

## Overview

The `summarizer.go` file, part of the `internal/llm/prompt` package, defines the system prompt specifically for the "Summarizer" agent. This agent is tasked with creating summaries of conversations. The prompt guides the LLM on how to perform this summarization effectively.

## Key Components

### Functions
- `SummarizerPrompt(_ models.ModelProvider) string`:
    - This function returns a static string that serves as the system prompt for the Summarizer agent.
    - The `models.ModelProvider` argument is present in the function signature (likely to maintain consistency with other prompt-generating functions like `CoderPrompt`), but it is not used within the function body. This means the summarizer prompt is currently the same regardless of the underlying LLM provider.
    - The prompt instructs the AI:
        - Its role: "a helpful AI assistant tasked with summarizing conversations."
        - The desired output: "a detailed but concise summary."
        - Key areas to focus on in the summary:
            - What was done.
            - What is currently being worked on.
            - Which files are being modified.
            - What needs to be done next.
        - The overall goal of the summary: "comprehensive enough to provide context but concise enough to be quickly understood."

## Important Variables/Constants

This file does not define any exported package-level constants or variables beyond the `SummarizerPrompt` function itself. The prompt content is an unexported string literal within the function.

## Usage Examples

The `SummarizerPrompt` function is called by `GetAgentPrompt` (in `prompt.go`) when the agent name is `config.AgentSummarizer`. This system prompt is then used to initialize the LLM provider for the Summarizer agent.

```go
// Conceptual usage by GetAgentPrompt:
// import "github.com/opencode-ai/opencode/internal/llm/prompt"
// import "github.com/opencode-ai/opencode/internal/config"
// import "github.com/opencode-ai/opencode/internal/llm/models"

// agentType := config.AgentSummarizer
// providerType := models.ProviderOpenAI // or any other provider
//
// systemPromptForSummarizer := prompt.GetAgentPrompt(agentType, providerType)
// // systemPromptForSummarizer will contain the text defined in SummarizerPrompt()
```
This prompt then guides the `summarizeProvider` (configured in `internal/llm/agent/agent.go`) when it's asked to summarize a conversation.

## Dependencies and Interactions

- **Internal Dependencies:**
    - `github.com/opencode-ai/opencode/internal/llm/models`: For the `models.ModelProvider` type in the function signature, though not actively used for logic in this specific prompt.
- **External Libraries:** None.
- **Interactions:**
    - Provides a fixed system prompt for the Summarizer agent.
    - This prompt is used by the `internal/llm/agent` package when an agent of type `config.AgentSummarizer` is created and used for summarization tasks.
    - The content of the prompt directly influences how the LLM performs conversation summarization, guiding its focus and output style.
