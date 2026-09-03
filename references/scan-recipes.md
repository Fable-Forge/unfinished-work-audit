# Scan Recipes

The snippets below are portable starting points for Windows, Git Bash, and Python. Adjust the date literal and discovered roots for the current machine.

## Console encoding

Chinese thread names mojibake when printed to the Windows console. Always write UTF-8 to a scratch file and read the file back.

```python
import io
w = io.open(r'<scratchpad>\out.txt', 'w', encoding='utf-8')
w.write(line + "\n")
w.close()
```

## Codex: thread index first pass

Use the thread index as the cheapest first-pass signal before opening full rollouts.

```python
import json
from pathlib import Path

rows = []
for line in open(Path.home() / '.codex' / 'session_index.jsonl', encoding='utf-8'):
    try:
        d = json.loads(line)
        rows.append((d.get('updated_at', '')[:10], d.get('thread_name', '')))
    except Exception:
        pass
rows.sort()
recent = [r for r in rows if r[0] >= '<YYYY-MM-DD>']
```

Repeated `thread_name` values with a `(2)` / `(3)` suffix mean the user re-asked — usually a sign the earlier thread did not land.

## Codex: full rollouts

```bash
find "$HOME/.codex/sessions" -name "rollout-*.jsonl" -newermt "<YYYY-MM-DD>" | wc -l
```

Pre-digested outcomes are cheaper than raw rollouts when they exist:

```bash
ls -1 "$HOME/.codex/memories/rollout_summaries"
```

## Claude Code: per-project sessions

Directory names encode the repository path. Session mtime is the reliable date; there may be no separate index file.

```python
import json, glob, os, datetime
from pathlib import Path

for f in glob.glob(str(Path.home() / '.claude' / 'projects' / '*' / '*.jsonl')):
    mt = datetime.datetime.fromtimestamp(os.path.getmtime(f))
    if mt < datetime.datetime.fromisoformat('<YYYY-MM-DD>'):
        continue
    proj = os.path.basename(os.path.dirname(f))
    first = last = ''
    for line in open(f, encoding='utf-8'):
        try:
            d = json.loads(line)
        except Exception:
            continue
        if d.get('type') == 'user' and not first:
            c = d.get('message', {}).get('content')
            if isinstance(c, list):
                c = ' '.join(x.get('text', '') for x in c
                             if isinstance(x, dict) and x.get('type') == 'text')
            # skip system-injected turns
            if isinstance(c, str) and c.strip() and not c.startswith('<'):
                first = ' '.join(c.split())[:110]
        if d.get('type') == 'assistant':
            for c in d.get('message', {}).get('content', []) or []:
                if isinstance(c, dict) and c.get('type') == 'text' and c.get('text', '').strip():
                    last = ' '.join(c['text'].split())[:300]
```

Two gotchas: user turns whose content starts with `<` are system reminders, not the user speaking; and `message.content` is sometimes a string, sometimes a list of blocks.

## Which skills actually fired

Useful for a related question — whether an existing skill is being triggered at all.

```python
if d.get('type') == 'assistant':
    for c in d.get('message', {}).get('content', []) or []:
        if isinstance(c, dict) and c.get('type') == 'tool_use' and c.get('name') == 'Skill':
            fired.add((c.get('input') or {}).get('skill'))
```

## Open-thread keyword filter

Match against the **last** assistant message, then verify in the repo:

```
下一步  待办  未完成  尚未  接下来  后续  TODO  pending  pending_external
我现在停止  不再追加  暂停  被阻塞  失败  报错
```
