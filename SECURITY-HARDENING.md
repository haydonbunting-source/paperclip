# Security hardening notes

This fork is hardened for a founder-operated/local-agent deployment where unattended agents may have shell, repository, and model-provider access.

## Audit focus

Reviewed backdoor-like and high-risk surfaces:

- outbound telemetry and analytics;
- unattended permission-bypass flags for local coding agents;
- host environment and model-provider secret propagation into child agent processes;
- plugin loading and plugin worker environment passthrough;
- secrets and token literals in source, tests, and docs;
- shell execution / remote runtime paths.

## Hardened defaults in this fork

### Telemetry is opt-in

Paperclip telemetry is disabled unless explicitly enabled by either:

- `PAPERCLIP_TELEMETRY_ENABLED=1`, or
- `telemetry.enabled: true` in Paperclip config.

Even when explicitly enabled, these still disable telemetry:

- `PAPERCLIP_TELEMETRY_DISABLED=1`
- `DO_NOT_TRACK=1`
- CI environments

### Unattended permission bypass is opt-in

Agent adapters no longer default to bypassing interactive permissions:

- Claude local: `dangerouslySkipPermissions` defaults to `false`.
- OpenCode local: `dangerouslySkipPermissions` defaults to `false`.
- Cursor local: `--yolo` is added only when `dangerouslySkipPermissions: true` is explicitly configured.
- Hermes local: `--yolo` is added only when `dangerouslySkipPermissions: true` is explicitly configured.
- Codex local: `--dangerously-bypass-approvals-and-sandbox` is added only when explicitly configured.
- Gemini local: `--approval-mode yolo` and `--sandbox=none` are added only when `dangerouslySkipPermissions: true` is explicitly configured; sandbox mode defaults on.
- Grok local: `--permission-mode dontAsk` / `--always-approve` are not added by default.
- New-agent UI defaults set `dangerouslySkipPermissions: false`.

### Child agent processes receive an allowlisted host environment

Paperclip adapters no longer pass the entire Paperclip server process environment into spawned agent runs. The shared child-process runner starts from a small allowlist of non-secret runtime variables and then applies explicit per-agent config/env plus Paperclip-scoped runtime variables.

This prevents accidental propagation of host secrets such as model API keys, shell tokens, cloud credentials, and unrelated environment secrets.

The allowlist intentionally keeps runtime-discovery variables such as `PATH`, `HOME`, and XDG directory hints so authenticated CLIs can still locate their installed binaries and local login state. Treat host-level CLI login directories as sensitive: use Paperclip-managed homes/secrets for stronger per-agent isolation.

## Remaining high-risk surfaces to keep gated

- Local plugin installation and activation can execute plugin workers. Only install trusted plugins.
- Sandbox/remote runtime cleanup uses destructive shell commands against managed remote paths. Keep those operations restricted to Paperclip-managed directories.
- Model-provider credentials should be stored through explicit Paperclip secret/provider configuration rather than inherited from the host environment.

## Verification checklist

Run before trusting a new build:

```sh
pnpm install --frozen-lockfile
pnpm run typecheck
pnpm test:run
pnpm run check:tokens
pnpm audit --audit-level moderate
```
