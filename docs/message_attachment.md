# attachment.go (in internal/message)

## Overview

The `attachment.go` file, part of the `internal/message` package, defines a single struct: `Attachment`. This struct is used to represent a file attachment that can be associated with a message, likely for sending to an LLM that supports multimodal inputs (e.g., images, documents alongside text prompts).

## Key Components

### Structs
- `Attachment`: Represents a file attachment.
    - `FilePath (string)`: The original path to the file on the local system. This might be used for reference or if the LLM needs to know the source location.
    - `FileName (string)`: The name of the file (e.g., "image.png", "document.pdf"). This is often displayed in UIs or used by the LLM.
    - `MimeType (string)`: The MIME type of the attachment (e.g., "image/png", "application/pdf"), which is crucial for the LLM or receiving system to correctly interpret the content.
    - `Content ([]byte)`: The actual binary content of the file attachment.

## Important Variables/Constants

This file defines no package-level constants or variables, only the `Attachment` struct.

## Usage Examples

This struct would be populated when a user attaches a file to a message they are sending to the AI.

```go
// import "github.com/opencode-ai/opencode/internal/message"
// import "os"

func createAttachment(filePath string) (*message.Attachment, error) {
    data, err := os.ReadFile(filePath)
    if err != nil {
        return nil, err
    }

    // Basic MIME type detection (a more robust solution would use http.DetectContentType or similar)
    var mimeType string
    switch strings.ToLower(filepath.Ext(filePath)) {
    case ".png":
        mimeType = "image/png"
    case ".jpg", ".jpeg":
        mimeType = "image/jpeg"
    case ".txt":
        mimeType = "text/plain"
    default:
        mimeType = "application/octet-stream" // Default binary type
    }

    return &message.Attachment{
        FilePath: filePath,
        FileName: filepath.Base(filePath),
        MimeType: mimeType,
        Content:  data,
    }, nil
}

// Later, when constructing a message:
// attachments := []message.Attachment{}
// if attachmentData, err := createAttachment("path/to/image.png"); err == nil {
//     attachments = append(attachments, *attachmentData)
// }
//
// // This 'attachments' slice would then be associated with a message.Message
// // or directly used when constructing the payload for an LLM API call.
```
The `Content []byte` would then be encoded (e.g., base64) if required by the specific LLM provider's API when sending binary data.

## Dependencies and Interactions

- **Internal Dependencies:** None from other OpenCode packages within this specific file. This struct is likely used by `message.go` (for `message.Message` or `message.ContentPart`) and by LLM provider implementations that handle multimodal inputs.
- **External Libraries:** None directly in this file. (Usage examples might use `os`, `path/filepath`, `strings` for populating it).
- **Interactions:**
    - Serves as a data structure to hold information and content for file attachments.
    - It's a passive data structure, primarily defined here to be used by other components that manage message creation, storage, and transmission to LLMs.
    - The `MimeType` is critical for correct interpretation by the receiving LLM or API.
