# Changelog

## 2.0.0 — 2026-11

- New default `vault_agent_install_method: archive`: official zip with a pinned sha256, installed to
  `vault_agent_binary`, downloaded only when the version differs. The apt method stays available.
- **BREAKING:** `vault_agent_hold` and `vault_agent_repo_enabled` apply to the apt method only. With the archive method
  the role creates the `vault` system user.
- New: `vault_agent_unit_path`, `vault_agent_log_level`, `vault_agent_capability_bounding_set`,
  `vault_agent_ambient_capabilities`, `vault_agent_no_new_privileges`, `vault_agent_user_groups`,
  `vault_agent_group_members`.
- Credentials are written with a trailing newline, as `echo` writes them.
- The unit gets start limits, `TimeoutStopSec`, `LimitNOFILE` and `LimitMEMLOCK`.

## 1.0.0 — 2026-11

First version: pinned Vault, AppRole auto-auth, templates with commands, sudo rules, molecule test against a dev server.
