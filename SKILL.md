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

When updating this skill from GitHub, first verify whether the active skill directory is itself a Git checkout. If it is a plain installed copy, do not run `git pull` in that directory. Pull or clone the remote into a separate worktree, merge the rule changes there, validate, then sync the updated files back to the active skill path.

## Command Construction Rules

- Prefer one shell at a time. If a task is pure PowerShell, use PowerShell cmdlets end to end.
- Prefer `-LiteralPath` for file paths that may contain brackets, wildcards, `&`, spaces, or non-ASCII characters.
- Prefer arrays for native executable arguments when constructing commands in variables.
- Windows npm/npx shims and Node CLIs that spawn package-manager commands can misparse working directories containing shell metacharacters such as `&`. If a project must remain in such a path, run the tool through a persistent junction or other alias whose path contains no shell metacharacters; keep the source files at their original location.
- `wsl.exe` command forwarding can also lose quoting around Linux paths derived from Windows directories containing `&`. For syntax checks or other stdin-capable commands, stream normalized file contents to WSL (for example, `Get-Content -Raw ... | wsl -- sh -n -`) instead of passing the metacharacter-containing path. For commands that require a path, copy to a verified short temporary path or run entirely within a safely quoted WSL script.
- When combining values that may be a scalar or an array, force array shape on both sides before using `+`. PowerShell treats scalar strings as strings, so `$domains + 'api.example.com'` can become one concatenated hostname instead of two items; use `@($domains) + @('api.example.com')` or assign the full explicit array.
- Avoid PowerShell double-quoted strings around bash/ssh scripts containing `$var`, `$(...)`, backticks, or `\`.
- Use single quotes for literal strings; use double quotes only when PowerShell interpolation is intended.
- In an interpolated string, delimit a variable with `${name}` when punctuation such as `:` immediately follows it. PowerShell can parse `$name:` as an invalid scoped-variable reference; use `"failed for ${name}: $code"`.
- Do not nest a PowerShell here-string inside another here-string that uses the same quote style and terminator. The inner terminator closes the outer string during parsing; build the inner content from a string array joined with `[Environment]::NewLine`, or use a safely different representation.
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

- JSON files such as `package-lock.json` commonly contain an empty-string property key. Parse these with `ConvertFrom-Json -AsHashTable`; normal object conversion rejects empty property names.

- Under `Set-StrictMode -Version Latest`, reading a missing property from `ConvertFrom-Json` output throws instead of returning `$null`. For optional API fields, check `$obj.PSObject.Properties['name']` first or use a helper that returns `$null` for absent fields.
- Do not pipe data into a command that also uses a heredoc for its program text. For example, `curl ... | python3 - <<'PY'` sends the heredoc script to Python stdin, so the piped JSON is lost. Use `python3 -c '...'` to read piped stdin, or write the data to a temporary file and pass the file path as an argument.
- When writing scripts that will run in WSL or Linux, strip CRLF or generate them inside WSL. CRLF symptoms include `sort\r: command not found`, broken heredoc terminators, and correct-looking paths reported as missing.
- When patching files that live only inside WSL, do not pass Linux paths like `/home/...` to a Windows-side patcher; it may resolve them as `D:\home\...` or another invalid Windows path. Either use a Windows UNC path such as `\\wsl.localhost\Ubuntu-24.04\home\...` from Windows, or run the patch tool entirely inside WSL. Preserve the file's existing line endings when using ad hoc scripts so a one-line edit does not become a whole-file diff.
- Windows-side tools can hang or time out on `\\wsl.localhost\...` paths. If a patch or file operation against a WSL UNC path stalls, stop retrying it from Windows; create/apply the patch inside WSL instead, for example `git apply` from the repository root. Do not assume helper commands such as `apply_patch` exist inside WSL unless you have verified them.
- When a WSL repo intentionally stores a file with CRLF line endings, default `git diff --check` can report every added CRLF line as trailing whitespace. First confirm the file's stored/working line endings, then run the check with `git -c core.whitespace=blank-at-eol,blank-at-eof,space-before-tab,cr-at-eol diff --check` instead of normalizing the whole file just to satisfy the check.

## Native Commands

PowerShell parsing differs from the target program's parsing. If arguments contain characters PowerShell treats specially, pass them as separate arguments:

```powershell
& ssh cmsg-root 'df -h /'
& curl.exe -sS -o NUL -w 'status=%{http_code}\n' 'https://example.com'
```

Use `curl.exe` when the real curl binary is required; `curl` can be an alias on older Windows PowerShell environments.

`rg` returns exit code `1` when no matches are found. If no matches are an acceptable result, handle codes greater than `1` as errors and explicitly finish the PowerShell command successfully so the stale native exit code does not mark the whole step as failed.

```powershell
& rg -n 'pattern' $path
if ($LASTEXITCODE -gt 1) { throw "rg failed: $LASTEXITCODE" }
exit 0
```

When stopping processes selected by `Win32_Process.CommandLine`, never rely on command-line text alone: the current PowerShell host can contain the same search text in its own invocation. Restrict the executable name to the expected process types and explicitly exclude `$PID` before calling `Stop-Process`.

An enumerated process can exit before `Stop-Process` runs. When disappearance is an acceptable outcome, stop each verified target with `-ErrorAction SilentlyContinue` so this race does not abort the remaining cleanup or restart sequence.

```powershell
$targets = Get-CimInstance Win32_Process | Where-Object {
    $_.ProcessId -ne $PID -and $_.Name -in @('python.exe', 'uvicorn.exe') -and
    $_.CommandLine -and $_.CommandLine -match 'uvicorn.*8000'
}
$targets | ForEach-Object { Stop-Process -Id $_.ProcessId -Force -ErrorAction SilentlyContinue }
```

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

Do not keep growing an inline `bash -lc '...'` command once it needs variable assignment, command substitution such as `$(date ...)`, nested SSH, redirects, or several semicolon-separated steps. Put the bash code in a PowerShell single-quoted here-string, normalize CRLF, and run that script inside WSL. This avoids cases where correct-looking inline commands produce empty variables or break quoting at the PowerShell/WSL boundary.

Do not embed nested shell quote escapes such as `'\''` inside a PowerShell command string for non-trivial one-liners, especially when the inner command contains Python/JavaScript code, JSON string literals, or operators like `+`. A single mismatched quote can make PowerShell parse the inner code itself, causing errors such as `You must provide a value expression following the '+' operator`. Use a literal here-string and run it through WSL instead:

```powershell
@'
set -euo pipefail
curl -fsS https://example.com/status |
  python3 -c 'import json,sys; data=json.load(sys.stdin); print("version=" + str(data.get("version")))'
