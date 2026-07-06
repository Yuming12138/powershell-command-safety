---
name: powershell-command-safety
description: Use whenever Codex writes, runs, explains, or debugs PowerShell commands or scripts on Windows, including ordinary filesystem/process/service commands, environment variables, JSON/text processing, native executables, WSL/SSH/Docker interop, uploads/downloads, deployment automation, quoting, escaping, CRLF/encoding, and safe cleanup.
---

# PowerShell Command Safety

## Default Stance

Treat PowerShell as its own shell, not as bash with different syntax. Before running a command, identify:

`PowerShell -> optional native executable -> optional WSL/bash -> optional SSH remote shell -> optional container shell`

Decide which layer should expand variables, globs, quotes, redirection, and command substitution. Most failures come from the wrong layer expanding first.

## Continuous Maintenance

Keep this skill current. Whenever a PowerShell command or script fails in a reusable way, update this skill after the immediate task is stable.

Add a new rule when all are true:

- The failure was caused by PowerShell behavior, Windows-native command behavior, encoding/CRLF, quoting/escaping, environment variables, filesystem semantics, native executable interop, WSL/SSH/container boundaries, transfer reliability, or deployment scripting.
- The lesson is likely to apply again.
- The rule can be written without secrets, one-off paths, or excessive task-specific context.

Prefer a concise prevention rule plus one minimal example. Do not add long incident reports. If the new lesson is a project-specific deployment constraint, add it only when it changes how PowerShell commands should be written or executed.

After editing this skill, run:

```powershell
python C:\Users\13849\.codex\skills\.system\skill-creator\scripts\quick_validate.py C:\Users\13849\.codex\skills\powershell-command-safety
```

## Command Construction Rules

