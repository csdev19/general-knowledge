# HTTP APIs and CLIs — verbatim text

**Status:** probed.

The evidence is the command and its verbatim output, in a code block. It is searchable, copyable
and diffable; an image of it is none of these.

```bash
curl -sS -i -X POST http://localhost:3000/api/items -H 'content-type: application/json' -d '{"name":"x"}' \
  > .evidence/<topic>/create-item.txt 2>&1; echo "exit: $?"
```

- `-i` keeps status line and headers; avoid `-v` in posted evidence, it prints request headers
  including `Authorization` and cookies.
- Redact tokens, cookies and personal data before posting.
- Only when an image is asked for: render the text into an HTML `<pre>` and screenshot it with the
  [web recipe](./web-playwright.md); no extra tool to install.
