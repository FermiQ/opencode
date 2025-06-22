# images.go (in internal/tui/image)

## Overview

The `images.go` file, part of the `internal/tui/image` package, provides utilities related to image handling, specifically for display within a terminal user interface (TUI). Its main functions are to validate image file sizes against a limit and to generate a string representation of an image preview using ANSI escape codes for color, effectively rendering a low-resolution version of the image in the terminal.

## Key Components

### Functions
- `ValidateFileSize(filePath string, sizeLimit int64) (bool, error)`:
    - Checks if the file at `filePath` exceeds the given `sizeLimit` (in bytes).
    - Returns `true` if the file size is greater than `sizeLimit`, `false` otherwise.
    - Returns an error if file info cannot be obtained.
- `ToString(width int, img image.Image) string`:
    - Takes a target `width` (in terminal characters) and an `image.Image` object.
    - Resizes the input `img` to the specified `width` while maintaining aspect ratio, using `imaging.Lanczos` resampling filter.
    - Iterates through the resized image pixels, processing two vertical pixels at a time to create a single character cell in the output:
        - It uses the "upper half block" character (`▀`, U+2580).
        - The foreground color of this character is set to the color of the top pixel (`img.At(x, heightCounter)`).
        - The background color of this character is set to the color of the bottom pixel (`img.At(x, heightCounter+1)`).
        - If there's no bottom pixel (i.e., at the last row of an odd-height image), the background color defaults to the foreground color.
        - Colors are converted from `image.Color` to `colorful.Color` and then to `lipgloss.Color` via their hex representation.
    - Builds a string representation of the image using these colored half-block characters, with newlines after each row.
    - Returns the resulting ANSI-formatted string.
- `ImagePreview(width int, filename string) (string, error)`:
    - A convenience function that takes a target `width` and a `filename`.
    - Opens the image file specified by `filename`.
    - Decodes the image using `image.Decode` (which can handle various common image formats like PNG, JPEG, GIF).
    - Calls `ToString(width, img)` with the decoded image to get the ANSI string representation.
    - Returns the ANSI string and any error encountered during file opening or decoding.

## Important Variables/Constants
This file does not define exported package-level constants or variables.

## Usage Examples

Validating an image file size before processing:
```go
// import "github.com/opencode-ai/opencode/internal/tui/image"

// filePath := "path/to/my_image.png"
// const maxImageDisplaySize = 2 * 1024 * 1024 // 2MB

// isTooLarge, err := image.ValidateFileSize(filePath, maxImageDisplaySize)
// if err != nil {
//     // Handle error (e.g., file not found)
// }
// if isTooLarge {
//     fmt.Println("Image is too large to display.")
// } else {
//     // Proceed with preview generation
// }
```

Generating an image preview for the TUI:
```go
// import "github.com/opencode-ai/opencode/internal/tui/image"
// import "fmt"

// terminalWidth := 80 // Desired width of the preview in characters
// imagePath := "path/to/my_image.jpg"

// previewString, err := image.ImagePreview(terminalWidth, imagePath)
// if err != nil {
//     // Handle error (e.g., not an image, file not found)
//     fmt.Printf("Could not generate preview: %v\n", err)
// } else {
//     fmt.Println(previewString) // This will print the ANSI colored image to the terminal
// }
```
This `previewString` would then be rendered by a TUI component that supports ANSI escape codes.

## Dependencies and Interactions

- **Internal Dependencies:** None from other OpenCode packages.
- **External Libraries:**
    - `github.com/charmbracelet/lipgloss`: For applying ANSI colors (foreground, background) to text.
    - `github.com/disintegration/imaging`: For image resizing (`imaging.Resize`, `imaging.Lanczos`).
    - `github.com/lucasb-eyer/go-colorful`: For color manipulation and conversion (specifically to get hex values which `lipgloss` can use).
    - Standard Go libraries: `fmt`, `image` (and its sub-packages for decoding specific formats, which are registered via blank imports in typical image processing applications, though not explicitly shown here), `os`, `strings`.
- **Interactions:**
    - `ValidateFileSize` interacts with the filesystem to get file metadata.
    - `ImagePreview` and `ToString` perform image processing (opening, decoding, resizing) and then generate a text-based representation using ANSI color codes.
    - The quality of the terminal preview depends on the terminal's color support (TrueColor recommended for best results with `lipgloss`) and the font's rendering of the "upper half block" character.
    - This package is likely used by TUI components responsible for displaying message attachments or image previews.
