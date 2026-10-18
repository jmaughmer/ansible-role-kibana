# ansible-role-kibana

An Ansible role that installs and configures Kibana 9.x on RHEL, AlmaLinux and Rocky Linux 9 and 10, and on Ubuntu 24.04 and 26.04. It can manage one instance or several behind a load balancer.

What it does:

- Serves Kibana over HTTPS, using certificates from the `ssl_certificates` host variable.
- Connects to Elasticsearch over verified TLS, with a service account token (recommended) or the `kibana_system` user.
- Before changing anything, checks from every Kibana host that Elasticsearch is reachable, trusts the CA, accepts the credentials and isn't older than Kibana.
- Keeps secrets in the Kibana keystore: Elasticsearch credentials, encryption keys and the key passphrase.
- Checks that encryption keys are identical on every instance.
- Sets node roles (`ui`, `background_tasks`) and the Node.js heap per host.
- Restarts instances in batches of limited size when configuration changes, and upgrades them in a coordinated way when the version changes.
- Works over high-latency, low-bandwidth links such as satellite. See [Slow and unreliable networks](#slow-and-unreliable-networks).

## Requirements

- **Control node:** ansible-core 2.15 or later, plus the `ansible.posix` and `community.general` collections:

  ```bash
  ansible-galaxy collection install -r requirements.yml
  ```

- **Target hosts:** RHEL, AlmaLinux or Rocky Linux 9 or 10, or Ubuntu 24.04 or 26.04 (see [Supported platforms](#supported-platforms)), with `openssl` installed. On STIG-hardened hosts see [Hardened hosts](#hardened-hosts-fips-selinux-fapolicyd) — fapolicyd has to allow Ansible before anything runs. They need a package source: Elastic's repository (`artifacts.elastic.co`) or a mirror, or (Enterprise Linux only) a Red Hat Satellite or Katello server. See [Package source](#package-source).
- **Elasticsearch 9.x** at the same or a newer minor version than Kibana, reachable from the Kibana hosts. For example, one deployed with [ansible-role-elasticsearch](https://github.com/jmaughmer/ansible-role-elasticsearch).

## Supported platforms

| Distribution | `ansible_facts['distribution']` | Versions |
|---|---|---|
| Red Hat Enterprise Linux | `RedHat` | 9, 10 |
| AlmaLinux | `AlmaLinux` | 9, 10 |
| Rocky Linux | `Rocky` | 9, 10 |
| Ubuntu | `Ubuntu` | 24.04, 26.04 |

- **Package managers.** Kibana comes from Elastic's RPM packages through dnf on Enterprise Linux, and from Elastic's Debian packages through apt on Ubuntu. Everything else runs the same tasks on both.
- **Ubuntu differences.** The firewall is ufw instead of firewalld (see [Firewall](#firewall)). SELinux and fapolicyd are normally absent, so their `auto` settings do nothing. Satellite is Enterprise Linux only.
- **Other distributions.** Preflight stops the run on any other distribution or major version. To try another Enterprise Linux 9 or 10 rebuild, such as CentOS Stream (`CentOS`) or Oracle Linux (`OracleLinux`), add its distribution fact to `kibana_supported_distributions`. Those aren't tested.
- **Satellite or Katello.** `kibana_repo_source: satellite` needs `subscription-manager` and a host registered to the server. RHEL hosts can use Red Hat Satellite or Katello. AlmaLinux and Rocky Linux hosts are normally registered to Katello; check your Satellite version's supported host operating systems before using it for them.
- **Elastic's support.** See [Elastic's support matrix](https://www.elastic.co/support/matrix) for the operating systems Elastic supports for your Kibana release.

## Installation

Add the role to your project's `requirements.yml`:

```yaml
roles:
  - name: kibana
    src: https://github.com/jmaughmer/ansible-role-kibana.git
    scm: git
    version: main          # or a release tag

collections:
  - name: ansible.posix
    version: ">=1.5.0"
  - name: community.general
    version: ">=8.0.0"
```

```bash
ansible-galaxy install -r requirements.yml
```

## Package source

`kibana_repo_source` selects where the `kibana` package comes from:

| Value | Package source |
|---|---|
| `elastic` (default) | Elastic's public repository, or a mirror set with `kibana_repo_baseurl` and `kibana_repo_gpgkey`. On Enterprise Linux the role writes a disabled repository file and enables it only for its own dnf tasks, so a routine `dnf update` never upgrades Kibana. On Ubuntu it saves the key to `/etc/apt/keyrings/elastic.asc`, adds `/etc/apt/sources.list.d/elastic-9.x.sources` (suite `stable`, component `main`) and holds the package, so a routine `apt upgrade` never upgrades it. The default `kibana_repo_baseurl` is the yum or apt repository to match the host. |
| `satellite` | Enterprise Linux only. A repository synced into Red Hat Satellite or Katello, delivered to registered hosts through subscription-manager. See [Red Hat Satellite or Katello](#red-hat-satellite-or-katello). |
| `none` | Repositories you manage yourself. The role installs from whatever is enabled on the host. |

**Ubuntu and other Elastic roles.** The elasticsearch, logstash and elastic_fleet roles write the same key file and apt source, so a host running more than one of them keeps a single definition. apt refuses two sources for one URL with different keys, so remove any Elastic source added by hand (for example `elastic-9.x.list` from Elastic's install guide) first. To upgrade Kibana outside the role, `apt-mark unhold kibana` first.

**Version check.** Before installing or upgrading, the role checks that `kibana_version` is available from the package source, so a missing version fails before anything changes. You can turn this off with `kibana_package_version_check`.

**Migrating.** `kibana_repo_source` replaces `kibana_manage_repo`; use `kibana_repo_source: none` where you had `false`. Preflight fails if the old variable is still set.

### Red Hat Satellite or Katello

One Satellite repository of Elastic's packages serves all three roles: Elasticsearch, Logstash and Kibana.

**On Satellite:**
1. Create a product, for example `Elastic`.
2. Create a content credential from Elastic's GPG key (`https://artifacts.elastic.co/GPG-KEY-elasticsearch`), so hosts verify package signatures.
3. In that product, create a yum repository with the upstream URL `https://artifacts.elastic.co/packages/9.x/yum` and the content credential, then sync it.
4. Add the repository to the content view the hosts use, then publish and promote it to their lifecycle environment.
5. Make sure the hosts' activation key gives them access to the product.

**On a registered host,** find the repository ID. It is normally `<organization label>_<product label>_<repository label>`:

```bash
subscription-manager repos --list | grep -i -A4 elastic
```

**Role variables:**

```yaml
kibana_repo_source: satellite
kibana_satellite_repo_id: ACME_Elastic_Elastic_9_x

# Optional
kibana_satellite_repo_state: disabled      # unmanaged (default), enabled or disabled
kibana_satellite_repo_overrides:           # applied with subscription-manager repo-override
  timeout: "600"
  retries: "20"
kibana_satellite_gpgkey: ""                # extra key to import (URL or path on the host)
```

**What the role does in `satellite` mode:**
- **No repository file.** subscription-manager owns `redhat.repo`. The role reads it on the host, without contacting Satellite, and passes the repository to dnf with `enablerepo`. Installs and upgrades work whether the repository is enabled or disabled.
- **Checks that `subscription-manager` is installed**, and fails with a clear message if it isn't.
- **Removes its own Elastic repository file** if one is left over from `elastic` mode, so packages don't come from two sources.
- **Checks the repository exists.** If it doesn't, the role runs `subscription-manager refresh` once (`kibana_satellite_refresh`), then fails with the related repository IDs it can see.
- **Enabled state** (`kibana_satellite_repo_state`):
  - `unmanaged` leaves it to Satellite.
  - `enabled` or `disabled` sets it with `subscription-manager repos`.
  - `disabled` keeps routine `dnf update` from upgrading Kibana, like `elastic` mode does.
- **Content overrides** (`kibana_satellite_repo_overrides`) are applied with `subscription-manager repo-override` when a value differs. Use them for dnf options such as `timeout`, `retries` and `minrate`, because `redhat.repo` can't be edited directly. `enabled`, `baseurl` and `name` aren't allowed.
- **GPG keys.** dnf imports the key assigned to the repository in Satellite. Set `kibana_satellite_gpgkey` to import a different key, from a URL or a file on the host.

Enabled-state changes and content overrides made with `subscription-manager` are stored on the Satellite server, as part of that host's settings.

## Quick start

[`examples/`](examples/) deploys two Kibana instances behind a load balancer (`kibana.example.com`):

```
examples/
├── ansible.cfg                      # SSH settings for slow links, roles/collections paths
├── requirements.yml                 # this role + ansible.posix
├── playbook.yml
└── inventory/
    ├── hosts.yml                    # kibana hosts (+ elasticsearch hosts to connect to)
    ├── group_vars/all/network.yml   # retries/timeouts for satellite links
    ├── group_vars/kibana/
    │   ├── main.yml                 # public URL, ES auth, sessions, settings
    │   └── vault.yml                # service account token, encryption keys
    └── host_vars/
        └── kb-N.example.com.yml     # node roles, heap, ssl_certificates
```

**Before the first run:**
1. Create a service account token. The command is in `examples/inventory/group_vars/kibana/vault.yml`.
2. Generate three encryption keys with `openssl rand -hex 32`.
3. Put the token and keys into `vault.yml`, then encrypt it.

Then:

```bash
cd examples
ansible-galaxy install -r requirements.yml
ansible-vault encrypt inventory/group_vars/kibana/vault.yml
ansible-playbook playbook.yml --ask-vault-pass
```

To run the examples against your working copy instead of the published role, link it into place instead of installing it:

```bash
mkdir -p roles && ln -s ../.. roles/kibana
```

Useful tags: `kibana_config`, `kibana_keystore`, `kibana_tls`, `kibana_service`.

## Elasticsearch authentication

**Service account token (recommended).** Create a token once with an admin user, then vault its `token.value`:

```bash
curl -u elastic --cacert ca.crt -X POST \
  "https://es-data-1.example.com:9200/_security/service/elastic/kibana/credential/token/kibana-prod"
```

```yaml
kibana_elasticsearch_auth: service_account_token
kibana_elasticsearch_service_account_token: "{{ vault_kibana_service_account_token }}"
```

**Basic auth with `kibana_system`:**

```yaml
kibana_elasticsearch_auth: basic
kibana_elasticsearch_username: kibana_system
kibana_elasticsearch_password: "{{ vault_kibana_system_password }}"
# Optional: set the password in Elasticsearch when it does not authenticate
kibana_elasticsearch_manage_password: true
kibana_elasticsearch_admin_password: "{{ vault_elasticsearch_elastic_password }}"
```

**How the connection is checked** (`kibana_elasticsearch_check`). From each Kibana host, the role calls `GET /` on every host in `kibana_elasticsearch_hosts`, using Kibana's credentials and the configured CA. It runs *before* the keystore or `kibana.yml` are written.

| Result | What happens |
|---|---|
| A host rejects the credentials | The run fails. |
| No host is reachable | The run fails. |
| Some hosts are unreachable | A warning is shown and the run continues. |
| Elasticsearch is an older major.minor than `kibana_version` | The run fails. |

`kibana_elasticsearch_hosts` defaults to every host in the `elasticsearch` inventory group (`kibana_elasticsearch_group`), on port 9200. The hosts don't have to be managed by the same playbook.

## Encryption keys

`kibana_security_encryption_key` and `kibana_encrypted_saved_objects_encryption_key` are required; `kibana_reporting_encryption_key` is optional.

- Each must be at least 32 characters. Generate one with `openssl rand -hex 32`.
- They must be **identical on every instance** of a deployment. Otherwise sessions break when the load balancer switches instances, and encrypted saved objects (alerting rules, connectors) can't be decrypted.
- Preflight compares SHA-256 fingerprints of the keys across all hosts in the play.

## TLS: `ssl_certificates`

Certificates are supplied per host, with the private key vaulted:

```yaml
# host_vars/kb-1.example.com.yml
ssl_certificates:
  - name: "kb-1.example.com"
    certificate:
      content: |
        -----BEGIN CERTIFICATE-----
        ...
    key:
      content: !vault |
        $ANSIBLE_VAULT;1.1;AES256
        ...
      # passphrase: !vault | ...             # only for encrypted PEM keys
    # ca:
    #   content: |                           # optional, overrides kibana_ssl_ca_certificate
```

The role uses the entry named `kibana_ssl_certificate_name`, which defaults to the inventory hostname. `kibana_ssl_ca_certificate` is trusted for Elasticsearch unless `kibana_elasticsearch_ca_certificate` is set. The role checks that the key matches the certificate and that the certificate chains to the CA.

### Certificates issued by ansible-role-elastic-tls

[ansible-role-elastic-tls](https://github.com/jmaughmer/ansible-role-elastic-tls) can issue these certificates instead, from a private CA it creates and keeps, or from
Let's Encrypt. Run it first in the same play:

```yaml
  roles:
    - role: elastic_tls
    - role: kibana
```

It publishes what it issued as `elastic_tls_ssl_certificates`, which this role reads **after**
`ssl_certificates`. A manual entry with the same name always wins, so hosts can be moved across one
at a time. Match the name this role looks for: ``kibana_ssl_certificate_name``.

Entries in either list can point at a file already on the host instead of carrying its content:

```yaml
ssl_certificates:
  - name: kb-1.example.com
    certificate: {path: /etc/pki/ansible-tls/certs/kb-1.example.com.crt}
    key: {path: /etc/pki/ansible-tls/private/kb-1.example.com.key}
    ca: {path: /etc/pki/ansible-tls/certs/ca.crt}
```

``kibana_ssl_ca_certificate_path`` does the same for the CA when it is not part of the entry. Inline content wins when an
entry carries both.

Note that Let's Encrypt issues server certificates only. That suits the address users open in a browser, which is the one endpoint here that benefits from a publicly trusted certificate. Kibana still has to trust Elasticsearch separately, so with an ACME certificate for the browser side, point `kibana_elasticsearch_ca_certificate_path` at the CA the cluster uses.

## Node roles and sizing

```yaml
# host_vars
kibana_node_roles: [ui]                 # UI only, behind the load balancer
kibana_node_roles: [background_tasks]   # dedicated alerting / task manager instance
kibana_max_old_space_size_mb: 4096      # written to node.options
```

Other settings go in `kibana_settings_extra` as flat keys. They are merged last and override anything the role renders:

```yaml
kibana_settings_extra:
  xpack.task_manager.capacity: 20
  server.maxPayload: 1048576
```

The role builds `kibana.yml` from these settings as one YAML map, so there are never duplicate keys, and numbers and booleans keep their types.

## Restarts and upgrades

**Changes that need a restart:**

- `kibana.yml`
- keystore entries
- certificates
- `node.options`
- `/etc/sysconfig/kibana` (`/etc/default/kibana` on Ubuntu)
- the systemd drop-in

**Configuration changes (same version).** Running instances are restarted in batches of `kibana_restart_batch_size`. Each batch waits until `/api/status` answers before the next batch starts.

- Kibana answers 503 while starting.
- If `kibana_status_username` and `kibana_status_password` are set, the overall status level must also be `available`.
- An invalid setting stops the rollout at the first batch.

**Version upgrades.** Kibana does **not** support rolling upgrades: saved object migrations need exclusive use of the Kibana system indices. When `kibana_version` changes, the role:

1. Checks that Elasticsearch is already at the new version or newer.
2. Stops **every** instance in the play. Preflight refuses to upgrade if `kibana_group` has instances outside the play, unless `kibana_upgrade_allow_partial` is set.
3. Upgrades the packages.
4. Starts the first instance and waits, up to `kibana_migration_retries` × `kibana_ready_delay`, for migrations to finish.
5. Starts the remaining instances.

Kibana is unavailable while this runs.
- Upgrade Elasticsearch before Kibana.
- Run upgrades against the whole `kibana` group, not with `--limit`.

Don't use `serial` on the play; preflight checks for it. Keep `any_errors_fatal: true`.

## Behind a load balancer

- Set `kibana_server_public_base_url` to the load balancer's URL. Include that name in the certificate SANs if the load balancer passes TLS through.
- If TLS ends at the load balancer, set `kibana_server_ssl_enabled: false`. `xpack.security.secureCookies` stays on while the public URL is `https`.
- Use `/api/status` as the load balancer health check. It returns 503 while an instance is starting or migrating.

## Slow and unreliable networks

The role is built to work over high-latency, low-bandwidth links such as satellite. It doesn't rely on SSH pipelining.

**Connection settings.** Put these in your project's `ansible.cfg`; [`examples/ansible.cfg`](examples/ansible.cfg) has them.
- A 60 s connect timeout.
- Keepalives every 30 s, and a connection is only given up after 10 missed replies.
- Compression, and control connections that persist for 30 minutes.
- `retries = 10`, so Ansible reconnects when SSH drops mid-task.
- `gather_timeout = 120`.

**What the role does.**
- **Downloads.** Package downloads run as async jobs. They keep running on the host if the SSH session drops, and Ansible polls for the result. In `--check` mode they run as normal tasks, because Ansible rejects async tasks in check mode.
  - Failed downloads are retried.
  - The dnf repository gets a longer per-request timeout and more retries: `kibana_repo_timeout` and `kibana_repo_retries` for the Elastic repository, or `kibana_satellite_repo_overrides` for a Satellite repository.
  - dnf waits for a lock held by an interrupted earlier attempt.
- **Elasticsearch check.** Hosts that can't be reached are retried, but a rejection (401/403) fails immediately. Setting the `kibana_system` password is retried and safe to repeat.

The defaults suit ordinary networks. For satellite links, use the values in [`examples/inventory/group_vars/all/network.yml`](examples/inventory/group_vars/all/network.yml):

| Variables | Value |
|---|---|
| `kibana_task_retries`, `kibana_task_delay` | 8 retries, 60 s apart |
| `kibana_download_async_timeout` | 4 hours per download attempt |
| `kibana_api_timeout` | 180 s per request |
| `kibana_ready_retries`/`_delay`, `kibana_stop_retries`/`_delay` | 180 × 20 s |

A retried task can take up to retries × (delay + attempt time) before it gives up. If many hosts share one link, lower `forks`.

## Firewall

With `kibana_manage_firewall` (on by default) the role opens `kibana_server_port` in firewalld,
permanently and immediately, in `kibana_firewall_zone`. On a host running ufw instead (Ubuntu), it
adds `allow` rules for the same ports:

| Variable | Default | |
|---|---|---|
| `kibana_manage_firewall` | `true` | Set to `false` to manage the firewall yourself |
| `kibana_firewall_zone` | `public` | firewalld zone the rules go in |
| `kibana_firewall_ports` | `[]` | Anything else this host serves, e.g. `["80/tcp"]` for a redirect |
| `kibana_firewall_services` | `[]` | Named firewalld services (firewalld only; list their ports for ufw) |

The role never starts or enables a firewall. If neither firewalld nor ufw is active it says which
port still has to be reachable rather than failing.

## Hardened hosts (FIPS, SELinux, fapolicyd)

The role is meant to run on hosts with the DISA STIG applied. What that takes:

### fapolicyd

**Ansible itself has to be allowed first.** fapolicyd blocks whatever it does not trust, and that
includes the Python module Ansible copies to the host for *every* task. Nothing runs until that is
allowed, so add a rule once, before using these roles:

```
# /etc/fapolicyd/rules.d/30-ansible.rules
allow perm=open exe=/usr/bin/python3.9 : dir=/root/.ansible/tmp/
```

Load it with `fagenrules --load`. The file has to sort before the rules that deny untrusted
interpreted files, and the interpreter and directory have to match the host's Python and the user you
become (`ansible_remote_tmp`). The same rule covers the `repo_lookup.py` helper these roles run in
Satellite mode. Denials are audited, so `ausearch -m fanotify -ts recent` shows what was blocked.

Beyond that, Kibana needs nothing: everything it runs comes from the rpm, so the trust database
already covers it. The role therefore has no fapolicyd settings.

### FIPS mode

Kibana has no FIPS mode. It runs the Node.js build it ships with, which uses its own OpenSSL and
does not inherit the system policy, and Elastic does not validate it. A host in FIPS mode runs Kibana
unchanged.

What matters in practice:

- **The server private key must not be in the traditional encrypted PEM format** (`Proc-Type:
  4,ENCRYPTED`), whose key derivation uses MD5. Preflight fails with that message when the host
  reports FIPS; convert it with `openssl pkcs8 -topk8 -in old.key -out new.key`, or let
  [ansible-role-elastic-tls](https://github.com/jmaughmer/ansible-role-elastic-tls) issue it.
- **TLS protocols and ciphers** can be stated explicitly, for evidence:

  ```yaml
  kibana_server_ssl_supported_protocols: ["TLSv1.3", "TLSv1.2"]
  kibana_server_ssl_cipher_suites: ["TLS_AES_256_GCM_SHA384"]
  ```

  Empty keeps Kibana's defaults, which are already TLS 1.3 and 1.2.
- **Session limits** the STIG asks for are `kibana_session_idle_timeout` and `kibana_session_lifespan`.

### SELinux

Kibana ships no SELinux policy and runs in `unconfined_service_t`, which works under a targeted
policy in enforcing mode. `kibana_path_data` and the directory holding `kibana_log_file` are labelled
correctly where the package puts them; moved elsewhere they come out as `default_t`, and
`kibana_selinux_manage: auto` adds an `fcontext` rule and runs `restorecon` for the ones you changed.

- **Ansible needs `python3-libselinux`** on the hosts, or the file and copy modules cannot set
  contexts. It is part of a normal RHEL install.
- **`semanage` comes from `policycoreutils-python-utils`,** which is required only when the role
  manages contexts.

### noexec /tmp

The STIG mounts `/tmp`, `/var/tmp` and `/dev/shm` with `noexec`. Kibana needs nothing for that:
Node.js loads its native modules from Kibana's home, and Reporting's headless Chromium runs from there
too, keeping only its profile and shared memory in `/tmp` and `/dev/shm`. The `kvm-noexec` scenario
tests both (see [Testing](#testing)).

## Notes and caveats

- **Check mode** is not supported for first-time installs, because the keystore CLI needs the Kibana binaries. It works on hosts where Kibana is already installed.

## Variables

[`defaults/main.yml`](defaults/main.yml) documents every variable.

## Testing

[`molecule/default/`](molecule/default/) runs the role against three systemd containers on one
container network: `kes-1`, a single Elasticsearch node at the role's default `kibana_version`, and
two Kibana instances, `kb-1` and `kb-2`. [`prepare.yml`](molecule/default/prepare.yml) makes a test CA
and a certificate for every host, starts Elasticsearch with TLS on HTTP, and creates a service
account token for `elastic/kibana`. The Kibana certificates are left on the hosts as files, as
ansible-role-elastic-tls leaves them. The sequence is:

1. **converge**, then **idempotence**: a second converge must change nothing;
2. **check**: converge in check mode on the installed hosts, which must not fail;
3. **side effect** ([`side_effect.yml`](molecule/default/side_effect.yml)):
   - a `kibana.yml` change restarts both instances, one per batch;
   - a keystore change with `kibana_restart_on_change: false` is recorded and flagged, but
     restarts nothing;
   - `kibana_force_restart` restarts both;
4. **verify** ([`verify.yml`](molecule/default/verify.yml)):
   - Kibana is running and enabled at the default version;
   - the configuration, keystore, keystore state, certificates and key have their owners and
     modes, and `kibana.yml` holds no secret;
   - the keystore holds exactly the managed entries;
   - the heap size, file limit and service environment are applied;
   - `/api/status`, reached through the server certificate, reports Kibana, its Elasticsearch
     connection and saved objects `available`;
   - a session cookie issued by one instance is accepted by the other, so the encryption keys
     match.

Other scenarios:

| Scenario | What it covers |
|---|---|
| `basic` | Basic authentication as `kibana_system`, whose password the role sets with admin credentials, and TLS material given inline with an encrypted key and its passphrase. |
| `upgrade` | Two instances at an older 9.x release are upgraded to the default version. An upgrade run that leaves an instance out is refused first. Every instance stops before the first starts again, and that one starts alone. Going back to the older release is refused. |
| `preflight` | Settings preflight must refuse, each in its own check, without printing a secret. These include an unsupported distribution, a host outside `kibana_group`, an inexact or 8.x version, a bad package source, unknown node roles, missing or forbidden credentials, short or differing encryption keys, a malformed firewall port, and missing TLS material. Nothing is installed. |
| `docker` | The default scenario on Docker instead of Podman. |

```bash
pip install -r molecule/requirements.txt
```

```bash
for s in default basic upgrade preflight; do molecule test -s "$s" || break; done
```

The scenarios use Podman (rootless works), except `docker`, which needs Docker. They download
Elasticsearch and Kibana from Elastic's repository, and Elasticsearch and two Kibana instances need
about 4 GB of memory. `MOLECULE_DISTRO` picks the image (default `rockylinux9`; also `rockylinux10`):

```bash
MOLECULE_DISTRO=rockylinux10 molecule test -s preflight
```

Lint with `yamllint .` and `ansible-lint` (configured by [`.yamllint`](.yamllint) and
[`.ansible-lint`](.ansible-lint)). There is no GitHub Actions workflow; run the lint and Molecule
scenarios locally.

Not covered by containers: SELinux contexts for relocated directories, firewalld rules,
FIPS mode, and Satellite repositories. These need virtual machines with SELinux enforcing,
firewalld running, FIPS enabled, or a Satellite registration.

### On KVM virtual machines

[`molecule/kvm/`](molecule/kvm/) runs the default scenario on real AlmaLinux 10 virtual machines
instead of containers: the same hosts, groups, playbooks and inventory, one VM per host, on local
KVM/libvirt (`qemu:///system`). Each VM has its own kernel, so SELinux is enforcing and settings such
as `vm.max_map_count` are the VM's own. Overrides for the VMs go in
`molecule/kvm/inventory/group_vars/<group>/kvm.yml`.

[`molecule/kvm-ubuntu2404/`](molecule/kvm-ubuntu2404/) and
[`molecule/kvm-ubuntu2604/`](molecule/kvm-ubuntu2604/) run the same thing on Ubuntu 24.04 and 26.04
VMs. They only swap the cloud image, and share `molecule/kvm/`'s playbooks and inventory.

```bash
for s in kvm kvm-ubuntu2404 kvm-ubuntu2604; do molecule test -s "$s" || break; done
```

The machine running Molecule needs libvirt, `virt-install`, `qemu-img` and `xorriso`, and its user
must be able to manage `qemu:///system`. Each cloud image downloads once, checksum verified, to
`/var/lib/libvirt/images/molecule`; each VM gets a throwaway disk on top of it. VM names include the
scenario, so the three scenarios can also run side by side if the machine has the memory.

[`molecule/kvm-noexec/`](molecule/kvm-noexec/) runs the kvm scenario with `/tmp`, `/var/tmp`,
`/dev/shm` and `/home` mounted `noexec`, as the STIG has them, and also renders a page with
Reporting's headless Chromium the way Reporting runs it (`molecule test -s kvm-noexec`).

## Role layout

```
├── defaults/main.yml
├── vars/main.yml
├── meta/main.yml
├── files/repo_lookup.py        # reads repository definitions, including redhat.repo
├── tasks/
│   ├── main.yml
│   ├── facts.yml / preflight.yml
│   ├── repository.yml              # Elastic repository file, or Satellite repository checks
│   ├── package_version_check.yml   # requested version is available before install/upgrade
│   ├── install.yml
│   ├── system.yml               # systemd drop-in, sysconfig, node.options heap, firewalld
│   ├── certificates.yml
│   ├── elasticsearch_check.yml  # connectivity, credentials, version compatibility
│   ├── keystore.yml
│   ├── configure.yml            # kibana.yml
│   ├── service.yml / wait_ready.yml
│   ├── rolling_restart.yml / restart_batch.yml
│   └── upgrade.yml              # coordinated all-instance upgrade
├── templates/
├── examples/
└── molecule/                   # default, basic, upgrade, preflight (Podman) and docker scenarios
```

## Related roles

- [ansible-role-elasticsearch](https://github.com/jmaughmer/ansible-role-elasticsearch): the cluster Kibana connects to.
- [ansible-role-logstash](https://github.com/jmaughmer/ansible-role-logstash)
- [ansible-role-elastic-tls](https://github.com/jmaughmer/ansible-role-elastic-tls): issues the certificates these roles install, from a private CA or Let's Encrypt.
- [ansible-role-keepalived](https://github.com/jmaughmer/ansible-role-keepalived): a virtual address so users reach one name whichever instance is serving.

All three use the same `ssl_certificates` host variable format.

## License

MIT. See [LICENSE](LICENSE).
