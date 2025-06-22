# filepicker.go (in internal/tui/components/dialog)

## Overview

The `filepicker.go` file defines the `filepickerCmp` TUI component. This component provides a dialog for browsing directories and selecting files, primarily intended for use as an attachment picker in the chat interface. It displays a list of files and directories, allows navigation up and down the directory tree, and shows a preview for supported image files. Users can also manually type a path.

## Key Components

- **Constants**:
    - `maxAttachmentSize (5MB)`: Size limit for attachments.
- **`FilePrickerKeyMap` struct**: Defines key bindings for navigation (up, down, enter dir, go back), opening the picker, closing, and manual path input.
- **`filepickerCmp` struct**: The main `tea.Model` for the file picker.
    - Manages `basePath`, `width`, `height`, `cursor` position in the list, current directory entries (`dirs`), a `cursorChain` (stack) to remember cursor positions when navigating directories, a `viewport` for image previews, `selectedFile` path, a `textinput.Model` (`cwd`) for manual path input, and a `ShowFilePicker` flag.
    - `app (*app.App)`: Reference to the main application.
- **`DirNode` struct**: Represents a node in the directory navigation tree (current directory and its parent/child).
- **`stack []int`**: Helper type for `cursorChain`.
- **Message Types**:
    - `AttachmentAddedMsg`: Sent when a file is successfully selected and processed as an attachment. Contains `message.Attachment`.
- **Core Functionality**:
    - `Init()`: Returns `nil`.
    - `Update(msg tea.Msg)`: Handles messages:
        - `tea.WindowSizeMsg`: Adjusts component dimensions.
        - `tea.KeyMsg`:
            - If `cwd` text input is focused, forwards keys to it.
            - Navigation keys (`up`, `down`, `l` for forward, `h`/`backspace` for backward) update `cursor`, `cwdDetails`, and `dirs`.
            - `enter`: If `cwd` is focused, tries to navigate to the typed path. If list is focused, enters selected directory or calls `addAttachmentToMessage()` for a file.
            - `esc`: Clears cursor chain or blurs `cwd` input.
            - `i`: Focuses the `cwd` text input for manual path entry.
            - `ctrl+f` (OpenFilePicker): Refreshes the current directory view.
    - `View() string`: Renders the file picker UI, including:
        - The current path (editable via `cwd` text input).
        - A list of files/directories in the current path, highlighting the `cursor` item.
        - An image preview in a separate `viewport` if a supported image is selected.
        - Help text for path input.
    - `addAttachmentToMessage()`: Validates the `selectedFile` (model support, extension, size), reads its content, determines MIME type, and sends an `AttachmentAddedMsg`.
    - `getCurrentFileBelowCursor()`: Updates the image preview viewport when the cursor moves to a supported image file.
- **Helper Functions**:
    - `readDir()`: Reads directory contents, sorts them (dirs first, then by name), filters out hidden files unless `showHidden` is true, and only includes supported image files or directories. Includes a timeout for reading.
    - `IsHidden()`: Checks if a file is hidden (starts with ".").
    - `isExtSupported()`: Checks if a file extension is a supported image type (jpg, jpeg, webp, png).
- **`FilepickerCmp` interface & `NewFilepickerCmp()` constructor**: Standard component setup.

## Dependencies and Interactions

- Uses `app.App` to get model capabilities (attachment support).
- Uses `config.GetSelectedModel` and `config.WorkingDirectory`.
- Uses `image.ValidateFileSize` and `image.ImagePreview` from `internal/tui/image` for file validation and preview rendering.
- Uses `message.Attachment` from `internal/message`.
- Uses `textinput.Model` and `viewport.Model` from `charmbracelet/bubbles`.
- Uses `lipgloss` for styling.
- Communicates selection via `AttachmentAddedMsg`.

## Purpose

This component enables users to navigate their filesystem within the TUI to select files, primarily images, to be attached to chat messages. It provides visual feedback with image previews and handles basic validation.
