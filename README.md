# Banquise MariaDB agent - mariadb-plugin-banquise-agent

`banquise_agent` is the pull-based MariaDB component of Banquise. It maintains
an `INFORMATION_SCHEMA.BANQUISE_AGENT` catalog view, registers once with the
control plane, heartbeats with MariaDB/OS/architecture and observed plugin
state, and executes typed signed-catalog tasks queued by an administrator.

The agent never accepts SQL or shell commands from the controller. Supported
actions are `install`, `update`, `uninstall`, and `load`. Release assets retain
the HTTPS, Minisign, SHA-256, archive-path, and atomic-file protections from
`lefred_repo`.

## Build

Place or symlink this directory below MariaDB's `plugin/` directory:

```sh
cmake -S . -B build -DPLUGIN_BANQUISE_AGENT=DYNAMIC
cmake --build build --target banquise_agent
```

Dependencies are libcurl, libarchive, and OpenSSL development headers.

## Configuration

```ini
[mariadb]
plugin_load_add=banquise_agent
banquise_agent_controller_url=https://banquise.example.com
banquise_agent_trusted_key_file=/etc/banquise/catalog.pub
banquise_agent_enrollment_token_file=/etc/banquise/enrollment.token
banquise_agent_state_file=/var/lib/mysql/banquise-agent.state
banquise_agent_poll_interval=60
banquise_agent_enabled=ON
# Optional; otherwise a stable UID is generated and stored in state_file.
# banquise_agent_server_uid=production-db-01
```

Install the catalog public key as root-owned mode `0644`, and the enrollment
token as `root:mysql` mode `0640` (replace `mysql` with the MariaDB service
group). The MariaDB OS account must be able to read the token at first
enrollment and create the state file. After successful
enrollment, the token file is no longer read; the unique returned credential is
stored mode `0600` in `state_file`.

The new server appears in quarantine in the Banquise UI. It sends heartbeats
but receives no tasks until an administrator approves it.

Runtime controls:

```sql
SET GLOBAL banquise_agent_enabled=OFF;
SET GLOBAL banquise_agent_poll_interval=300;
SHOW STATUS LIKE 'banquise_agent_message';
SELECT * FROM information_schema.banquise_agent;
```
