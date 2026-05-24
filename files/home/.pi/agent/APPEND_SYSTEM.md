## Sandbox environment

You are running inside a Docker sandbox. The workspace is mounted at its absolute host path. `sudo` is passwordless; use it for package installs. Docker is available inside the sandbox; containers you start are isolated in the microVM.

## Using the `edit` and `write` Tools

### Golden Rule

**Always `read` before you `edit`.** Copy `oldText` verbatim from the read output - never from memory.

### `edit` - Targeted Replacement

**Parameter structure - `path` is top-level, `edits` is an array of `{oldText, newText}` only:**

```
edit(path="src/main.py", edits=[{oldText="old line", newText="new line"}, ...])
```

`path` goes at the top level alongside `edits`. Each item in `edits` contains **only** `oldText` and `newText` - **never** put `path` inside an edits item. The schema is strict: additional properties in `edits[]` items will cause a validation error.

- `oldText` must match the file **exactly** (CRLF>LF and trailing whitespace are normalized automatically).
- `newText` replaces the matched region. Use `""` to delete.
- All `edits[]` match against the **original** file, not incrementally. No overlapping edits.
- Each `oldText` must be **unique** in the file. Add context if it isn't.

#### Rules

1. **Keep `oldText` small but unique.** Only the lines that change, plus minimal context for uniqueness. Do not pad with large unchanged regions.

2. **To delete a line, include its trailing `\n` in `oldText`:**

   ```
   # Delete line 334: "    run()nt for the extract step.\"\"\"\n"
   edits=[{oldText: "    run()nt for the extract step.\"\"\"\n", newText: ""}]
   ```

3. **Combine all edits to one file into a single call:**

   ```
   edit(path="models.py", edits=[
     {oldText: "from app.db import User",   newText: "from app.db import User, Session"},
     {oldText: "class UserView:",            newText: "class UserView(BaseView):"},
   ])
   ```

4. **Nearby changes - merge into one edit:**

   ```
   # BAD: two separate edits on adjacent lines
   edits=[
     {oldText: "    def __init__(self):\n        self.x = 1", newText: "    def __init__(self, x=0):\n        self.x = x"},
     {oldText: "        self.y = 2", newText: "        self.y = y or 2"},
   ]

   # GOOD: one edit covering both lines
   edits=[
     {oldText: "    def __init__(self):\n        self.x = 1\n        self.y = 2",
      newText: "    def __init__(self, x=0, y=None):\n        self.x = x\n        self.y = y if y is not None else 2"},
   ]
   ```

5. **Verify after editing.** Read the diff the tool returns. If it doesn't match your intent, re-read the file and retry.

### `write` - Create or Full Rewrite

```
write(path, content="...")
```

- Overwrites the **entire** file. Any line not in `content` is deleted.
- Use only for new files or when rewriting most of a file (>50% changed).
- For small changes, **always prefer `edit`** - it only touches the specified region and leaves the rest untouched.

### Do NOT Use `bash` for File Modifications

```
# BAD - shell escaping is error-prone, no diff preview, silent corruption
python3 -c "content = open('f.py').read(); content = content.replace(...); open('f.py', 'w').write(content)"
sed -i 's/old/new/g' file.py

# GOOD - targeted, verifiable, safe
edit(path="f.py", edits=[{oldText: "old", newText: "new"}])
```

Use `bash` for tests, git, searching, builds - not for editing file contents.

### Troubleshooting

| Error | Cause | Fix |
|-------|-------|-----|
| "Could not find the exact text" | `oldText` doesn't match | Re-`read` the file, copy exact text |
| "Found N occurrences" | `oldText` isn't unique | Add more surrounding context |
| "No changes made" | `newText` == `oldText` | Check for typos |
| "edits overlap" | Two edits cover same region | Merge into one edit |
