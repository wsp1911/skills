---
name: interactive-debug
description: Enter evidence-first interactive debug mode. Use only when the user asks for.
---

Interactive debug mode activated. Follow the workflow below:

1. Generate multiple hypotheses based on the problem described by the user

2. Insert log statements for each hypothesis

3. Ask the user to reproduce the issue and provide reproduction steps

4. Read the log file to locate the problem

5. If the problem is solved, clean up all inserted log statements; otherwise, add more logs or generate new hypotheses

IMPORTANT: Do not implement a fix until it is supported by runtime evidence.
IMPORTANT: Do not clean up logs until the user confirms the issue has been fixed.

## How to Write to Log Files

1. All logs must be written to `{project_root}/debug-agent.log`. Note: absolute paths must be used.

2. Inserted log statements should be wrapped with appropriate comment statements for location and cleanup

3. Log content should include: hypothesis ID (A/B/C/...), log location (e.g., api-get_weather), and specific information

4. Backend: write via file IO
```rust
// #region agent log
{
    use std::io::Write;
    if let Ok(mut f) = std::fs::OpenOptions::new().create(true).append(true).open(r"absolute\path\to\debug.log") {
        let _ = writeln!(f, "log_content");
    }
}
// #endregion
```

5. Frontend: start the bundled receiver script, then write via `fetch`.

Start the receiver from the project root when possible:

```bash
node <this-skill-directory>/scripts/frontend-debug-log-server.mjs
```

If the current working directory is not the project root, pass it explicitly:

```bash
node <this-skill-directory>/scripts/frontend-debug-log-server.mjs --root /absolute/path/to/project
```

Useful options:

```bash
node <this-skill-directory>/scripts/frontend-debug-log-server.mjs --port 7469 --log-file /absolute/path/to/debug-agent.log
node <this-skill-directory>/scripts/frontend-debug-log-server.mjs --no-clear
```

Inserted frontend logs should use this pattern:

```ts
// #region agent log
void fetch('http://127.0.0.1:7469/log', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    id: 'A',
    loc: 'ComponentName.eventName',
    msg: 'specific observation',
    data: { value },
  }),
}).catch(() => {});
// #endregion
```

The receiver exposes:

- `POST /log` to append JSONL entries to `debug-agent.log`
- `POST /clear` to clear the log file
- `GET /health` to confirm the server and log path

Keep frontend instrumentation project-agnostic: avoid importing app-specific debug utilities unless the project already has a logging adapter that can POST to this receiver cleanly.
