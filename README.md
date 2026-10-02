# MCP Tool Poisoning Scanner

Paste an MCP server's tool definitions and statically scan them for prompt injection, hidden characters, tool shadowing, and exfiltration-shaped parameters before you install it.

**Live demo:** https://0xelitesystem.github.io/mcp-tool-poisoning-scanner/

## What tool poisoning is

An MCP server advertises its tools through `tools/list`. Every tool carries a name, a description, and an input schema, and all of that text goes into the model's context. The human operator usually sees the tool name and clicks approve. The model reads the whole description.

That gap is the attack. A server can put instructions to the agent inside a description: call this tool first, use it instead of the one the user picked, pass along the contents of an environment variable, do not mention any of this. The text can be padded out so nobody reads to the end, or hidden behind zero-width characters so there is nothing to read at all.

This tool reads those definitions the way the model does, and reports what it finds.

## Use

1. Get the server's tool definitions: a JSON-RPC `tools/list` response, an object with a `tools` array, a bare array of tools, or a single tool object. Or press **Load sample**.
2. Paste it into the box and press **Scan definitions**.
3. Read each finding: severity, tool, JSON path, the offending excerpt and what to do about it. The **Reveal hidden characters** panel shows every string with its invisible, control or confusable characters replaced by labeled markers.
4. Press **Copy report** or **Copy findings as JSON** to keep the result.

## Why this exists

An operator approving an MCP server usually sees tool names, while the model reads the full descriptions, and that is where injected instructions hide. This scanner reads the definitions the way the model does, before you install the server. It is one HTML file that runs in your browser, with no tracking and no server, under the MIT license.

## Features

- **Tolerant input.** Accepts a JSON-RPC `tools/list` response, an object with a `tools` array, a bare array of tool definitions, or a single tool object. It reports which shape it found. Trailing commas and `//` comments are stripped before parsing so a copied source snippet still works. Nested `inputSchema` properties, `items`, and `oneOf` / `anyOf` / `allOf` branches are walked recursively.
- **Eleven named checks**, each producing findings with a severity, the exact tool, the JSON path, the offending excerpt, why it matters, and what to do about it:
  - Agent-directed imperatives (`always`, `you must first`, `before using any other tool`, `ignore previous instructions`, `without asking the user`)
  - Concealment language (`do not tell the user`, `keep this hidden`, `omit this from your response`)
  - Cross-tool interference: descriptions that redefine, discredit, or claim priority over other tools, plus any mention of another tool by name. This is the shadowing pattern.
  - Invisible and confusable characters: zero-width space, joiner and non-joiner, word joiner, soft hyphen, no-break and other exotic spaces, all eleven bidirectional controls, variation selectors, Unicode tag characters, private-use and control code points, and roughly seventy Cyrillic, Greek, fullwidth, and letterlike homoglyphs of ASCII letters. Each finding reports the code point, its Unicode name, and its offset.
  - Hidden characters delivered as literal `\u` escapes in the raw payload, which stay invisible even when you read the decoded string.
  - Exfiltration-shaped parameters: `callback_url`, `webhook`, `endpoint`, `upload_to`, `notify`, `forward`, `base_url`, `proxy`, `send_to` and relatives, plus any parameter documented as a place to paste file contents, environment variables, credentials, or conversation history.
  - Over-broad inputs: free-form `command`, `query`, `path`, `script`, `code`, and `sql` parameters, `additionalProperties` not set to `false`, closed-set parameters with no `enum`, strings with no `maxLength`, arrays with no `items` or `maxItems`, a missing `required` array, unconstrained objects, and excessive nesting.
  - Name and description mismatch: a tool named `read`, `get`, `list`, or `search` whose description or parameters imply writing, sending, deleting, or executing. Negated phrasing ("cannot delete") is ignored.
  - Prompt markup inside descriptions: fenced code blocks, HTML and XML tags, chat-template turn markers, and fake `system:` or `Assistant:` turns.
  - Bulk signals: very long descriptions, embedded URLs, encoded blobs, long hex runs, missing descriptions, and non-ASCII tool names.
  - Collection-level signals: duplicate tool names and unusually large tool counts.
