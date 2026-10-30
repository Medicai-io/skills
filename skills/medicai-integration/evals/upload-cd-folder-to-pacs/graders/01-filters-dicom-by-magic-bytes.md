---
type: llm
focus: trace
weight: 1
---

The script detects DICOM files by reading the leading bytes (DICM at offset 128, or the older [8,0] prefix) rather than by file extension, and skips dicomdir and dirfile.