'@ | wsl -d Ubuntu-24.04 -- bash -lc "tr -d '\r' > /tmp/check.sh && bash /tmp/check.sh"
```

For complex WSL scripts, pipe a single-quoted here-string and strip CRLF:

```powershell
@'
set -euo pipefail
release=new-api-example
echo "$release"
'@ | wsl -d Ubuntu-24.04 -- bash -lc "tr -d '\r' > /tmp/task.sh && bash /tmp/task.sh"
```

Inside a PowerShell single-quoted here-string, write bash quotes normally. Do not change bash `"` to `\"` unless the target script really needs a literal backslash. PowerShell will not interpolate a single-quoted here-string, so extra backslashes reach bash and can turn arguments such as `"$repo"` into a literal or empty-looking `""` path.

Avoid this pattern because PowerShell expands `$release` before WSL sees it:

```powershell
wsl -d Ubuntu-24.04 -- bash -lc "release=x; echo $release"
```

When a generated bash script embeds an `ssh host "..."` command, remember the local bash still expands `$1`, `$NF`, `$(...)`, and similar tokens inside that double-quoted remote command before SSH runs it. Prefer a streamed remote script, single-quoted remote command, or commands that avoid `$` expansion such as `cut` when extracting fields.

When a PowerShell-generated shell script exits with odd messages like `exit: 0\r: numeric argument required`, treat it as CRLF contamination. Normalize the script before execution (`tr -d '\r' > /tmp/task.sh`) or replace CRLF in PowerShell before piping:

