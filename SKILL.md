---
name: powershell-command-safety
description: Use when writing, running, explaining, or debugging PowerShell on Windows, especially commands that cross into native executables, WSL/Bash, SSH, containers, file transfers, JSON/text processing, or destructive filesystem operations.
---

# PowerShell Command Safety

Use PowerShell as its own language. Before running a command, identify every
parser boundary:

`PowerShell -> native executable -> WSL/Bash -> SSH remote shell -> container`

Decide which layer owns variable expansion, quoting, globbing, redirection,
command substitution, stdin, and exit status. Keep one shell responsible for
each piece of syntax.

## Core Workflow

1. Inspect the current state before changing files, processes, services, or
   remote deployments.
2. Prefer a short, single-shell command. Use a script file or literal
   here-string when nesting becomes non-trivial.
3. Use explicit paths and arguments. Verify inferred paths before invoking them.
4. Check native exit codes and verify the expected artifact after each important
   step.
5. Do not print secrets; report presence, length, suffix, or a hash instead.

For version-sensitive work, prefer PowerShell 7 when installed:

```powershell
$pwsh = (Get-Command pwsh -ErrorAction Stop).Source
& $pwsh -NoLogo -NoProfile -Command '<command>'
```

Verify `$PSVersionTable.PSVersion` if the version affects behavior. Use
Windows PowerShell 5.1 only when the task specifically requires it.

## Command Construction

- Use single-quoted strings for literals and double quotes only for intended
  PowerShell interpolation.
- PowerShell uses the backtick, not backslash, as its escape character. Do not
  write Bash-style `\"` to escape a PowerShell string.
- Use `${name}` when punctuation follows an interpolated variable, such as
  `"failed for ${name}: $code"`.
- Prefer `-LiteralPath` for reads, copies, moves, deletes, and commands that
  must not interpret wildcard characters. `New-Item` uses `-Path` for creation;
  use `-LiteralPath` for later operations.
- Distinguish a current-directory path from a drive-rooted path. Verify
  executables with `Test-Path -LiteralPath` before depending on them.
- Use `Join-Path`, `Resolve-Path`, and `[System.IO.Path]` instead of manual path
  concatenation.
- Prefer arrays for native argv, but inspect the wrapper: a wrapper that joins
  argv and invokes `shell: true` destroys argument boundaries. Prefer
  `spawn`/`execFile`-style wrappers or quote target-shell arguments explicitly.
- Do not use variable names that collide with automatic variables such as
  `$HOME`, `$Matches`, `$Args`, or `$PID`.
- Force array shape before concatenating values that may be scalar or arrays:
  `@($items) + @($moreItems)`.
- Do not pipe directly from a `foreach` statement block. Collect first, or use
  pipeline-native `ForEach-Object`:

```powershell
$rows = foreach ($item in $items) {
    [pscustomobject]@{ Name = $item.Name }
}
$rows | Format-Table -AutoSize
```

- Do not put a statement-form `if/else` inside ordinary parentheses as though
  it were an expression. Assign the result first:
  `$expected = if ($ok) { $a } else { $b }`.
- Do not separate command invocations with commas inside `@(...)`; commas are
  expressions, not command separators.
- Do not place a PowerShell here-string inside another here-string with the
  same delimiter. Build the inner text from an array joined with
  `[Environment]::NewLine`, or use a different representation.

## Errors And Exit Status

For multi-step scripts, start with:

```powershell
$ErrorActionPreference = 'Stop'
Set-StrictMode -Version Latest
```

Native commands usually signal failure with `$LASTEXITCODE`, not an exception:

```powershell
& git status --short
if ($LASTEXITCODE -ne 0) { throw "git status failed: $LASTEXITCODE" }
```

Do not rely on `$?` after several commands. If a wrapper times out, inspect
the child process, logs, and output artifact before retrying; the child may
still be running.

When output may be empty, normalize it before reading properties:

```powershell
$measure = @(Get-ChildItem -LiteralPath $path -File | Measure-Object Length -Sum)
$sum = if ($measure.Count -and $measure[0].PSObject.Properties['Sum']) {
    $measure[0].Sum
} else {
    0
}
```

## Paths, Files, And Cleanup

- Discover filenames with `rg --files` or verify them with
  `Test-Path -LiteralPath`; do not combine guessed and verified paths in one
  diagnostic command.
- On Windows, pass a real directory to `rg` and filter with `--glob`; do not
  pass an unexpanded wildcard as an explicit input path.
