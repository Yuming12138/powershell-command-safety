# Deployment Reference

Read this reference only for release, service, container, or remote deployment
work. For remote operations, use the explicit OpenSSH executable and also
follow the WSL and SSH reference linked directly from `SKILL.md`.

## Preflight

1. Read the current compose/service definition, release path, mounts, running
   process, and health status.
2. Back up configuration before patching it.
3. Confirm the exact target host, release name, architecture, and rollback path.
4. Build or package into a persistent path. Use temporary directories only for
   one-shot scripts.

## Files And Mounts

Preserve file-versus-directory semantics. Inspect the live mount declaration
before uploading:

- A file mount needs the exact binary or config file at the host-side source.
- A directory mount needs the directory layout expected by the container.

Do not infer the remote mount source from the local build directory. Verify the
remote source type and final path independently after upload.

## Runtime Checks

- Use non-interactive privilege escalation (`sudo -n`) so automation fails
  instead of hanging on a password prompt.
- For a Go binary in a minimal Linux container, verify architecture and static
  linkage with `file` before diagnosing `exec ... no such file or directory` as
  a missing path.
- After switching releases, verify the process/container health, local status
  endpoint, public status endpoint when applicable, and the rollback target.
- If a wrapper times out, inspect child processes, logs, and release artifacts
  before starting another copy.

## Release Integrity

Record the release identifier and hashes from the machine, not by hand. Verify:

- uploaded file type and permissions;
- local and remote SHA-256 values;
- active process command line or container image;
- health response and expected version;
- old release remains available for rollback.

Do not treat a zero exit status alone as proof that a deployment succeeded.
