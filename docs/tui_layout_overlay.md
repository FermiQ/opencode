# overlay.go (in internal/tui/layout)

## Overview

The `overlay.go` file, found within the `internal/tui/layout` package, provides a utility function `PlaceOverlay`. This function is designed to render one string (potentially containing ANSI escape codes for styling, the "foreground" or `fg`) on top of another string (the "background" or `bg`). It allows specifying the x/y coordinates for placing the foreground and can optionally render a simple shadow effect beneath the foreground content. The implementation carefully handles ANSI escape codes to ensure correct width calculations and rendering. Much of the code is acknowledged as borrowed and modified from the `charmbracelet/lipgloss` library and related pull requests.

## Key Components

### Structs
- `whitespace`: An unexported struct used to manage the rendering of whitespace, potentially with custom characters or styles (though not extensively used in `PlaceOverlay`'s current public options).
    - `style (termenv.Style)`: Style for the whitespace.
    - `chars (string)`: Characters to use for filling whitespace.

### Types
- `WhitespaceOption (func(*whitespace))`: A functional option type for configuring `whitespace` (currently no public options are exposed that use this directly for `PlaceOverlay`).

### Functions
- `getLines(s string) (lines []string, widest int)`:
    - An unexported helper function that splits a string `s` into lines based on `\n`.
    - It also calculates and returns `widest`, the printable width of the longest line, correctly accounting for ANSI escape codes using `ansi.PrintableRuneWidth()`.
- `PlaceOverlay(x, y int, fg, bg string, shadow bool, opts ...WhitespaceOption) string`:
    - The main public function of this file.
    - Takes `x`, `y` coordinates for the top-left placement of `fg` onto `bg`.
    - `fg`: The foreground string (content to overlay).
    - `bg`: The background string (content to be overlaid).
    - `shadow (bool)`: If true, a simple shadow effect is rendered beneath the `fg` content.
        - The shadow is created by generating a new background (`shadowbg`) consisting of a slightly offset block of shadow characters (░) styled with theme colors. The `fg` is then placed on this `shadowbg`, and this composite becomes the new `fg` for placement onto the original `bg`.
    - `opts ...WhitespaceOption`: Optional whitespace styling (not directly used by the core overlay logic in this version).
    - **Logic**:
        1.  Calculates the dimensions (lines, width, height) of `fg` and `bg`.
        2.  If `shadow` is true, it first renders the `fg` onto a generated shadow background, then updates `fg`'s dimensions.
        3.  Clamps the `x`, `y` placement coordinates to ensure `fg` fits within `bg`. If `fg` is larger than `bg`, it currently returns `fg` (this behavior is noted with a "FIXME").
        4.  Iterates through each line of the `bg` string.
        5.  If the current `bg` line is outside the vertical placement of `fg`, the `bg` line is appended directly.
        6.  If the current `bg` line is where `fg` should be overlaid:
            - It takes the part of the `bg` line to the left of `fg`'s `x` position.
            - Appends the corresponding line from `fg`.
            - Takes the part of the `bg` line to the right of where `fg` ends.
            - It uses `ansi.PrintableRuneWidth` and custom `cutLeft` to handle character widths correctly with ANSI codes.
        7.  Constructs and returns the final overlaid string.
- `cutLeft(s string, cutWidth int) string`:
    - An unexported helper that cuts `cutWidth` printable characters from the left of string `s`, correctly handling ANSI escape codes. It uses `chAnsi.Cut` from `charmbracelet/x/ansi`.
- `max(a, b int) int`: Standard max function.
- `(w whitespace) render(width int) string`:
    - Method on `whitespace` to render a string of given `width` using `w.chars` and `w.style`. (Less directly used by `PlaceOverlay`'s main logic but available for whitespace options).

## Important Variables/Constants
This file does not define exported package-level constants or variables beyond the `WhitespaceOption` type.

## Usage Examples

This function is likely used by TUI components that need to render modal dialogs, pop-ups, tooltips, or any element that appears on top of existing content.

```go
// import "github.com/opencode-ai/opencode/internal/tui/layout"

// backgroundContent := "Line 1 of background\nLine 2 of background\nLine 3 is much longer\nLine 4"
// foregroundContent := "Overlay\nBox"

// // Place "Overlay Box" at (x=5, y=1) on the background with a shadow
// result := layout.PlaceOverlay(5, 1, foregroundContent, backgroundContent, true)

// fmt.Println(result)
// // This would print the backgroundContent with the foregroundContent (and its shadow)
// // rendered on top at the specified coordinates.
```

## Dependencies and Interactions

- **Internal Dependencies:**
    - `github.com/opencode-ai/opencode/internal/tui/styles`: For `styles.BaseStyle()`.
    - `github.com/opencode-ai/opencode/internal/tui/theme`: For `theme.CurrentTheme()` to get colors for shadow rendering.
    - `github.com/opencode-ai/opencode/internal/tui/util`: For `util.Clamp()`.
- **External Libraries:**
    - `github.com/charmbracelet/lipgloss`: For styling the shadow characters.
    - `github.com/charmbracelet/x/ansi`: For `chAnsi.Cut` to correctly cut strings with ANSI codes.
    - `github.com/muesli/ansi`: For `ansi.PrintableRuneWidth()` to calculate the visible width of strings with ANSI codes.
    - `github.com/muesli/reflow/truncate`: For `truncate.String()`.
    - `github.com/muesli/termenv`: For `termenv.Style` used in the `whitespace` struct.
- **Interactions:**
    - Provides a sophisticated way to layer ANSI-styled text blocks.
    - The core challenge it addresses is correctly calculating widths and composing strings when ANSI escape codes are present, as these codes affect rendering but not logical string length or printable width in the same way plain characters do.
    - The shadow effect is a simple visual enhancement achieved by rendering slightly offset, darker characters behind the foreground content before placing it on the main background.
    - It's a crucial utility for building complex, layered TUIs.