- Prefer one shell at a time. If a task is pure PowerShell, use PowerShell cmdlets end to end.
- Prefer `-LiteralPath` for file paths that may contain brackets, wildcards, `&`, spaces, or non-ASCII characters.
- Prefer arrays for native executable arguments when constructing commands in variables.
- Avoid PowerShell double-quoted strings around bash/ssh scripts containing `$var`, `$(...)`, backticks, or `\`.
- Use single quotes for literal strings; use double quotes only when PowerShell interpolation is intended.
- Never compose destructive file operations by enumerating in PowerShell and deleting in `cmd /c`, bash, or another shell.
- Avoid variable names that collide with built-in variables. PowerShell variable names are case-insensitive, so `$home` collides with read-only `$HOME`, `$matches` collides with automatic `$Matches`, and `$Args` collides with automatic `$args`. Use names such as `$ArgList`, `$Rows`, or `$ResultItems` for function parameters and local collections.
- Do not pipe directly from a `foreach (...) { ... }` statement block; it can fail with `An empty pipe element is not allowed.` Collect results in an array or use pipeline-native `ForEach-Object` when the output needs to be piped.

Safe examples:

```powershell
Get-ChildItem -LiteralPath 'D:\path with spaces'
Remove-Item -LiteralPath $target -Recurse -Force
& git -C 'D:\repo' status --short
$rows = foreach ($item in $items) { [pscustomobject]@{ Name = $item.Name } }
$rows | Format-Table -AutoSize
```

## Error Handling

For multi-step scripts, start with strict behavior:

```powershell
$ErrorActionPreference = 'Stop'
Set-StrictMode -Version Latest
```

Remember that many native commands signal failure with `$LASTEXITCODE`, not PowerShell exceptions. Check it when the next step depends on success:

```powershell
& git status --short
if ($LASTEXITCODE -ne 0) { throw "git status failed: $LASTEXITCODE" }
```

Do not rely on `$?` after several commands; it only reflects the most recent pipeline status.

## Paths And Files

- Use `Join-Path`, `Resolve-Path`, and `[System.IO.Path]` rather than manual string concatenation for Windows paths.
- `New-Item` creates paths with `-Path`, not `-LiteralPath`; use `-Path` for creation, then use `-LiteralPath` for later reads, copies, moves, or deletes.
- Before recursive delete or move, resolve the final target and verify it is inside the intended directory.
- Use `Copy-Item`/`Move-Item`/`Remove-Item` with `-LiteralPath`; avoid `del`, `rd`, and `cmd /c` unless explicitly required.
- If creating temporary files for scripts, use `$env:TEMP` or `[System.IO.Path]::GetTempPath()`.

Safe recursive cleanup pattern:

```powershell
$root = (Resolve-Path -LiteralPath 'D:\safe-root').Path
$target = (Resolve-Path -LiteralPath $candidate).Path
if (-not $target.StartsWith($root + [IO.Path]::DirectorySeparatorChar, [StringComparison]::OrdinalIgnoreCase)) {
  throw "Unsafe target: $target"
}
Remove-Item -LiteralPath $target -Recurse -Force
```

## Text, Encoding, And JSON

- For non-ASCII output from Python or external tools, set `$env:PYTHONIOENCODING='utf-8'` when needed.
- Prefer structured PowerShell JSON handling over regex:

```powershell
$obj = Get-Content -Raw -LiteralPath $path | ConvertFrom-Json
$obj.value
```

- When writing scripts that will run in WSL or Linux, strip CRLF or generate them inside WSL. CRLF symptoms include `sort\r: command not found`, broken heredoc terminators, and correct-looking paths reported as missing.

## Native Commands

PowerShell parsing differs from the target program's parsing. If arguments contain characters PowerShell treats specially, pass them as separate arguments:

```powershell
& ssh cmsg-root 'df -h /'
& curl.exe -sS -o NUL -w 'status=%{http_code}\n' 'https://example.com'
```

Use `curl.exe` when the real curl binary is required; `curl` can be an alias on older Windows PowerShell environments.

## Environment Variables

- Read/write PowerShell environment variables with `$env:NAME`.
- Do not print secrets. Show only presence, length, suffix, or hash.
- When passing environment variables into WSL or SSH, be explicit; do not assume they cross boundaries.

```powershell
$env:FOO = 'bar'
wsl -d Ubuntu-24.04 -- bash -lc 'printf "%s\n" "$FOO"' # may be empty unless WSL imports it
```

## WSL And Bash From PowerShell

For simple WSL commands, protect bash code with PowerShell single quotes:

```powershell
wsl -d Ubuntu-24.04 -- bash -lc 'cd /home/gmchen/work/new-api-cmsg && git status --short'
```

For complex WSL scripts, pipe a single-quoted here-string and strip CRLF:

```powershell
@'
set -euo pipefail
release=new-api-example
echo "$release"
'@ | wsl -d Ubuntu-24.04 -- bash -lc "tr -d '\r' > /tmp/task.sh && bash /tmp/task.sh"
```

Avoid this pattern because PowerShell expands `$release` before WSL sees it:

```powershell
wsl -d Ubuntu-24.04 -- bash -lc "release=x; echo $release"
```

## SSH And Remote Scripts

For complex remote scripts, prefer:

1. PowerShell single-quoted here-string.
2. WSL `tr -d '\r' > /tmp/remote.sh`.
3. `ssh host 'sudo -n bash -s' < /tmp/remote.sh`.

Template:

```powershell
@'
set -euo pipefail
cd /opt/new-api
sudo -n docker compose -f docker-compose.prod.yml ps
'@ | wsl -d Ubuntu-24.04 -- bash -lc "tr -d '\r' > /tmp/remote.sh && ssh cmsg-root 'sudo -n bash -s' < /tmp/remote.sh"
```

If a local bash script is itself fed on stdin, nested `ssh host ...` may consume the rest of that script. Use `ssh -n` for SSH commands that do not need stdin:

```bash
ssh -n cmsg-root "df -h /"
```

Do not use `ssh -n` when intentionally streaming a file:

```bash
ssh cmsg-root "cat > /tmp/artifact.bin" < artifact.bin
```

## Transfers

For large files over unstable SSH, prefer resumable or chunked approaches:

1. `rsync -avP --partial --append-verify -e ssh local host:/tmp/file`
2. Split into small chunks, upload with `ssh host "cat > /tmp/chunks/part-aa" < part-aa`, concatenate remotely.
3. Always verify `sha256sum` before installing.

## Deployment And Runtime Config

- Read current state first: compose file, running mounts, current release path, health status.
- Back up config before patching.
- Preserve file-vs-directory mounts exactly. For `cmsg-root:/opt/new-api`, compose mounts one built binary file to `/new-api`; do not mount a release directory there.
- For Go binaries running in Alpine containers, build static Linux binaries and verify `file` says `statically linked`.
- Use `sudo -n` in automation so commands fail instead of hanging on password prompts.
- After switching releases, verify container health plus local and public status endpoints.

## Quick Debug Checklist

When PowerShell behavior looks wrong:

- Did PowerShell expand a `$var` or `$()` that was meant for bash?
- Did a nested SSH command consume stdin from a piped script?
- Did CRLF break a Linux script?
- Did an alias like `curl` or `ls` resolve differently than expected?
- Did a native command fail but only set `$LASTEXITCODE`?
- Did a path need `-LiteralPath`?
- Are secrets being printed accidentally?
