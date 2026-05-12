# vibe-skill

A Claude Code skill that delegates coding tasks to **Mistral Vibe** and supervises the result.

Claude orchestrates. Vibe codes. You review the diff.

---

## What it does

When you type `/vibe <instruction>` in Claude Code, this skill:

1. Decomposes the task into atomic sub-tasks (if needed)
2. Writes a self-contained prompt for each sub-task
3. Runs `vibe-delegate` — a shell script that launches Vibe programmatically in a pseudo-TTY
4. Streams structured output: `[read]`, `[write]`, `[WARN]`, `[SYNTAX ERROR]`
5. Runs post-run syntax checks on modified `.py` and `.js` files automatically
6. Reports the git diff and any issues to you

---

## Prerequisites

- [Mistral Vibe](https://vibe.mistral.ai/) CLI installed and authenticated (`vibe --version`)
- [Claude Code](https://claude.ai/code) with skills enabled
- `script` command available (GNU coreutils — comes with Linux; on macOS use `brew install util-linux`)
- `python3` available (for the streaming parser and syntax checks)
- `node` available (optional — for JS syntax checks)
- A git repository to work in

---

## Installation

```bash
# 1. Clone this repo
git clone https://github.com/pcx-wave/vibe-skill.git
cd vibe-skill

# 2. Install the delegate script
cp tools/vibe-delegate ~/tools/vibe-delegate
chmod +x ~/tools/vibe-delegate

# (create ~/tools if needed)
mkdir -p ~/tools

# 3. Install the skill for Claude Code
mkdir -p ~/.claude/skills/vibe
cp SKILL.md ~/.claude/skills/vibe/SKILL.md

# 4. Edit the "Known projects" table in ~/.claude/skills/vibe/SKILL.md
#    to list your own projects with their paths.
```

### Verify the install

```bash
# Check vibe is available
vibe --version

# Test the delegate script
~/tools/vibe-delegate /tmp "Say hello in one sentence." 3
# Should print: [vibe] Hello! ...
```

---

## Usage

In a Claude Code session, just describe what you want:

```
/vibe add a dark mode toggle to the settings page
```

```
/vibe the login form is not validating the email field — fix it
```

```
/vibe add pagination to the GET /posts route, 20 items per page
```

Claude will decompose the task, write the Vibe prompt, supervise execution, and report the diff.

---

## How vibe-delegate works

```
Claude Code
  └─ /vibe <instruction>
       └─ SKILL.md logic
            └─ ~/tools/vibe-delegate <workdir> <prompt> [turns] [agent] [timeout]
                 ├─ writes prompt to temp file (avoids shell injection with UTF-8/emoji)
                 ├─ generates a temp shell script for the vibe command
                 ├─ runs: script -q -c "<vibe-script>" /dev/null
                 │         └─ allocates pseudo-TTY (required — vibe hangs without one)
                 ├─ pipes JSON streaming output through Python parser
                 │         └─ prints [read] / [write] / [WARN] / [vibe] lines
                 ├─ runs syntax checks on modified .py and .js files
                 └─ prints git diff --stat
```

### Why the pseudo-TTY?

Vibe checks for a TTY on startup. Without one (when piped directly), it hangs
indefinitely — 0 tool calls, silent timeout. The `script -q -c "..." /dev/null`
trick allocates a pseudo-TTY without writing a typescript file.

### Why prompt via temp file?

Inline shell arguments break when the prompt contains Python dict syntax (`:` followed
by a space), emojis, accented characters, or multi-line code. Writing to a temp file
and using `printf '%q'` avoids all shell injection issues.

---

## Token economics

Vibe's internal turns (file reads, search/replace attempts) consume **Mistral tokens**,
not Claude tokens. Claude only receives the compressed final output (~500–1500 tokens/run).

For a task with 6 reads of an 800-line file: ~4800 tokens on Mistral's side, 0 on Claude's.
Real advantage on exploratory/implementation tasks. Neutral if Vibe fails and produces
long error output.

---

## Customization

Edit `~/.claude/skills/vibe/SKILL.md`:

- **Known projects table** — list your repos with their absolute paths
- **Max turns** — adjust defaults per project complexity
- **Agents** — configure preferred agents for specific task types

---

## Known bugs in Vibe (and workarounds)

| Bug | Workaround |
|-----|-----------|
| `search_replace failed` on UTF-8/emoji | Edit with `python3 str.replace()` instead |
| Duplicated code block at end of file | `git diff`, delete the duplicate manually |
| Variable declared twice in same scope | Grep the var name before relaunching |
| Prompt truncated silently | Use the temp-file path (already default in vibe-delegate) |

See `examples/anti-patterns.md` for full examples with root causes.

---

## Examples

- `examples/good-prompts.md` — prompt patterns that reliably work
- `examples/anti-patterns.md` — what fails and why, with fixes

---

## Feedback

See `/tmp/retour_claude_vibe.txt` in this repo for the original feedback from Claude
that led to the improvements in `vibe-delegate`.

---

## License

MIT
