---
name: codex-session-search
description: Search local Codex CLI session logs under `~/.codex/sessions/**/*.jsonl` to find the best past session to resume. Use when the user asks to resume, continue, find, search, or identify a previous Codex session by topic, repo, date, branch, PR, Slack thread, keyword, or related task, and you need to return candidate session IDs plus an exact `codex resume <id>` command.
---

# Codex Session Search

Use fixed `find` + `jq` commands to search `~/.codex/sessions`. Prefer these commands over ad-hoc parsing or custom scripts.

If the user wants a shareable Markdown transcript instead of a resume candidate, use `codex-session-export`.

## Key Rule

Search `event_msg.user_message` first.

Do not start with `response_item.role=user`, because that often includes AGENTS instructions and environment preamble that create false positives.

## List Recent Sessions In The Current Repo

```bash
cwd="$PWD"
find ~/.codex/sessions -type f -name '*.jsonl' | while read -r f; do
  jq -sr -r --arg cwd "$cwd" '
    (map(select(.type=="session_meta"))[0].payload) as $meta
    | [ .[]
        | select(.type=="event_msg" and .payload.type=="user_message")
        | .payload.message
      ] as $msgs
    | select(($meta.cwd // "") == $cwd)
    | [
        ($meta.timestamp // ""),
        ($meta.id // ""),
        (($msgs[0] // "") | gsub("[\r\n\t]+"; " ") | .[0:180])
      ]
    | @tsv
  ' "$f"
done | sort -r | head -n 20
```

## Search By Phrase In User Messages

Use this first when the user knows the original wording or topic.

```bash
phrase='phrase'
cwd="$PWD"
find ~/.codex/sessions -type f -name '*.jsonl' | while read -r f; do
  jq -sr -r --arg cwd "$cwd" --arg phrase "$phrase" '
    (map(select(.type=="session_meta"))[0].payload) as $meta
    | [ .[]
        | select(.type=="event_msg" and .payload.type=="user_message")
        | .payload.message
      ] as $msgs
    | select(($meta.cwd // "") == $cwd)
    | select((($msgs | join("\n")) | ascii_downcase) | contains($phrase | ascii_downcase))
    | [
        ($meta.timestamp // ""),
        ($meta.id // ""),
        (($msgs[0] // "") | gsub("[\r\n\t]+"; " ") | .[0:180]),
        ("codex resume " + ($meta.id // ""))
      ]
    | @tsv
  ' "$f"
done | sort -r
```

## Broaden To Transcript Search

If the phrase only appears later in the conversation or in assistant commentary, search user and assistant messages together.

```bash
phrase='7788'
find ~/.codex/sessions -type f -name '*.jsonl' | while read -r f; do
  jq -sr -r --arg phrase "$phrase" '
    (map(select(.type=="session_meta"))[0].payload) as $meta
    | [ .[]
        | select(.type=="event_msg" and .payload.type=="user_message")
        | .payload.message
      ] as $user_msgs
    | [ .[]
        | select(.type=="event_msg" and .payload.type=="agent_message")
        | .payload.message
      ] as $agent_msgs
    | (($user_msgs + $agent_msgs) | join("\n")) as $transcript
    | select(($transcript | ascii_downcase) | contains($phrase | ascii_downcase))
    | [
        ($meta.timestamp // ""),
        ($meta.id // ""),
        (($user_msgs[0] // "") | gsub("[\r\n\t]+"; " ") | .[0:180]),
        ($meta.cwd // ""),
        ("codex resume " + ($meta.id // ""))
      ]
    | @tsv
  ' "$f"
done | sort -r | head -n 20
```

## Inspect One Candidate Before Recommending It

```bash
session_id='019d2018-c9a5-7871-ac97-5021e5deea60'
src="$(find ~/.codex/sessions -type f -name "*${session_id}*.jsonl" | sort -r | head -n 1)"
jq -sr -r '
  .[]
  | select(.type=="event_msg" and (.payload.type=="user_message" or .payload.type=="agent_message"))
  | if .payload.type=="user_message" then
      "## User\n" + .payload.message + "\n"
    else
      "## Assistant\n" + .payload.message + "\n"
    end
' "$src" | sed -n '1,200p'
```

## Fallback Only If `event_msg` Misses

If a session obviously exists but `event_msg` search returns nothing, inspect `response_item.role=user` carefully and exclude AGENTS preamble.

```bash
session_id='019d2018-c9a5-7871-ac97-5021e5deea60'
src="$(find ~/.codex/sessions -type f -name "*${session_id}*.jsonl" | sort -r | head -n 1)"
jq -sr -r '
  [ .[]
    | select(.type=="response_item" and .payload.type=="message" and .payload.role=="user")
    | .payload.content[]?
    | select(.type=="input_text")
    | .text
    | select(startswith("# AGENTS.md instructions") | not)
    | select(startswith("<environment_context>") | not)
  ]
' "$src"
```

## Output Policy

After searching, return:

- the best 1-3 candidate session IDs
- one short reason each candidate matches
- the exact `codex resume <id>` command

When multiple sessions are close, prefer the one whose `cwd` matches the current repo and whose first user message is closest to the user's stated goal.
