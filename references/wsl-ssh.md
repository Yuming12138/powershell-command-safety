# WSL And SSH Reference

Use this reference when a PowerShell command crosses into WSL/Bash, SSH, a
container, or a file-transfer boundary. For actual server operations, use the
installed `ssh-skill` and its Python helpers; the commands below describe shell
ownership and validation patterns, not an exception to that rule.

## WSL Scripts

Use a literal PowerShell here-string for non-trivial Bash. Normalize line
endings at the Linux boundary and give concurrent scripts unique paths:

```powershell
$scriptPath = "/tmp/check-$([guid]::NewGuid().ToString('N')).sh"
@'
set -euo pipefail
repo=/home/user/project
cd "$repo"
git status --short
'@ | wsl.exe -- bash -lc "tr -d '\r' > '$scriptPath' && bash '$scriptPath'; rc=`$?; rm -f '$scriptPath'; exit `$rc"
```

Keep Bash quotes normal inside the single-quoted PowerShell here-string. Do not
add `\"` or nested `\'` escapes unless Bash itself needs a literal backslash.

For a script with a Bash shebang or `[[ ... ]]`, validate with `bash -n`, not
`sh -n`. Do not assume WSL `/tmp` survives a later PowerShell invocation; use a
persistent Linux path for artifacts that must outlive one command.

When Bash values must become JSON, do not interpolate them into JSON text.
Pass them as argv to an encoder that performs escaping:

```bash
python3 -c 'import json,sys; print(json.dumps({"path": sys.argv[1]}))' "$path"
```

## SSH Script Boundaries

An independent SSH command starts in the remote account's home directory. Use
an absolute path or `cd` to a verified root in every standalone probe.

If a script is streamed to a remote shell, avoid nested commands that consume
the same stdin. Use `ssh -n` only for commands that do not need stdin. Never
write both `< remote.sh` and `< /dev/null` on the same command: the latter
overrides the former. If the remote script needs stdin disabled for a nested
command, apply `</dev/null` to that nested command instead.

```bash
# Illustrative shell shape only; use `ssh-skill` helpers for actual operations.
# The outer script receives remote.sh.
ssh host 'bash -s' < remote.sh

# A nested command, not the outer shell, is given empty stdin.
docker compose exec -T service command </dev/null
```

Do not mix a remote heredoc with a local piped data stream. For a remote script
plus data, pass data as a safely encoded argument or create a verified temporary
file. PowerShell does not support Bash `<<<` here-strings.

For quoted heredocs, Python snippets, regexes, or nested SSH, create the script
as literal text and pass values as `bash -s --` arguments. Avoid deeply nested
`ssh "sudo bash -lc '...'"` strings.

## Encoding And Transfer Checks

- Normalize CRLF before a Linux shell parses the script. A trailing `\r` can
  break heredoc terminators and produce misleading missing-file errors.
- Do not pipe a PowerShell string directly to a native SSH decoder when exact
  bytes matter; native pipelines can serialize records with CRLF. Use the
  `ssh-skill` upload helper or a verified temporary file.
- In PowerShell, resolve `ssh-skill` helpers from `$env:USERPROFILE` or another
  verified absolute local path. Do not pass a quoted `~/.codex/...` path to a
  native executable and assume PowerShell will expand it. Windows local paths
  may use backslashes; remote Linux paths should use forward slashes.
- Prefer resumable transfer for large files. Verify the destination with a
  machine-derived SHA-256, never a hand-transcribed value:

```powershell
$expected = (Get-FileHash -LiteralPath $localPath -Algorithm SHA256).Hash.ToLowerInvariant()
# Ask the remote helper for sha256sum and compare its parsed hash to $expected.
```

- A checksum file should contain the target filename, not the source machine's
  absolute path.
- A successful transport command is insufficient: verify remote type, size,
  checksum, permissions, and the artifact's intended location. Establish the
  expected owner/mode before declaring success.
- If a resumed upload has the wrong size or hash, do not keep appending or
  overwrite the target implicitly. Preserve the evidence, obtain authorization
  to replace it, upload to a fresh temporary destination, verify it, and only
  then switch it into place.

## Layer Checklist

Before execution, answer:

1. Which shell expands each `$var`, `$(...)`, glob, and regex?
2. Which process owns stdin at each nesting level?
3. Which line ending reaches Linux?
4. Which exit code is authoritative?
5. What exact artifact proves success?
