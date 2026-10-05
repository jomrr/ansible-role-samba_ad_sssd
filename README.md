# Ansible Role: samba_ad_sssd

![GitHub](https://img.shields.io/github/license/jomrr/ansible-role-samba_ad_sssd)
![GitHub last commit](https://img.shields.io/github/last-commit/jomrr/ansible-role-samba_ad_sssd)
![GitHub issues](https://img.shields.io/github/issues-raw/jomrr/ansible-role-samba_ad_sssd)
[![dev](https://img.shields.io/github/actions/workflow/status/jomrr/ansible-role-samba_ad_sssd/dev.yml?branch=dev&label=dev)](https://github.com/jomrr/ansible-role-samba_ad_sssd/actions/workflows/dev.yml?query=branch%3Adev)
[![main](https://img.shields.io/github/actions/workflow/status/jomrr/ansible-role-samba_ad_sssd/main.yml?branch=main&label=main)](https://github.com/jomrr/ansible-role-samba_ad_sssd/actions/workflows/main.yml?query=branch%3Amain)

Join Linux systems to Active Directory with SSSD, RFC2307 identities, and native
PAM integration.

## Purpose

Establish AD account logins on Linux servers and workstations with SSSD. The
default reads centrally assigned RFC2307 UID/GID attributes from AD and enforces
GPO login access rules through SSSD.

## Scope

### Managed

- AD join and machine keytab through jomrr.samba.samba_join_sssd (adcli).
- System Kerberos configuration with the AD default realm, DNS KDC discovery,
  domain mapping and machine keytab path.
- SSSD configuration, NSS identity lookup, PAM authentication and optional home
  creation.
- Enabled and running SSSD, with oddjob on Red Hat systems for automatic home
  creation.

### Not Managed

- Samba file shares, smbd, winbind, and file ownership migrations.
- DC provisioning, RFC2307 attribute allocation, DNS resolver setup, time
  synchronization, and host naming.
- Domain leave, automatic rejoin after trust failures, and authoring or linking
  AD policies.
- Samba gpupdate client configuration and application of Linux computer or user
  policies.

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

Join password from a secret store; needed for the initial join or forced rejoin.

### `samba_ad_sssd_server`

Type: `str`. Required: `false`.

DC hostname for the initial join; empty uses adcli discovery. Runtime DC
selection uses domain_options.ad_server.

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

Machine keytab used by adcli, SSSD and the system Kerberos configuration; its
parent directory must exist.

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
samba_ad_sssd_authselect_features:
  - without-nullok
  - with-faillock
```

### `samba_ad_sssd_authselect_force`

Type: `bool`. Required: `false`.

Allow authselect to replace existing unmanaged PAM/NSS files using its native
backup mechanism.

Default:

```yaml
samba_ad_sssd_authselect_force: false
```

## Managed Files

- `/etc/krb5.conf (complete file, mode 0644, root-owned, previous version backed
  up), with /etc/krb5.conf.d snippets retained.`
- `/etc/sssd/sssd.conf (0640, root-owned, native SSSD service group, native
  validation and backup).`
- `/etc/nsswitch.conf and native PAM profile selection; PAM files are generated
  by distribution tools.`

## Check Mode

Supported on an already prepared host; adcli reports a pending join without
performing it.

- A first run in check mode cannot install required tools or create a usable
  machine keytab. Run a normal converge before checking a fresh host.

## Service Behavior

Configuration and keytab changes restart SSSD before native PAM integration.

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
- On Red Hat systems, authselect enables without-nullok and with-faillock by
  default. These remove pam_unix's empty-password allowance and enable PAM
  failure lockouts. Lockout thresholds and duration follow
  /etc/security/faillock.conf and distribution defaults; the role does not
  manage that file. Coordinate local lockouts with AD policy; local_users_only
  can restrict faillock to local accounts.
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

- The role writes /etc/krb5.conf before joining. It sets the default realm and
  keytab from samba_ad_sssd_realm and samba_ad_sssd_keytab, discovers KDCs
  through DNS and maps the AD DNS domain and its subdomains to the realm. DNS
  realm lookup and reverse/canonical hostname rewriting are disabled. Existing
  settings in the main file are replaced; retain site-specific configuration in
  /etc/krb5.conf.d without conflicting with these settings.
- Kerberos credential caches use FILE on Debian/Ubuntu and persistent KEYRING on
  Red Hat/openSUSE, matching samba_ad_member. Native Kerberos libraries and
  distribution crypto-policy snippets determine encryption types. There is no
  standalone Kerberos candidate-file validator in the installed client tools;
  functional joins and authentication are covered by the integration scenario.
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
- samba_ad_sssd_authselect_features selects additional features of the native
  Red Hat sssd profile. An override replaces the list; include without-nullok
  and with-faillock to retain the defaults. An empty list disables these
  additional features. Home creation remains controlled by
  samba_ad_sssd_mkhomedir, which adds or removes with-mkhomedir independently.
  These authselect flags do not affect Debian/Ubuntu or openSUSE PAM
  configuration.
- Check authselect list-features sssd and authselect requirements sssd with the
  desired feature names on the target. Available features depend on the
  installed profile. The feature list selects PAM/NSS integration; optional
  modules, hardware enrollment and supporting services require separate
  provisioning.
- with-fingerprint needs pam_fprintd and an enrolled fingerprint reader.
  with-smartcard requires trusted certificates and SSSD certificate
  authentication, including pam_options.pam_cert_auth=true;
  with-smartcard-required enforces smartcard authentication and
  with-smartcard-lock-on-removal adds desktop locking when a card is removed.
  with-gssapi requires pam_sss_gss and allowed pam_options.pam_gssapi_services.
- with-pam-u2f enables U2F authentication and with-pam-u2f-2fa enables it as a
  second factor. Both need pam_u2f and enrolled keys. without-pam-u2f-nouserok
  makes enrollment mandatory with the second-factor feature, including for root.
  with-pam-gnome-keyring needs the GNOME keyring PAM module and session
  integration.
- with-files-access-provider subjects local regular users to SSSD access checks
  and requires a suitable SSSD local-user domain outside this role's AD
  configuration. with-pwhistory enables local password history; it does not
  configure AD password policy.
- with-sudo adds SSSD as a sudo rule source; it requires a configured sudo
  provider, responder and directory rules. with-subid adds SSSD as a
  subordinate-ID source and needs a compatible provider. These flags alone do
  not configure either service. with-libvirt requires libvirt NSS modules for
  guest hostname resolution.
- with-silent-lastlog suppresses the last-login notice;
  without-lastlog-showfailed suppresses the failed-login count. Both depend on
  support in the installed authselect profile.
- The role owns domains, services, id_provider, auth_provider, ad_domain,
  krb5_realm, ad_hostname, both keytab paths, and the two mapping booleans.
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
  - without-nullok
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

## References

- [authselect SSSD features](https://github.com/authselect/authselect/blob/master/profiles/sssd/README)
- [authselect feature requirements](https://github.com/authselect/authselect/blob/master/profiles/sssd/REQUIREMENTS)
- [PAM lockout configuration](https://github.com/linux-pam/linux-pam/blob/master/modules/pam_faillock/faillock.conf.5.xml)
- [MIT Kerberos configuration](https://web.mit.edu/kerberos/krb5-latest/doc/admin/conf_files/krb5_conf.html)
- [SSSD AD provider manual](https://github.com/SSSD/sssd/blob/master/src/man/sssd-ad.5.xml)
- [authselect 1.8.0 login integration](https://github.com/authselect/authselect/blob/1.8.0/profiles/sssd/system-auth)
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
