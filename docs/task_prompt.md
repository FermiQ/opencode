# task.go (in internal/llm/prompt)

## Overview

The `task.go` file, within the `internal/llm/prompt` package, is responsible for generating the system prompt for the "Task" agent. This type of agent is typically used for more focused, information-retrieval sub-tasks, often invoked by another agent (like the "Coder" agent via the `AgentTool`). The prompt emphasizes conciseness and directness in responses.

## Key Components

### Functions
- `TaskPrompt(_ models.ModelProvider) string`:
    - This function constructs and returns the system prompt for the Task agent.
    - Similar to `SummarizerPrompt`, the `models.ModelProvider` argument is present but not currently used to alter the prompt content, meaning the Task agent's base instructions are the same regardless of the LLM provider.
    - The core instructions for the Task agent are:
        - Role: "You are an agent for OpenCode."
        - Objective: "Given the user's prompt, you should use the tools available to you to answer the user's question."
        - Key Behavioral Notes:
            1.  **Conciseness**: Emphasizes being "concise, direct, and to the point" because responses are for a command-line interface. It explicitly states, "One word answers are best," and to avoid preambles, conclusions, or unnecessary explanations.
            2.  **Relevance**: Instructs to share relevant file names and code snippets.
            3.  **Absolute Paths**: Mandates that any file paths returned in the final response MUST be absolute.
    - After defining these core instructions, the function calls `getEnvironmentInfo()` (from `coder.go` in the same package, but used here too) to append information about the current working directory, git status, OS, date, and a directory listing. This provides the Task agent with immediate context about its operating environment.
    - The final prompt is a combination of the core instructions and the environment information.

## Important Variables/Constants

This file does not define any exported package-level constants or variables beyond the `TaskPrompt` function. The prompt itself is constructed from string literals and the output of `getEnvironmentInfo()`.

## Usage Examples

The `TaskPrompt` function is called by `GetAgentPrompt` (in `prompt.go`) when the agent name is `config.AgentTask`. This system prompt is then used when a Task agent instance is created, for example, by the `AgentTool` when the Coder agent delegates a sub-task.

```go
// Conceptual usage by GetAgentPrompt:
// import "github.com/opencode-ai/opencode/internal/llm/prompt"
// import "github.com/opencode-ai/opencode/internal/config"
// import "github.com/opencode-ai/opencode/internal/llm/models"

// agentType := config.AgentTask
// providerType := models.ProviderOpenAI // or any other provider
//
// systemPromptForTaskAgent := prompt.GetAgentPrompt(agentType, providerType)
// // systemPromptForTaskAgent will contain the text defined in TaskPrompt(),
// // including the environment information.
```
This prompt guides the sub-agent created by `AgentTool` in its focused task execution.

## Dependencies and Interactions

- **Internal Dependencies:**
    - `github.com/opencode-ai/opencode/internal/llm/models`: For the `models.ModelProvider` type in the function signature.
    - Relies on `getEnvironmentInfo()` (defined in `coder.go` but accessible within the same package) to include dynamic environment details in the prompt.
- **External Libraries:**
    - `fmt`: For string formatting.
- **Interactions:**
    - Provides a specialized system prompt for Task agents, emphasizing brevity and directness suitable for sub-tasks that primarily retrieve and return information.
    - This prompt is used by the `internal/llm/agent` package when an agent of type `config.AgentTask` is created.
    - The inclusion of environment information helps the Task agent operate with immediate awareness of its context, even though it might be invoked for a very specific, narrow query.
