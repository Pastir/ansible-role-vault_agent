# pastir.vault_agent

Installs Vault and runs **Vault Agent** with AppRole auto-auth and templates. A typical use: render an application
`.env` from secrets and restart the workers afterwards.

Two install methods:

- `archive` (default): the official zip from releases.hashicorp.com. The checksum is verified against a table in the
  role (signed `SHA256SUMS`, HashiCorp key `C874011F0AB405110D02105534365D9472D7468F`). For unknown versions the role
  falls back to the `SHA256SUMS` file next to the archive. The download happens only when the installed binary
  reports another version.
- `apt`: the `vault` package from the HashiCorp APT repository, pinned and held. It ships its own `vault.service` (a
  Vault **server** unit), so the agent unit needs another name.

A version change (up or down) needs `-e vault_agent_allow_upgrade=true`. Without the flag the role stops with a message.

## Requirements

- ansible-core >= 2.16, collection `community.general` (sudoers rule) (`requirements.yml` in the role root)

## Variables

| Variable | Default | Description |
|---|---|---|
| `vault_agent_version` | `2.0.3` | |
| `vault_agent_allow_upgrade` | `false` | Allows changing the installed version |
| `vault_agent_install_method` | `archive` | `archive` or `apt` |
| `vault_agent_archive_base_url` | `https://releases.hashicorp.com/vault` | Change for a mirror with the same layout |
| `vault_agent_archive_checksum` | `""` | sha256 of the zip. Empty: built-in table, then `SHA256SUMS` |
| `vault_agent_binary` | `/usr/local/bin/vault` | archive method. The apt package uses `/usr/bin/vault` |
| `vault_agent_hold` / `vault_agent_repo_enabled` | `true` / `true` | apt method |
| `vault_agent_service` | `vault-agent` | systemd unit name |
| `vault_agent_unit_path` | `/etc/systemd/system/<service>.service` | Set it to keep an existing unit in place, e.g. `/usr/lib/systemd/system/vault.service` |
| `vault_agent_log_level` | `info` | `VAULT_LOG_LEVEL` |
| `vault_agent_capability_bounding_set` / `vault_agent_ambient_capabilities` | `[]` | Capabilities of the service. Empty: systemd defaults |
| `vault_agent_no_new_privileges` | `true` without sudo commands | Must be `false` for `sudo` in template commands |
| `vault_agent_user` / `vault_agent_group` | `vault` | |
| `vault_agent_user_create` | `true` for archive | System user without home and shell. The apt package creates its own |
| `vault_agent_user_groups` | `[]` | Extra groups of the agent user, e.g. the application group |
| `vault_agent_group_members` | `[]` | Existing users added to the agent group. Missing users fail the run |
| `vault_agent_config_dir` | `/etc/vault.d/vault-agent` | `agent-config.hcl`, `role-id`, `secret-id`. Env file: `/etc/vault.d/<service>.env` |
| `vault_agent_address` | — | **Required.** Vault URL |
| `vault_agent_ca_cert` | `""` | CA bundle for the Vault server |
| `vault_agent_tls_skip_verify` | `false` | Do not enable in production. The role warns if it is enabled |
| `vault_agent_role_id` / `vault_agent_secret_id` | — | **Required.** Written with `no_log`, mode `0600` |
| `vault_agent_remove_secret_id_after_reading` | `false` | |
| `vault_agent_env` | `{}` | Environment of the service (available as `env "NAME"` in templates) |
| `vault_agent_templates` | `[]` | `[{source, destination, perms, command, error_on_missing_key}]` |
| `vault_agent_sudo_commands` | `[]` | Passwordless sudo for the agent user, for template commands. Escape `:` `,` `=` in arguments as `\:` |

## Restarts

A change of the binary, configuration, credentials, environment or unit restarts the agent. After a restart the agent
renders the templates again, but runs a template `command` only if the rendered file changed. The test checks this:
a restart does not restart the workers.

## Taking over a hand-installed agent

Set the variables to the existing layout, so that only file formatting changes:

```yaml
vault_agent_service: vault
vault_agent_unit_path: /usr/lib/systemd/system/vault.service
vault_agent_binary: /usr/bin/vault
vault_agent_user_groups: [app]
vault_agent_group_members: [app]
```

The role writes credentials with a trailing newline (as `echo` does), so existing `role-id`/`secret-id` stay unchanged.
Directory and file modes become `0750`/`0640`, the unit `0644`.

## Example

```yaml
vault_agent_address: https://vault.example.com:8200
vault_agent_role_id: "{{ vault_app_role_id }}"
vault_agent_secret_id: "{{ vault_app_secret_id }}"
vault_agent_env: {V_ENV: stage}
vault_agent_templates:
  - source: /var/www/app/env.ctmpl
    destination: /var/www/app/.env
    perms: "0640"
    command: >-
      sudo /usr/bin/chown app:app /var/www/app/.env &&
      sudo /usr/bin/supervisorctl restart all
vault_agent_sudo_commands:
  - /usr/bin/chown app\:app /var/www/app/.env
  - /usr/bin/supervisorctl restart all
```

## Testing

`molecule test` runs a Vault dev server with AppRole and a KV v2 secret, plus three agents:

- `agent-legacy-noble`: installed by a shell script first (zip in `/usr/bin`, unit in `/usr/lib/systemd/system`,
  capabilities, sudo stop/start of a workers service), then taken over by the role;
- `agent-apt-noble`: apt method;
- `agent-resolute`: archive defaults, upgrade from 2.0.2.

The test checks the rendered secret, the version, private credentials, the hold, group memberships and sudo rules. It
also restarts the taken-over agent and checks that the workers were not restarted.

## Installation

```yaml
# requirements.yml
roles:
  - name: pastir.vault_agent
    src: git+git@github.com:Pastir/ansible-role-vault_agent.git
    version: main  # or a tag
# A role cannot install collections by itself: the project lists them (see requirements.yml in the role root).
collections:
  - name: community.general
```

```bash
ansible-galaxy install -r requirements.yml   # roles and collections
```

## License

MIT