```powershell
$script = $script -replace "`r`n", "`n"
$script | wsl -d Ubuntu-24.04 -- bash -lc 'cat > /tmp/task.sh && bash /tmp/task.sh'
```

## SSH And Remote Scripts

Do not pipe a Base64 string produced by PowerShell directly into `ssh host 'base64 -d'`. The native pipeline can transcode or decorate text before SSH receives it, causing remote `base64: invalid input`. For small payloads, pass the Base64 text as a quoted SSH command argument and decode from remote `printf`; for larger files, use `rsync` or `scp` with checksum verification instead.

```powershell
$encoded = [Convert]::ToBase64String([Text.Encoding]::UTF8.GetBytes($text))
& ssh cmsg-root "printf '%s' '$encoded' | base64 -d > /tmp/payload.txt"
```

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

If a remote command contains shell metacharacters such as `|`, `(`, `)`, `*`, `$`, `>`, or regex alternation like `grep -E "^(foo|bar):"`, do not embed it as a deeply nested inline `ssh host "..."` string from PowerShell. PowerShell may parse or split parts of the command before SSH receives them. Put the command in a literal here-string and stream it to `ssh host 'bash -s'` or `ssh host 'sudo -n bash -s'`:

```powershell
@'
set -euo pipefail
grep -nE '^(proxy-url|utls-pool-size):' /app/config.yaml || true
'@ | ssh cmsg-root 'sudo -n bash -s'
```

If a local bash script is itself fed on stdin, nested `ssh host ...` may consume the rest of that script. Use `ssh -n` for SSH commands that do not need stdin:

```bash
ssh -n cmsg-root "df -h /"
```

The same stdin rule applies one layer deeper. If the remote bash script is itself being streamed over SSH stdin, nested commands that also read stdin, such as `docker exec -i ...`, `docker compose exec -T ...`, `psql <<SQL`, `cat <<EOF`, or `mysql < file`, can consume the rest of the remote script and leave the outer heredoc unterminated. Prefer one of these patterns:

```bash
ssh host 'bash -s' < remote.sh              # remote.sh should not contain nested stdin consumers
ssh host 'bash -s' < remote.sh </dev/null   # only when the remote script does not need stdin
ssh host "docker exec container sh -lc 'psql -c \"SELECT 1\"'"  # no nested -i/stdin
ssh host 'bash -s' < remote.sh  # for nested docker compose exec, add </dev/null to each exec
```

For multi-line SQL or scripts inside a streamed remote script, write the inner content to a temporary file on the remote side first, then pass that file to the nested command.

If a remote script itself contains quoted heredocs or Python snippets, avoid embedding it inside a double-quoted `ssh "sudo bash -lc '...'"` command. Create the remote script as literal text in WSL and pass values as `bash -s --` arguments so neither PowerShell nor the intermediate shell strips quotes:

```powershell
@'
set -euo pipefail
cat > /tmp/remote.sh <<'REMOTE'
set -euo pipefail
python3 - "$1" <<'PY'
import sys
print(sys.argv[1])
PY
REMOTE
ssh host 'sudo -n bash -s -- arg-value' < /tmp/remote.sh
'@ | wsl -d Ubuntu-24.04 -- bash -lc "tr -d '\r' > /tmp/task.sh && bash /tmp/task.sh"
```

Do not use `ssh -n` when intentionally streaming stdin to the remote side, including heredoc-fed scripts and file uploads:

```bash
ssh cmsg-root "cat > /tmp/artifact.bin" < artifact.bin
ssh cmsg-root 'sudo -n bash -s -- arg' < /tmp/remote.sh
```

## Transfers

For large files over unstable SSH, prefer resumable or chunked approaches:

1. `rsync -avP --partial --append-verify -e ssh local host:/tmp/file`
2. Split into small chunks, upload with `ssh host "cat > /tmp/chunks/part-aa" < part-aa`, concatenate remotely.
3. Always verify `sha256sum` before installing.

When generating checksum files for upload, do not keep the source machine's absolute path in the `.sha256` file. Either write the checksum with the target filename only, or pass the expected hash as a separate argument and compare it to `sha256sum <remote-file>`. A checksum line such as `/home/me/cache/release/bin` will fail on the remote host even when the upload succeeded.

## Deployment And Runtime Config

- Read current state first: compose file, running mounts, current release path, health status.
- Back up config before patching.
- Preserve file-vs-directory mounts exactly. For `cmsg-root:/opt/new-api`, compose mounts one built binary file to `/new-api`; do not mount a release directory there.
- Do not infer a container mount source from the local artifact layout. Before uploading or switching releases, read the remote compose line and verify the host-side source path type matches it. If compose mounts `./releases/name/new-api:/new-api:ro`, upload the built binary as that exact `new-api` file, not as a directory containing it.
- Do not rely on WSL `/tmp` to persist deployment artifacts, checksums, or release-name marker files across separate PowerShell/WSL commands. WSL may stop between commands and clear tmpfs-backed `/tmp`; use a persistent Linux path such as `$HOME/.cache/<project>/releases/<release>` for build outputs, while keeping `/tmp` only for one-shot scripts within a single command.
- For Go binaries running in Alpine containers, build static Linux binaries and verify `file` says `statically linked`. If a container says an existing binary has `exec ... no such file or directory`, suspect a missing dynamic loader or wrong architecture before looking for a missing path.
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
