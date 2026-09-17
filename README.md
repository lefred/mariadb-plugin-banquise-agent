# Banquise MariaDB agent - mariadb-plugin-banquise-agent

![mariabd-plugin-banquise-agent](logo/banquise_agent_logo.png)

`banquise_agent` is the pull-based MariaDB component of Banquise. It maintains
an `INFORMATION_SCHEMA.BANQUISE_CATALOG` catalog view, registers once with the
control plane, heartbeats with MariaDB/OS/architecture and observed plugin
state, and executes typed signed-catalog tasks queued by an administrator.

The agent never accepts SQL or shell commands from the controller. Supported
actions are `install`, `update`, `uninstall`, and `load`. Release assets retain
the HTTPS, Minisign, SHA-256, archive-path, and atomic-file protections from
`lefred_repo`.

## Centrally managed catalogs

Banquise Server defines named catalog URLs and signing public keys in
**Administration → Agent catalogs**. An administrator verifies each catalog,
then a fleet manager assigns catalogs from the agent's detail page. Catalogs
may share plugin names; every task identifies its repository explicitly.

The agent does not read local `[banquise:name]` sections. Those sections belong
to Banquise Lite. Provision trusted public keys locally instead:

The shared repository configuration parser accepts `enabled=1` (the default)
and `enabled=0`; this setting is used by Banquise Lite. Agent catalog
assignments are enabled or disabled centrally in Banquise Server.

```ini
[mariadb]
banquise_agent_trusted_keys_dir=/etc/mariadb/banquise/trusted.d
```

```sh
sudo install -d -o root -g root -m 0755 /etc/mariadb/banquise/trusted.d
sudo install -o root -g root -m 0644 community.pub /etc/mariadb/banquise/trusted.d/community.pub
sudo install -o root -g root -m 0644 internal.pub /etc/mariadb/banquise/trusted.d/internal.pub
```

Obtain and authenticate these keys through an administrative channel. The
controller supplies only catalog names, HTTPS URLs, and key IDs. It cannot
supply a local key path or install trusted keys. The directory must be
root-owned, not a symlink, and not group- or world-writable. Keys must satisfy
the same ownership/write rules and be regular, non-symlink `.pub` files under
4 KiB. Key additions and removals are detected at the next poll; changing the
directory setting requires a restart. An empty directory reports no trusted
keys and permits no catalog assignments.

Enrollment and each heartbeat report `catalog_key_ids`. The server sends:

```json
{
  "status": "active",
  "catalogs": [
    {"name": "community", "url": "https://example.org/catalog.json", "key_id": "0123456789abcdef"}
  ],
  "tasks": [
    {"id": 42, "action": "install", "plugin_name": "vmstat", "catalog_name": "community"}
  ]
}
```

Assigned catalogs are refreshed independently. A catalog that has a DNS,
network, key, signature, or parsing failure is skipped while available catalogs
remain usable; if every assigned catalog fails, the refresh fails and the
previous catalog view is retained. Unknown keys, duplicate repository names,
and tasks for unassigned repositories are rejected. An empty assignment list
clears the catalog view. SQL refresh uses the latest assignments received from
the server; it never discovers catalog URLs locally.

```sql
SELECT CATALOG, NAME FROM information_schema.BANQUISE_CATALOG;
SELECT banquise_agent_install('vmstat', 'community');
SELECT banquise_agent_update('vmstat', 'community');
SELECT banquise_agent_uninstall('vmstat', 'community');
```

The SQL repository argument is optional only when exactly one assigned catalog
offers the compatible plugin. Inventory reports retain the repository name.
Update Banquise Server and Agent together; the old single-catalog protocol is
not supported. Existing agent credentials continue to work. Repository
assignments must be made explicitly after the first new heartbeat.

## Build

Place or symlink this directory below MariaDB's `plugin/` directory:

```sh
cmake -S . -B build -DPLUGIN_BANQUISE_AGENT=DYNAMIC
cmake --build build --target banquise_agent
```

Dependencies are libcurl and libarchive development headers. OpenSSL is used when available; MariaDB builds using bundled wolfSSL automatically use the native wolfCrypt verifier. Set `-DBANQUISE_AGENT_CRYPTO_BACKEND=OPENSSL` or `WOLFSSL` to select explicitly.

## Configuration

```ini
[mariadb]
plugin_load_add=banquise_agent
banquise_agent_controller_url=https://banquise.example.com
banquise_agent_trusted_keys_dir=/etc/mariadb/banquise/trusted.d
banquise_agent_enrollment_token_file=/etc/mariadb/banquise/enrollment.token
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
SELECT * FROM information_schema.banquise_catalog;
```

`banquise_agent` and `banquise_lite` are mutually exclusive. Both modules
register the `BANQUISE_CATALOG` information-schema component, so MariaDB rejects
an attempt to install the second module before any of its service components
are activated. Neither module can manage the other's shared object as a catalog
plugin.

When upgrading from an Agent build that exposed
`INFORMATION_SCHEMA.BANQUISE_AGENT`, uninstall that old module before replacing
the shared object, then install the new module so MariaDB persists both the
`BANQUISE_CATALOG` table component and the `BANQUISE_AGENT` daemon component:

```sql
UNINSTALL SONAME 'banquise_agent';
-- replace banquise_agent.so
INSTALL SONAME 'banquise_agent';
```

## Tests

Run the `banquise_agent` suite with MariaDB's `mariadb-test-run.pl`.
The standalone configuration and repository selection tests need only C++11:

```sh
c++ -std=c++11 -Wall -Wextra -Werror tests/repositories.cc -o /tmp/banquise-repositories-test
/tmp/banquise-repositories-test
```