- Before recursive deletion or movement, resolve the final target and verify
  it is below the intended root:

```powershell
$root = (Resolve-Path -LiteralPath 'C:\safe-root').Path
$target = (Resolve-Path -LiteralPath $candidate).Path
$prefix = $root.TrimEnd('\') + [IO.Path]::DirectorySeparatorChar
if (-not $target.StartsWith($prefix, [StringComparison]::OrdinalIgnoreCase)) {
    throw "Unsafe target: $target"
}
$targetItem = Get-Item -LiteralPath $target -Force
if ($targetItem.Attributes -band [IO.FileAttributes]::ReparsePoint) {
    throw "Refusing reparse-point target without an explicit deletion policy: $target"
}
Remove-Item -LiteralPath $target -Recurse -Force
```

- Never enumerate paths in PowerShell and then delete them in `cmd`, Bash, or
  another shell.
- Use `$env:TEMP` or `[IO.Path]::GetTempPath()` for temporary files. Give
  concurrent jobs unique names; never have parallel jobs share `/tmp/task.sh`.
- Do not use `ReadAllBytes()` merely to inspect a large file. Use a stream for
  a prefix and `Get-FileHash` for a full-file digest.
- A deep Windows staging path can cause `WinError 3` because of path length.
  Verify an extended-length path (`\\?\D:\...`) or use a genuinely shorter
  checkout before assuming the source is missing.

## Text, Encoding, And JSON

- Prefer structured parsing over regex:

```powershell
$obj = Get-Content -Raw -LiteralPath $path | ConvertFrom-Json
```

- For JSON with empty-string property names, use `ConvertFrom-Json -AsHashTable`.
- Under strict mode, check optional JSON properties before reading them.
- Native output can be a PowerShell array. Join it before applying a
  whole-document regex:
  `$text = @(& tool.exe) -join "`n"`.
- Do not hand-build JSON with interpolated Bash or PowerShell values. Pass
  values as argv to a JSON encoder such as Python, or construct a PowerShell
  object and use `ConvertTo-Json`.
- If a CLI emits advisory text with JSON, use its JSON-only mode. Otherwise
  capture the documented JSON record, check `$LASTEXITCODE`, and parse only
  that record; do not blindly parse the whole stdout stream.
- If a log repeats a metric, use `[regex]::Matches()` and select the intended
  occurrence instead of taking the first match.
- Set `$env:PYTHONIOENCODING = 'utf-8'` when a Python/native tool's non-ASCII
  output needs deterministic decoding.
- Do not pipe data into a command that also uses a heredoc for its program
  text, for example `curl ... | python3 - <<'PY'`; the heredoc consumes stdin.
  Use `python3 -c` or a temporary data file.

## Native, WSL, And Remote Boundaries

- Pass native arguments as separate argv elements:

```powershell
& curl.exe -sS -o NUL -w 'status=%{http_code}`n' 'https://example.com'
```

- Do not rely on aliases such as `curl` or `ls` when the target must be a
  native program; call `curl.exe`, `Get-ChildItem`, or the intended command
  explicitly.
- For simple WSL commands, use a PowerShell single-quoted Bash command. Once
  the script needs variables, substitutions, redirects, nested SSH, or many
  steps, use a literal here-string and normalize CRLF before Bash runs it:

```powershell
@'
set -euo pipefail
release=example
printf '%s\n' "$release"
'@ | wsl.exe -- bash -lc "tr -d '\r' > /tmp/check.sh && bash /tmp/check.sh"
```

- Do not put Bash variables inside a PowerShell double-quoted `bash -lc`
  command; PowerShell expands them first.
- Do not use Bash redirection syntax such as `<<< $value` in PowerShell.
- If the task involves SSH, server access, uploads, downloads, tunnels, or
  remote commands, discover the installed client first and invoke the intended
  executable explicitly (`ssh.exe`, `scp.exe`, or `sftp.exe` on Windows). Use
  configured host aliases, preserve normal host-key verification, check every
  native exit code, and verify the remote result independently. Read
  [references/wsl-ssh.md](references/wsl-ssh.md) for shell-boundary patterns
  and transfer verification.

## Validation

After editing this skill, run:

```powershell
python <codex-skill-creator-root>\scripts\quick_validate.py <skill-path>
```

For any command that changes state, verify the expected file, process, service,
remote artifact, checksum, or health endpoint. A zero exit code without the
expected result is not success.

For deployment-specific release checks, read
[references/deployment.md](references/deployment.md) only when the task is
actually a deployment.
