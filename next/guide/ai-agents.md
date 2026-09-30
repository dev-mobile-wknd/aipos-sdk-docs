# Using with AI agents

These docs are also available in formats that AI coding assistants — Claude Code,
Cursor, Codex, or ChatGPT — can read directly. That way the agent writes code against
the SDK's real API instead of guessing function names.

| What you need | Use |
|---|---|
| A coding agent that knows the SDK in every session | [Agent skill](#agent-skill) |
| The whole documentation as context | [`llms.txt`](#llmstxt) |
| Ask questions about a single page | [Copy page menu](#copy-page-menu) |

## Agent skill

`SKILL.md` is an SDK summary written for agents: choosing an artifact, installation,
assembling the SDK, rules that prevent the most common bugs, and Swift differences.
The agent loads it automatically when you work on code that uses `com.weekendinc.aipos`.

For **Claude Code**, save it in your project's skills folder:

```bash
mkdir -p .claude/skills/aipos-sdk
curl -fsSL https://dev-mobile-wknd.github.io/aipos-sdk-docs/skills/aipos-sdk/SKILL.md \
  -o .claude/skills/aipos-sdk/SKILL.md
```

Commit the `.claude/skills/` folder so the whole team uses the same skill. To use it
across all your projects, save it to `~/.claude/skills/aipos-sdk/SKILL.md` instead.

For other agents that don't support skills yet, paste the file's contents into their
instruction file — for example `AGENTS.md` or a Cursor project rule.

:::warning[Download it again whenever you upgrade the SDK]
The skill is written for a specific SDK version (currently 0.1.1). After bumping
the SDK version, run the `curl` command above again so the agent doesn't use an old API.
:::

## llms.txt

The entire documentation is available as plain Markdown following the
[llms.txt](https://llmstxt.org) standard:

| Address | Contents |
|---|---|
| [`/llms.txt`](https://dev-mobile-wknd.github.io/aipos-sdk-docs/llms.txt) | Every page with a one-sentence summary |
| [`/llms-full.txt`](https://dev-mobile-wknd.github.io/aipos-sdk-docs/llms-full.txt) | The whole documentation in one file |
| `<page address>.md` | A single page as Markdown, e.g. [`/guide/installation.md`](https://dev-mobile-wknd.github.io/aipos-sdk-docs/guide/installation.md) |

Give the `llms.txt` address to an agent that can browse the web, then ask it to read the
relevant pages. For agents without web access, attach `llms-full.txt` as a file.

The Indonesian version lives under `/id/`, for example
[`/id/llms.txt`](https://dev-mobile-wknd.github.io/aipos-sdk-docs/id/llms.txt).

## Copy page menu

Every documentation page has a **Copy page** button at the top right of the title:

| Action | Result |
|---|---|
| **Copy page** | Copies the page as Markdown, ready to paste into an AI chat |
| **View as Markdown** | Opens the page's Markdown version in a new tab |
| **Open in ChatGPT** | Opens ChatGPT with a prompt to read this page |
| **Open in Claude** | Opens Claude with a prompt to read this page |

## Tips

- **Mention the SDK version** you use. All `com.weekendinc.aipos:*` artifacts must be on
  the same version.
- **Don't paste secrets** — `llmApiKey`, `apiKey`, or the online payment password —
  into an AI chat. Use placeholders and keep the real values in `local.properties`.
- **Check agent-generated payment code** against the
  [Production checklist](https://dev-mobile-wknd.github.io/aipos-sdk-docs/guides/production-checklist) before releasing.
