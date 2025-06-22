# models.go (in internal/tui/components/dialog)

## Overview

The `models.go` file (within `internal/tui/components/dialog`) defines the `ModelDialogCmp` TUI component. This dialog allows users to select an AI model for use by the Coder agent. It displays a list of available models, grouped by provider (e.g., OpenAI, Anthropic, Local). Users can navigate between providers (horizontally) and models within a provider (vertically).

## Key Components

- **Message Types**:
    - `ModelSelectedMsg`: Sent when a user selects a model. Contains the selected `models.Model`.
    - `CloseModelDialogMsg`: Sent when the dialog is closed without selection.
- **`ModelDialog` interface**: Public interface for the model selection dialog.
- **`modelDialogCmp` struct**: The main `tea.Model` for the dialog.
    - Manages a list of `models.Model` for the currently selected `provider`.
    - Tracks `availableProviders`, `selectedIdx` (for the model list), `scrollOffset` (for vertical scrolling of models), `hScrollOffset` (for horizontal scrolling through providers), and `hScrollPossible` flag.
- **`modelKeyMap` struct**: Defines key bindings for navigation (Up/Down/J/K for models, Left/Right/H/L for providers), selection (Enter), and closing (Escape).
- **Core Functionality**:
    - `Init()`: Calls `setupModels()` to load initial model and provider lists.
    - `Update(msg tea.Msg)`: Handles messages:
        - `tea.KeyMsg`: Processes navigation and selection keys.
            - Up/Down/J/K: Calls `moveSelectionUp()` or `moveSelectionDown()`.
            - Left/Right/H/L: Calls `switchProvider()` if horizontal scrolling is possible.
            - Enter: Sends `ModelSelectedMsg` with the selected model.
            - Escape: Sends `CloseModelDialogMsg`.
    - `View() string`: Renders the dialog:
        - Displays the current provider's name as a title.
        - Lists models for the current provider (up to `numVisibleModels`).
        - Highlights the `selectedIdx` model.
        - Shows scroll indicators (`↑`, `↓`, `←`, `→`) if more models/providers are available.
    - `setupModels()`: Initializes the list of available providers and sets up models for the initially selected provider (based on current config).
    - `setupModelsForProvider(provider models.ModelProvider)`: Filters `models.SupportedModels` for the given provider, sorts them (reverse alphabetical by name), and resets selection/scroll state.
- **Helper Functions**:
    - `moveSelectionUp()`, `moveSelectionDown()`: Handle vertical navigation within the model list, including wrapping and adjusting `scrollOffset`.
    - `switchProvider(offset int)`: Changes the current provider based on horizontal navigation, updates `hScrollOffset`, and calls `setupModelsForProvider`.
    - `getScrollIndicators()`: Generates the string for scroll indicators.
    - `GetSelectedModel(cfg *config.Config)`: Utility to get the currently configured Coder agent's model.
    - `getEnabledProviders(cfg *config.Config)`: Gets a list of enabled providers from config, sorted by `models.ProviderPopularity`.
    - `findProviderIndex(...)`: Finds the index of a provider in a list.
    - `getModelsForProvider(...)`: Retrieves and sorts models for a specific provider.
- `NewModelDialogCmp() ModelDialog`: Constructor.

## Dependencies and Interactions

- Uses `models.Model`, `models.ModelProvider`, `models.SupportedModels`, `models.ProviderPopularity` from `internal/llm/models`.
- Uses `config.Get()` and `config.AgentCoder` to determine the current/default model and provider.
- Interacts with `key.Binding` from `charmbracelet/bubbles`.
- Uses `lipgloss` for styling.
- Communicates selection or closure to a parent model via `ModelSelectedMsg` or `CloseModelDialogMsg`.

## Purpose

This component provides a user interface for browsing and selecting from a potentially large list of available AI models, grouped by their providers. It allows users to easily switch the model used by the Coder agent.