- **Reveal hidden characters.** Every string containing an invisible or confusable character is re-rendered with each one replaced by a labeled marker, so you can see exactly what is stored. Runs of Unicode tag characters are decoded back into the ASCII message they spell out.
- **Per-tool risk score and an overall verdict**, with a plain summary sentence, findings grouped by severity, and a collapsible card per tool.
- **Copyable plain-text report** suitable for pasting into a security review or a GitHub issue, plus a findings export as JSON.
- **Load sample** builds a realistic `tools/list` with one honest tool and three poisoned ones: one hides its instruction behind zero-width characters, tag characters, a bidi override, and a Cyrillic homoglyph in a property key, one carries an exfiltration webhook parameter and a description that shadows the other tools, and one is a name collision with the honest tool. All eleven check classes fire on it.

## How it works

The scanner is one HTML file with no external dependencies. All logic lives in pure functions that take data and return a result object, with the DOM layer wired on top:

- `parseToolDefinitions(text)` detects the payload shape and returns the tool list plus the JSON path of each tool.
- `findHiddenCharacters(str)` walks the string by code point and classifies each one against a code-point table plus range rules for tag characters, variation selectors, private-use areas, fullwidth forms, and mathematical alphanumerics.
- `revealHiddenCharacters(str)` returns a segment list where each flagged character becomes a marker object, which the renderer turns into a visible chip.
- `checkAgentImperatives`, `checkConcealmentLanguage`, `checkCrossToolInterference`, `checkSuspiciousMarkup`, `checkExfiltrationSignals`, `checkOverBroadInputs`, `checkNameDescriptionMismatch`, `checkBulkSignals`, and `checkToolCollection` each return their own findings.
- `scanToolDefinitions(text)` runs all of them, deduplicates, sorts, scores, and returns the whole result.
- `buildTextReport(result)` renders that result as plain text.

Scoring weights each finding (high 25, medium 9, low 3) and caps the total at 100. Bands: 0 clean, 1 to 14 low concern, 15 to 39 review before installing, 40 to 69 high risk, 70 and above treat as hostile. The score is a triage aid, not a measurement.

Every non-ASCII character the scanner knows about is written into the source as an escape sequence or built from its code point, so the file itself is pure ASCII and can be reviewed in any editor without the same trick being played on you.

## Honest limits

This is a static heuristic linter for tool **definitions**. Read this part before you trust a green result.

- It cannot see what the server does at runtime. A definition is a promise, not behaviour.
- It cannot detect a server that behaves well until it returns malicious content in tool **results**. Poisoned results are a real and separate attack, and nothing here touches it.
- Definitions can change after you audit them. A server can serve a clean list today and a poisoned one on the next connection. Nothing about a past scan constrains a future response.
- Heuristics produce false positives, and novel phrasing will slip past them. Several checks flag things that are merely schema hygiene, not attacks.
- A clean report is not a safety guarantee.

Pin the server version you reviewed, and re-scan on every update.

## Privacy

Everything runs in your browser. The JSON you paste is parsed, scanned, and rendered locally and never leaves your machine. There is no build step, no analytics, no network call, and no third-party code of any kind. Verify by viewing the page source, or by opening DevTools and watching the network tab: no requests are made.

For definitions from a server you already distrust, save the file and open it offline.

The only thing written to storage is your light or dark theme choice, saved in `localStorage` under the key `mcp-tps-theme`.

## Run locally

```bash
git clone https://github.com/0xelitesystem/mcp-tool-poisoning-scanner
cd mcp-tool-poisoning-scanner
# Open index.html in your browser, or:
python -m http.server 8000
```

## Build

No build step. The whole tool is one `index.html` file with its CSS and JavaScript inline, and nothing to install.

## Related work

- [mcp-server-registry](https://github.com/0xelitesystem/mcp-server-registry), a public registry of MCP servers. This scanner is the vetting step that sits in front of it.
- [system-prompt-leak-tester](https://github.com/0xelitesystem/system-prompt-leak-tester), the same idea pointed at the other end of the context window.

- [mcp-server-privilege-inventory](https://0xelitesystem.github.io/mcp-server-privilege-inventory/), works one level up: this scanner reads tool definitions, that one inventories privilege across every server you installed.

## License

MIT.

## More

- [The full catalog](https://0xelitesystem.github.io/), every tool and reference in one place.
- [elitesystem.ai](https://elitesystem.ai)
