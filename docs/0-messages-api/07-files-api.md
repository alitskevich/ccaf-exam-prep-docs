# Files API vs Inline Content

Use the **Files API** when a document is referenced across many turns or in many requests; use **inline content** for one-shot inputs that won't be reused. Inline base64 in chat history bloats every subsequent request.

### Key Concepts

| If you need... | Reach for | Why |
|---|---|---|
| Same doc referenced across many turns | **Files API** | Upload once, ID-only after |
| One-shot input, never reused | Inline content | Simpler, no upload step |
| Mixed text + visuals from PDFs | **Files API** + PDF support | Native rendering, cached upload |

- File IDs are workspace-scoped; sharing across orgs requires re-upload
- For PDFs especially, Files API + native PDF support beats running an OCR pipeline
- **ZDR not eligible** — files persist on Anthropic infrastructure until you delete them via `client.beta.files.delete(id)`; set retention conservatively for sensitive content

### How to

```python
# One-shot — inline
client.messages.create(model=M, messages=[{
    "role": "user",
    "content": [{"type": "document",
                 "source": {"type": "base64",
                            "media_type": "application/pdf",
                            "data": pdf_b64}}],
}])

# Reused doc — upload once, reference by ID
file = client.beta.files.upload(
    file=("contract.pdf", pdf_bytes, "application/pdf"))
client.beta.messages.create(model=M, messages=[{
    "role": "user",
    "content": [{"type": "document",
                 "source": {"type": "file", "file_id": file.id}},
                {"type": "text", "text": "Summarize section 3."}],
}])
```

- **Switch to Files API the moment a doc is referenced twice.**
- **Pair Files API with prompt caching** when the file sits at the start of a long, stable prompt — uploading deduplicates bytes, caching deduplicates tokens
- **Document the file ID lifecycle in code** (e.g., cleanup on session close) so sensitive uploads don't persist server-side

### Anti-patterns

- Re-uploading the same PDF on every request — use the **Files API** ❌
- Inline base64 in chat history that's referenced more than once — bloats every subsequent request ❌
- Forgetting cleanup of sensitive Files API uploads — they persist server-side until deleted ❌
- Assuming Files API IDs are portable across workspaces — they aren't; re-upload required ❌
