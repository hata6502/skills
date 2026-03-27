---
name: codex-session-export
description: Export current Codex CLI session logs to readable Markdown files from `~/.codex/sessions/**/*.jsonl`. Use when a user wants to share, archive, inspect, or hand off a Codex session, whether exporting the latest/current session or a specific session id.
---

# Codex Session Export

## Overview

Export current Codex session logs to Markdown with inline `jq` commands only. Assume the current `jsonl` storage format with `session_meta` and `event_msg` entries.

Default behavior:

- Write the export to `~/Downloads` only.
- Unless the user explicitly asks for the `jq` commands, run them immediately instead of pasting them back first.
- If writing to `~/Downloads` needs approval in the current sandbox, request approval and keep the destination fixed to `~/Downloads`.
- After writing, return the output path. Only show a content preview when the user asks for one.

## Workflow

1. Resolve the source `jsonl` file.
- For the latest session, use the newest `~/.codex/sessions/**/*.jsonl`.
- For a known session id, find the matching `jsonl` file under `~/.codex/sessions`.
- Fail immediately if `src` is empty or the file does not exist.

2. Build the output file path.
- Always use `~/Downloads`.
- Use `codex-session_<timestamp>_<first-user-message>_<short-id>.md`.
- Create `~/Downloads` if needed.

3. Run the export command.
- Read `session_meta` for id and timestamp.
- Read `event_msg` entries for user and assistant messages.
- Fail immediately if either structure is missing, because that means the local format likely changed.
- Run the export immediately for normal "export this session" requests.
- Only return the `jq` commands themselves when the user explicitly asks for the commands or wants to inspect them before execution.

4. Review before sharing.
- The transcript includes user and assistant messages only.
- The transcript excludes tool payloads and shell output.
- Check for pasted secrets or private content.

## Path Resolution

### Latest session

```sh
src="$(find ~/.codex/sessions -type f -name '*.jsonl' -print | sort -r | head -n1)"
[ -n "$src" ] && [ -f "$src" ] || { echo "No session jsonl found." >&2; exit 1; }
```

### Specific session id

```sh
session_id="019d2f73-9190-7ac1-ba2a-fa68f7f4e834"
src="$(find ~/.codex/sessions -type f -name "*${session_id}*.jsonl" -print | sort -r | head -n1)"
[ -n "$src" ] && [ -f "$src" ] || { echo "No matching session jsonl found." >&2; exit 1; }
```

## Export Commands

Use these internally when executing the export. Only paste them back to the user when they explicitly ask for the commands.

```sh
## 1. Validate the file shape
jq -e -sr 'map(select(.type=="session_meta")) | length > 0' "$src" >/dev/null \
  || { echo "session_meta not found; update the skill for the new format." >&2; exit 1; }

jq -e -sr '[ .[]
  | select(.type == "event_msg" and (.payload.type == "user_message" or .payload.type == "agent_message"))
] | length > 0' "$src" >/dev/null \
  || { echo "event_msg conversation entries not found; update the skill for the new format." >&2; exit 1; }
```

```sh
## 2. Build the output path

mkdir -p "$HOME/Downloads"

timestamp="$(
  jq -sr 'map(select(.type=="session_meta"))[0].payload.timestamp
    | sub("\\.[0-9]+Z$"; "Z")
    | gsub("T"; "_")
    | gsub(":"; "-")
    | sub("Z$"; "")' "$src"
)"

title="$(
  jq -sr '
    ([ .[]
       | select(.type == "event_msg" and .payload.type == "user_message")
       | .payload.message
     ][0] // "untitled")
    | gsub("[\r\n\t]"; " ")
    | gsub("/"; "／")
    | gsub(":"; "：")
    | gsub("[\\\\*?\"<>|]"; "_")
    | gsub("[[:space:]]+"; "-")
    | .[0:28]
    | if length == 0 then "untitled" else . end
  ' "$src"
)"

short_id="$(jq -sr 'map(select(.type=="session_meta"))[0].payload.id | split("-")[0]' "$src")"
out="$HOME/Downloads/codex-session_${timestamp}_${title}_${short_id}.md"
```

```sh
## 3. Render the Markdown

jq -sr '
  . as $rows
  | ($rows | map(select(.type=="session_meta"))[0].payload) as $meta
  | "# Codex Session\n\n"
    + "- Session ID: `" + $meta.id + "`\n"
    + "- Timestamp: `" + $meta.timestamp + "`\n\n"
    + (
        [ $rows[]
          | select(.type == "event_msg")
          | if .payload.type == "user_message" then
              "## User\n\n" + .payload.message + "\n"
            elif .payload.type == "agent_message" then
              "## Assistant\n\n" + .payload.message + "\n"
            else
              empty
            end
        ]
        | join("\n")
      )
' "$src" > "$out"
```

```sh
## 4. Return the result

echo "$out"
```

### Optional preview

Only run this when the user asks to inspect the exported Markdown.

```sh
sed -n '1,24p' "$out"
```

## Notes

- Do not mention legacy `.json` sessions in normal usage.
- If the local Codex format changes again, update this skill instead of adding back old-format branches.
