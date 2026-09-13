# Ansible Role: samba_ad_sssd

![GitHub](https://img.shields.io/github/license/jomrr/ansible-role-samba_ad_sssd)
![GitHub last commit](https://img.shields.io/github/last-commit/jomrr/ansible-role-samba_ad_sssd)
![GitHub issues](https://img.shields.io/github/issues-raw/jomrr/ansible-role-samba_ad_sssd)
[![dev](https://img.shields.io/github/actions/workflow/status/jomrr/ansible-role-samba_ad_sssd/dev.yml?branch=dev&event=push&label=dev)](https://github.com/jomrr/ansible-role-samba_ad_sssd/actions/workflows/dev.yml?query=branch%3Adev)
[![main](https://img.shields.io/github/actions/workflow/status/jomrr/ansible-role-samba_ad_sssd/main.yml?branch=main&event=push&label=main)](https://github.com/jomrr/ansible-role-samba_ad_sssd/actions/workflows/main.yml?query=branch%3Amain)

Join Linux systems to Active Directory with SSSD, RFC2307 identities, native PAM
integration, and Linux computer GPOs.

## Purpose

Establish AD account logins on Linux servers and workstations with SSSD. The
default reads centrally assigned RFC2307 UID/GID attributes from AD and applies
Linux computer GPOs.

## Scope

### Managed

- AD join and machine keytab through jomrr.samba.samba_join_sssd (adcli).
- SSSD configuration, NSS identity lookup, PAM authentication and optional home
  creation.
- Enabled and running SSSD and the native oddjob broker where needed for GPOs or
  home creation.
- Samba Linux computer GPO client, initial Samba machine credentials through
  samba_join_member, synchronized keytab, and optional periodic refresh.

### Not Managed

- Samba file shares, smbd, winbind, and file ownership migrations.
- DC provisioning, RFC2307 attribute allocation, DNS resolver setup, time
  synchronization, and host naming.
- Domain leave, automatic rejoin after trust failures, and authoring or linking
  AD policies.
- User GPO application at login; applications of computer GPOs depend on
  installed Samba extensions.

## Requirements

- A stable host FQDN, working AD DNS discovery, synchronized time, and network
  reachability to the domain.
- For RFC2307: users need uidNumber and gidNumber, groups need gidNumber;
  allocate these centrally and consistently. Provisioning the schema alone does
  not assign them.
- Native PAM configuration must be managed by authselect (Red Hat),
  pam-auth-update (Debian/Ubuntu), or pam-config (openSUSE). Set
  authselect_force explicitly to adopt unmanaged Red Hat files.
- Applications must use the system PAM stack. SSH password logins also require
  suitable sshd configuration managed outside this role.
- Samba 4.21 or later for native sync machine password to keytab support. The
  DNS realm must resolve to domain controllers when no explicit
  samba_ad_sssd_server is configured.
- Set gpo_workgroup to the actual AD NetBIOS domain when it differs from the
  first DNS realm label. This role owns smb.conf and targets SSSD clients
  without an existing Samba server configuration.
- Linux computer policies must be authored and linked in AD using extensions
  supported by the installed Samba version. Keep GPO payloads separate from
  files managed by this role to avoid configuration conflicts.

## Dependencies

```yaml
collections:
  - name: community.general
    version: '>=12.0.0'
  - name: jomrr.samba
    version: '>=2.0.0'
```

## Role Variables

### `samba_ad_sssd_realm`

Type: `str`. Required: `true`.

AD DNS domain and Kerberos realm; must remain stable after joining.

### `samba_ad_sssd_join_username`

Type: `str`. Required: `false`.

Account delegated permission to join computers to the domain.

Default:

```yaml
samba_ad_sssd_join_username: Administrator
```

### `samba_ad_sssd_join_password`

Type: `str`. Required: `false`.

Join password from a secret store; needed for the initial SSSD/GPO setup or
forced rejoin.

### `samba_ad_sssd_server`

Type: `str`. Required: `false`.

DC hostname for initial joins; empty uses adcli discovery and the DNS realm for
Samba. Runtime DC selection uses domain_options.ad_server.

Default:

```yaml
samba_ad_sssd_server: ''
```

### `samba_ad_sssd_computer_ou`

Type: `str`. Required: `false`.

LDAP DN of the computer OU; empty uses the domain default.

Default:

```yaml
samba_ad_sssd_computer_ou: ''
```

### `samba_ad_sssd_host_fqdn`

Type: `str`. Required: `false`.

Stable fully qualified machine hostname shared by adcli and SSSD.

Default:

```yaml
samba_ad_sssd_host_fqdn: '{{ ansible_facts.fqdn | lower }}'
```

### `samba_ad_sssd_keytab`

Type: `path`. Required: `false`.

Machine keytab used by adcli and SSSD; its parent directory must exist.

Default:

```yaml
samba_ad_sssd_keytab: /etc/krb5.keytab
```

### `samba_ad_sssd_force_join`

Type: `bool`. Required: `false`.

Rejoin on every run; enable only for deliberate machine account repair.

Default:

```yaml
samba_ad_sssd_force_join: false
```

### `samba_ad_sssd_id_mapping`

Type: `str`. Required: `false`.

rfc2307 reads centrally assigned POSIX IDs; autorid_compat enables SSSD
algorithmic mapping with autorid compatibility.

Default:

```yaml
samba_ad_sssd_id_mapping: rfc2307
```

### `samba_ad_sssd_sssd_options`

Type: `dict`. Required: `false`.

Native [sssd] options, for example domain_resolution_order. Domains and services
are role-owned.

Default:

```yaml
samba_ad_sssd_sssd_options: {}
```

### `samba_ad_sssd_domain_options`

Type: `dict`. Required: `false`.

Native AD domain options with scalar values and comma-separated strings for
lists. Role-owned provider, identity, keytab and mapping settings take
precedence.

Default:

```yaml
samba_ad_sssd_domain_options:
  access_provider: ad
  ad_gpo_access_control: enforcing
  enumerate: false
  cache_credentials: true
  use_fully_qualified_names: true
  fallback_homedir: /home/%d/%u
  default_shell: /bin/bash
```

### `samba_ad_sssd_nss_options`

Type: `dict`. Required: `false`.

Native [nss] responder options with scalar values, for example filter_users or
filter_groups.

Default:

```yaml
samba_ad_sssd_nss_options: {}
```

### `samba_ad_sssd_pam_options`

Type: `dict`. Required: `false`.

Native [pam] responder options with scalar values, for example
offline_credentials_expiration.

Default:

```yaml
samba_ad_sssd_pam_options: {}
```

### `samba_ad_sssd_config_no_log`

Type: `bool`. Required: `false`.

Redact template output and diffs when native SSSD options contain secrets.

Default:

```yaml
samba_ad_sssd_config_no_log: false
```

### `samba_ad_sssd_mkhomedir`

Type: `bool`. Required: `false`.

Enable native PAM home directory creation on login; disabling removes the
managed PAM feature and preserves homes.

Default:

```yaml
samba_ad_sssd_mkhomedir: true
```

### `samba_ad_sssd_authselect_features`

Type: `list`. Required: `false`.

Additional features of the Red Hat sssd profile; with-mkhomedir is controlled by
mkhomedir.

Default:

```yaml
samba_ad_sssd_authselect_features: []
```

### `samba_ad_sssd_authselect_force`

Type: `bool`. Required: `false`.

Allow authselect to replace existing unmanaged PAM/NSS files using its native
backup mechanism.

Default:

```yaml
samba_ad_sssd_authselect_force: false
```

### `samba_ad_sssd_gpo_workgroup`

Type: `str`. Required: `false`.

AD NetBIOS domain for Samba GPO access; override when it differs from the
realm's first label.

Default:

```yaml
samba_ad_sssd_gpo_workgroup: '{{ samba_ad_sssd_realm.split(".")[0] | upper }}'
```

### `samba_ad_sssd_gpo_refresh_enabled`

Type: `bool`. Required: `false`.

Apply computer GPOs automatically; disabling retains the client and synchronized
machine credentials.

Default:

```yaml
samba_ad_sssd_gpo_refresh_enabled: true
```

### `samba_ad_sssd_gpo_refresh_interval`

Type: `str`. Required: `false`.

Interval between automatic computer policy updates, using systemd time span
syntax.

Default:

```yaml
samba_ad_sssd_gpo_refresh_interval: 90min
```

### `samba_ad_sssd_gpo_randomized_delay`

Type: `str`. Required: `false`.

Maximum random delay added to scheduled policy updates, using systemd time span
syntax.

Default:

```yaml
samba_ad_sssd_gpo_randomized_delay: 30min
```

## Managed Files

- `/etc/krb5.conf default realm and disabled reverse/canonical hostname
  rewriting; other settings are preserved.`
- `/etc/sssd/sssd.conf (0640, root-owned, native SSSD service group, native
  validation and backup).`
- `/etc/nsswitch.conf and native PAM profile selection; PAM files are generated
  by distribution tools.`
- `/etc/samba/smb.conf (global GPO client settings only, native validation and
  backup).`
- `/var/lib/samba/private/secrets.tdb (machine credentials initialized by Samba
  and renewed by SSSD).`
- `/etc/systemd/system/samba-ad-sssd-gpupdate.service and .timer (native
  validation).`

## Check Mode

Supported on an already prepared host; adcli reports a pending join without
performing it.

- A first run in check mode cannot install required tools or create a usable
  machine keytab. Run a normal converge before checking a fresh host.

## Service Behavior

Configuration and keytab changes restart SSSD before native PAM integration. GPO
setup changes apply computer policies immediately when automatic refresh is
enabled.

### Handlers

- restart sssd
- restart oddjob
- reload policy units
- apply computer policies

## Security Notes

- Keep join credentials in Ansible Vault or a secret store. Both join tasks are
  redacted.
- Set config_no_log when native SSSD options contain secret values.
- The default AD access provider enforces GPO access rules. Successful identity
  lookup alone does not grant login or sudo privileges.

## Operational Notes

- samba_ad_sssd_id_mapping=rfc2307 sets id_provider=ad and
  ldap_id_mapping=false. autorid_compat sets ldap_id_mapping=true and
  ldap_idmap_autorid_compat=true.
- Autorid compatibility ignores RFC2307 IDs. Its domain allocation depends on
  discovery order and does not guarantee identical Winbind or cross-host IDs.
  Use domain_options.ldap_idmap_default_domain_sid to pin the main domain and
  configure matching ranges on all clients.
- Choose the mapping mode before deployment. Mapping changes require a planned
  SSSD cache and filesystem ownership migration; the role does not delete caches
  or rewrite ownership.
- domain_options controls enumerate (both users and groups),
  use_fully_qualified_names, caching, home/shell fallbacks, DC selection, ranges
  and access rules. sssd_options accepts domain_resolution_order for unqualified
  name resolution.
- Native option values are scalars; provide SSSD lists as comma-separated
  strings. Dictionaries follow normal Ansible variable replacement semantics;
  include desired defaults when replacing a dictionary.
- The role owns domains, services, id_provider, auth_provider, ad_domain,
  krb5_realm, ad_hostname, both keytab paths,
  ad_update_samba_machine_account_password=true, and the two mapping booleans.
  Other native options remain configurable.
- Existing /etc/sssd/conf.d snippets are included in validation and preserved.
  They must not override the role-owned identity settings.
- SSSD and winbind must not compete as identity providers for this domain. This
  role targets dedicated SSSD clients and does not migrate an existing winbind
  deployment.
- Join idempotency is based on local machine principals in the keytab, not a
  live trust test. Use force_join only for deliberate repair.
- AD user/group enumeration is deprecated since SSSD 2.10 and its extended
  support was removed in 2.12. enumerate is passed through for builds that
  support it; full listings cannot be guaranteed on current distributions. Named
  lookups and logins do not require enumeration.
- The integration fixture uses explicit domain_options.ad_server and krb5_server
  endpoints: current Fedora/openSUSE c-ares resolvers rejected the Samba test DC
  SRV responses with Misformatted DNS reply. Automatic discovery requires DNS
  responses accepted by the installed resolver. No DNS parser or security policy
  is bypassed by the role.
- Linux computer GPOs use oddjob-gpupdate on Fedora and openSUSE, and
  samba-gpupdate directly on AlmaLinux, Debian and Ubuntu. AlmaLinux standard
  repositories do not provide oddjob-gpupdate. The role does not enable or start
  smbd or winbind. On openSUSE, native GPO package dependencies include Samba
  server binaries; no shares are configured.
- SSSD ad_gpo_access_control evaluates login access rights independently of the
  Samba Linux policy client. The role does not configure a PAM hook for user GPO
  execution at login.
- Automatic computer refresh is enabled by default, every 90 minutes with up to
  30 minutes of random delay. The timer also schedules a boot refresh after 5
  minutes plus the random delay. oddjob-gpupdate itself is request-driven and
  does not provide a periodic scheduler.
- Set samba_ad_sssd_gpo_refresh_enabled=false to stop and disable the timer and
  suppress automatic application during Ansible runs. Client packages,
  configuration, and machine credential synchronization remain available. Manual
  refresh uses systemctl start samba-ad-sssd-gpupdate.service, including when
  the timer is disabled. Disabling refresh does not undo already applied
  policies.
- Samba GPO retrieval requires secrets.tdb. After the SSSD join,
  jomrr.samba.samba_join_member initializes it and synchronizes the SSSD keytab
  through Samba's native sync machine password to keytab setting. SSSD keeps
  both stores synchronized during subsequent password renewals through adcli
  --add-samba-data. Initial GPO setup on an existing SSSD client therefore needs
  the join credential again. force_join also refreshes the Samba credentials.
- Initializing secrets.tdb with adcli update --add-samba-data alone fails with
  some current Debian/Ubuntu package combinations. The role uses the native
  Samba join module for initialization; normal SSSD password renewal works once
  the Samba machine credentials exist.
- The Samba default idmap range only satisfies the native ADS client
  configuration. NSS and PAM continue to use SSSD and its RFC2307 or
  autorid_compat mapping; the role does not run Winbind for identity lookup.

## Supported Platforms

| OS Family | Distribution | Version | Container Image |
| --------- | ------------ | ------- | --------------- |
| RedHat | AlmaLinux | latest | [jomrr/molecule-almalinux:latest](https://hub.docker.com/r/jomrr/molecule-almalinux) |
| Debian | Debian | latest | [jomrr/molecule-debian:latest](https://hub.docker.com/r/jomrr/molecule-debian) |
| RedHat | Fedora | latest | [jomrr/molecule-fedora:latest](https://hub.docker.com/r/jomrr/molecule-fedora) |
| Suse | OpenSuse Leap | latest | [jomrr/molecule-opensuse-leap:latest](https://hub.docker.com/r/jomrr/molecule-opensuse-leap) |
| Suse | OpenSuse Tumbleweed | latest | [jomrr/molecule-opensuse-tumbleweed:latest](https://hub.docker.com/r/jomrr/molecule-opensuse-tumbleweed) |
| Debian | Ubuntu | latest | [jomrr/molecule-ubuntu:latest](https://hub.docker.com/r/jomrr/molecule-ubuntu) |

## Example Playbook

### RFC2307 domain login

Use centrally assigned IDs and create homes at login.

```yaml
---
- name: Configure AD logins
  hosts: linux_clients
  gather_facts: true
  roles:
    - role: jomrr.samba_ad_sssd
      samba_ad_sssd_realm: AD.EXAMPLE.COM
      samba_ad_sssd_join_password: "{{ vault_ad_join_password }}"
      samba_ad_sssd_computer_ou: OU=Linux,DC=ad,DC=example,DC=com
```

### Native SSSD options and short login names

Preserve the standard policies while customizing native options.

```yaml
samba_ad_sssd_sssd_options:
  domain_resolution_order: ad.example.com
samba_ad_sssd_domain_options:
  access_provider: ad
  ad_gpo_access_control: enforcing
  enumerate: false
  cache_credentials: true
  use_fully_qualified_names: false
  fallback_homedir: /home/%d/%u
  default_shell: /bin/bash
samba_ad_sssd_pam_options:
  offline_credentials_expiration: 7
samba_ad_sssd_authselect_features:
  - with-faillock
```

### Optional autorid compatibility

Pin the primary domain SID; this does not promise matching Winbind IDs.

```yaml
samba_ad_sssd_id_mapping: autorid_compat
samba_ad_sssd_domain_options:
  access_provider: ad
  ad_gpo_access_control: enforcing
  enumerate: false
  cache_credentials: true
  use_fully_qualified_names: true
  fallback_homedir: /home/%d/%u
  default_shell: /bin/bash
  ldap_idmap_range_min: 200000
  ldap_idmap_range_max: 2000200000
  ldap_idmap_range_size: 200000
  ldap_idmap_default_domain_sid: S-1-5-21-111111111-222222222-333333333
```

### Explicit domain controller endpoints

Select runtime LDAP and Kerberos endpoints when DNS SRV discovery is unavailable.
Include these keys in the desired domain_options dictionary.

```yaml
samba_ad_sssd_domain_options:
  ad_server: dc1.ad.example.com, dc2.ad.example.com
  krb5_server: dc1.ad.example.com, dc2.ad.example.com
  access_provider: ad
  ad_gpo_access_control: enforcing
  use_fully_qualified_names: true
  cache_credentials: true
  enumerate: false
  fallback_homedir: /home/%d/%u
  default_shell: /bin/bash
```

### Computer policies with manual refresh

Keep Linux GPO support while using an external scheduler or manual refresh.

```yaml
samba_ad_sssd_gpo_workgroup: EXAMPLE
samba_ad_sssd_gpo_refresh_enabled: false
```

### Computer policy refresh interval

Customize the interval and random delay for computer policy refresh.

```yaml
samba_ad_sssd_gpo_refresh_interval: 60min
samba_ad_sssd_gpo_randomized_delay: 15min
```

## References

- [Samba Linux group policy client](https://github.com/samba-team/samba/blob/master/source4/scripting/bin/samba-gpupdate)
- [Fedora oddjob-gpupdate](https://packages.fedoraproject.org/pkgs/oddjob-gpupdate/oddjob-gpupdate/index.html)
- [openSUSE oddjob-gpupdate](https://github.com/openSUSE/oddjob-gpupdate)
- [SSSD machine password renewal](https://github.com/SSSD/sssd/blob/master/src/providers/ad/ad_machine_pw_renewal.c)
- [SSSD enumeration lifecycle](https://sssd.io/release-notes/sssd-2.12.0.html)
- [SSSD AD provider](https://sssd.io/docs/ad/ad-provider.html)
- [SSSD ID mapping](https://github.com/SSSD/sssd/blob/master/src/man/include/ldap_id_mapping.xml)
- [Samba collection](https://github.com/jomrr/ansible-collection-samba)

## Author

[Jonas Mauer](https://github.com/jomrr)

## License

This project is licensed under the MIT License.
See [LICENSE](LICENSE) for the full license text.

Copyright (c) 2026 Jonas Mauer.
