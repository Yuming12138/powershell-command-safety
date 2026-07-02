# PowerShell Command Safety

A Codex skill for safer PowerShell command writing and debugging on Windows.

It covers ordinary PowerShell usage plus higher-risk interop with native executables, WSL, SSH, Docker, uploads/downloads, deployment scripts, quoting, escaping, CRLF/encoding, and safe cleanup.

## Install

Copy this repository into your Codex skills directory:

```powershell
git clone https://github.com/Yuming12138/powershell-command-safety.git "$env:USERPROFILE\.codex\skills\powershell-command-safety"
```

Then start a new Codex session so the skill can be discovered.

## Files

- `SKILL.md`: main skill instructions
- `agents/openai.yaml`: UI metadata

## Maintenance

Keep this skill current. When a reusable PowerShell pitfall is found, add the shortest prevention rule and re-run the Codex skill validator.
