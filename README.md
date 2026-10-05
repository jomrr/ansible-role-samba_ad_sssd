# Ansible Role: sssd

![GitHub](https://img.shields.io/github/license/jomrr/ansible-role-sssd)
![GitHub last commit](https://img.shields.io/github/last-commit/jomrr/ansible-role-sssd)
![GitHub issues](https://img.shields.io/github/issues-raw/jomrr/ansible-role-sssd)
[![dev](https://img.shields.io/github/actions/workflow/status/jomrr/ansible-role-sssd/dev.yml?branch=dev&label=dev)](https://github.com/jomrr/ansible-role-sssd/actions/workflows/dev.yml?query=branch%3Adev)
[![main](https://img.shields.io/github/actions/workflow/status/jomrr/ansible-role-sssd/main.yml?branch=main&label=main)](https://github.com/jomrr/ansible-role-sssd/actions/workflows/main.yml?query=branch%3Amain)

Join Linux systems to Active Directory and configure SSSD with centrally
assigned RFC2307 identities.

## Purpose

Establish AD account logins on Linux servers and workstations with SSSD. The
default reads centrally assigned RFC2307 UID/GID attributes from AD and enforces
GPO login access rules through SSSD.

## Scope

### Managed

- AD join and machine keytab through adcli, with a local klist check for a
  machine principal in the configured realm.
- SSSD configuration, including its NSS and PAM responder options and AD access
  policy.
- Enabled and running SSSD.

### Not Managed

- System Kerberos configuration; run jomrr.krb5 before this role.
- System PAM stacks, passwd/group NSS selection and home creation; managed by
  jomrr.pam.
- LDAP and IPA identity providers. Only the AD provider is supported.
- Samba file shares, smbd, winbind, and file ownership migrations.
- DC provisioning, RFC2307 attribute allocation, DNS resolver setup, time
  synchronization, and host naming.
- Domain leave, automatic rejoin after trust failures, and authoring or linking
  AD policies.
- Samba gpupdate client configuration and application of Linux computer or user
  policies.

## Requirements

- Run jomrr.krb5 before this role with krb5_realm matching sssd_realm and DNS
  KDC discovery enabled.
- Run jomrr.pam with pam_provider=sssd before this role to select native PAM and
  NSS integration.
- A stable host FQDN, working AD DNS discovery, synchronized time, and network
  reachability to the domain.
- For RFC2307: users need uidNumber and gidNumber, groups need gidNumber;
  allocate these centrally and consistently. Provisioning the schema alone does
  not assign them.
- Applications must use the system PAM stack. SSH password logins also require
  suitable sshd configuration managed outside this role.

## Dependencies

```yaml
collections:
  - name: community.general
    version: '>=12.0.0'
  - name: jomrr.samba
    version: '>=2.0.0'
roles:
  - name: jomrr.krb5
    src: https://github.com/jomrr/ansible-role-krb5.git
    scm: git
    version: main
  - name: jomrr.pam
    src: https://github.com/jomrr/ansible-role-pam.git
    scm: git
    version: main
```

## Role Variables

### `sssd_realm`

Type: `str`. Required: `true`.

AD DNS domain and Kerberos realm; must remain stable after joining.

### `sssd_join_username`

Type: `str`. Required: `false`.

Account delegated permission to join computers to the domain.

Default:

```yaml
sssd_join_username: Administrator
```

### `sssd_join_password`

Type: `str`. Required: `false`.

Join password from a secret store; needed for the initial join or forced rejoin.

### `sssd_server`

Type: `str`. Required: `false`.

DC hostname for the initial join; empty uses adcli discovery. Runtime DC
selection uses domain_options.ad_server.

Default:

```yaml
sssd_server: ''
```

### `sssd_computer_ou`

Type: `str`. Required: `false`.

LDAP DN of the computer OU; empty uses the domain default.

Default:

```yaml
sssd_computer_ou: ''
```

### `sssd_host_fqdn`

Type: `str`. Required: `false`.

Stable fully qualified machine hostname shared by adcli and SSSD.

Default:

```yaml
sssd_host_fqdn: '{{ ansible_facts.fqdn | lower }}'
```

### `sssd_keytab`

Type: `path`. Required: `false`.

Machine keytab used explicitly by adcli and SSSD; its parent directory must
exist.

Default:

```yaml
sssd_keytab: /etc/krb5.keytab
```

### `sssd_force_join`

Type: `bool`. Required: `false`.

Rejoin on every run; enable only for deliberate machine account repair.

Default:

```yaml
sssd_force_join: false
```

### `sssd_id_mapping`

Type: `str`. Required: `false`.

rfc2307 reads centrally assigned POSIX IDs; autorid_compat enables SSSD
algorithmic mapping with autorid compatibility.

Default:

```yaml
sssd_id_mapping: rfc2307
```

### `sssd_sssd_options`

Type: `dict`. Required: `false`.

Native [sssd] options, for example domain_resolution_order. Domains and services
are role-owned.

Default:

```yaml
sssd_sssd_options: {}
```

### `sssd_domain_options`

Type: `dict`. Required: `false`.

Native AD domain options with scalar values and comma-separated strings for
lists. Role-owned provider, identity, keytab and mapping settings take
precedence.

Default:

```yaml
sssd_domain_options:
  access_provider: ad
  ad_gpo_access_control: enforcing
  enumerate: false
  cache_credentials: true
  use_fully_qualified_names: true
  fallback_homedir: /home/%d/%u
  default_shell: /bin/bash
```

### `sssd_nss_options`

Type: `dict`. Required: `false`.

Native [nss] responder options with scalar values, for example filter_users or
filter_groups.

Default:

```yaml
sssd_nss_options: {}
```

### `sssd_pam_options`

Type: `dict`. Required: `false`.

Native [pam] responder options with scalar values, for example
offline_credentials_expiration.

Default:

```yaml
sssd_pam_options: {}
```

### `sssd_config_no_log`

Type: `bool`. Required: `false`.

Redact template output and diffs when native SSSD options contain secrets.

Default:

```yaml
sssd_config_no_log: false
```

## Managed Files

- `/etc/sssd/sssd.conf (0640, root-owned, native SSSD service group, native
  validation and backup).`

## Check Mode

Supported on an already prepared host; klist reads the keytab in check mode and
the adcli join task is skipped.

- A first run in check mode cannot install required tools or create a usable
  machine keytab. Run a normal converge before checking a fresh host.

## Service Behavior

SSSD configuration and keytab changes restart SSSD.

### Handlers

- restart sssd

## Security Notes

- Keep join credentials in Ansible Vault or a secret store. The join task is
  redacted. Delegate computer join permissions to a dedicated account instead of
  using a domain administrator in production.
- Set config_no_log when native SSSD options contain secret values.
- SSSD's AD provider uses Kerberos authentication and GSSAPI-protected LDAP. Its
  access policy remains configured through domain_options.
- The default AD access provider enforces GPO access rules. Successful identity
  lookup alone does not grant login or sudo privileges.
- Reserve centrally assigned RFC2307 UID/GID values against local accounts and
  system IDs.
- Machine credentials and their backups need restricted access. The role
  protects SSSD configuration with mode 0640 for root and the native service
  group. Protect the machine keytab and SSSD credential cache when backing up or
  restoring the host.
- Offline credential caching remains enabled for workstation logins; configure
  its expiry through pam_options.offline_credentials_expiration according to
  local policy.
- Authentication logging follows native PAM/SSSD facilities; collection and
  retention, firewalls, storage encryption, DNS and time synchronization remain
  host or site responsibilities.

## Operational Notes

- jomrr.krb5 owns /etc/krb5.conf and /etc/krb5.conf.d. This role sets
  krb5_keytab, ldap_krb5_keytab and adcli's host-keytab explicitly from
  sssd_keytab, independently of the library default keytab. Setting
  krb5_default_keytab is therefore unnecessary for SSSD.
- Native PAM and passwd/group NSS ownership belongs to jomrr.pam. Its
  pam_mkhomedir, pam_authselect_features and pam_authselect_force variables
  control native integration. sssd_pam_options and sssd_nss_options configure
  SSSD responders.
- sssd_id_mapping=rfc2307 sets id_provider=ad and ldap_id_mapping=false.
  autorid_compat sets ldap_id_mapping=true and ldap_idmap_autorid_compat=true.
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
  krb5_realm, ad_hostname, both keytab paths, and the two mapping booleans.
  Other native options remain configurable.
- Existing /etc/sssd/conf.d snippets are included in validation and preserved.
  They must not override the role-owned identity settings.
- SSSD and winbind must not compete as identity providers for this domain. This
  role targets dedicated SSSD clients and does not migrate an existing winbind
  deployment.
- Join idempotency checks sssd_keytab with klist -k for a host/...@REALM or
  NAME$@REALM machine principal. Realm matching is case-insensitive. A missing
  keytab or absent machine principal triggers adcli join; an existing matching
  principal skips it without contacting a DC. Use sssd_force_join only for
  deliberate repair. sssd_join_password is needed only when a join runs.
- jomrr.samba is used only by the Molecule AD controller fixture. Role tasks use
  klist and adcli directly.
- AD user/group enumeration is deprecated since SSSD 2.10 and its extended
  support was removed in 2.12. enumerate is passed through for builds that
  support it; full listings cannot be guaranteed on current distributions. Named
  lookups and logins do not require enumeration.
- The integration fixture uses explicit domain_options.ad_server and krb5_server
  endpoints: current Fedora/openSUSE c-ares resolvers rejected the Samba test DC
  SRV responses with Misformatted DNS reply. Automatic discovery requires DNS
  responses accepted by the installed resolver. No DNS parser or security policy
  is bypassed by the role.
- SSSD ad_gpo_access_control evaluates AD login access rights. Applying other
  Linux policies through gpupdate is outside the current role scope. Login
  integration is deferred until authselect 1.8.0 with its native with-gpupdate
  feature and the required oddjob-gpupdate components are available on the
  target platforms.

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
    - role: jomrr.krb5
      krb5_realm: AD.EXAMPLE.COM
    - role: jomrr.pam
      pam_provider: sssd
    - role: jomrr.sssd
      sssd_realm: AD.EXAMPLE.COM
      sssd_join_password: "{{ vault_ad_join_password }}"
      sssd_computer_ou: OU=Linux,DC=ad,DC=example,DC=com
```

### Native SSSD options and short login names

Preserve the standard policies while customizing native options.

```yaml
sssd_sssd_options:
  domain_resolution_order: ad.example.com
sssd_domain_options:
  access_provider: ad
  ad_gpo_access_control: enforcing
  enumerate: false
  cache_credentials: true
  use_fully_qualified_names: false
  fallback_homedir: /home/%d/%u
  default_shell: /bin/bash
sssd_pam_options:
  offline_credentials_expiration: 7
pam_authselect_features:
  - without-nullok
  - with-faillock
```

### Optional autorid compatibility

Pin the primary domain SID; this does not promise matching Winbind IDs.

```yaml
sssd_id_mapping: autorid_compat
sssd_domain_options:
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
sssd_domain_options:
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

## References

- [MIT Kerberos configuration](https://web.mit.edu/kerberos/krb5-latest/doc/admin/conf_files/krb5_conf.html)
- [SSSD AD provider manual](https://github.com/SSSD/sssd/blob/master/src/man/sssd-ad.5.xml)
- [authselect 1.8.0 login integration](https://github.com/authselect/authselect/blob/1.8.0/profiles/sssd/system-auth)
- [SSSD enumeration lifecycle](https://sssd.io/release-notes/sssd-2.12.0.html)
- [SSSD AD provider](https://sssd.io/docs/ad/ad-provider.html)
- [SSSD ID mapping](https://github.com/SSSD/sssd/blob/master/src/man/include/ldap_id_mapping.xml)
- [adcli manual](https://www.freedesktop.org/software/realmd/adcli/adcli.html)

## Author

[Jonas Mauer](https://github.com/jomrr)

## License

This project is licensed under the MIT License.
See [LICENSE](LICENSE) for the full license text.

Copyright (c) 2026 Jonas Mauer.
