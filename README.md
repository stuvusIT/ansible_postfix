# ansible_postfix

This role sets up a Debian-based system for use as a mailserver by installing and configuring
Postfix.
Additionally it can install and configure OpenDKIM for DKIM signing/verification and saslauthd for
complex authentication scenarios involving users provisioned to a LDAP directory.

## Requirements

This role assumes the presence of necessary DNS records (MX, SPF and DKIM) for email delivery.
It is developed and tested on a plain Debian install, but may work for Debian derivatives.

## Role Variables

| Name                                  |     Required/Default      | Description                                                                       |
| ------------------------------------- | :-----------------------: | --------------------------------------------------------------------------------- |
| `postfix_configs`                     |           `{}`            | Postfix configuration files, see [#postfix-configuration]                         |
| `postfix_opendkim_additional_configs` |           `{}`            | OpenDKIM additional configuration files, see [#opendkim-additional-configuration] |
| `postfix_opendkim_config`             | See `./defaults/main.yml` | OpenDKIM configuration [#opendkim]                                                |
| `postfix_opendkim_enabled`            |          `False`          | Enable OpenDKIM daemon, see [#opendkim]                                           |
| `postfix_opendkim_keys`               |           `{}`            | OpenDKIM private keys, see [#dkim-private-keys]                                   |
| `postfix_sasl_enabled`                |          `False`          | Enable saslauthd daemon, see [#saslauthd]                                         |
| `postfix_saslauthd_options`           |     See [#saslauthd]      | saslauthd configuration options, see [#saslauthd]                                 |

## Postfix Configuration

The `postfix_configs` variable is used to configure Postfix configuration files under
`/etc/postfix/`.
The two most important configuration files are `main.cf` and `master.cf` (see Postfix
documentation), but `postfix_configs` can specify arbitrary additional config files.
Example:

```yml
postfix_configs:
  main: |
    # Content of /etc/postfix/main.cf goes here
  master: |
    # Content of /etc/postfix/main.cf goes here
  arbitrary-additional-config: |
    # Content of /etc/postfix/arbitrary-additional-config.cf goes here
```

As displayed in the example, `postfix_configs` is a dict that maps file names to file contents.
Those files are placed in `/etc/postfix/` and file names are extended by `.cf`.

Any surplus config files `/etc/postfix/*.cf` are removed by this role, except for the following
files:

- `main.cf`
- `master.cf`
- `dynamicmaps.cf`

These are left untouched in case they are missing from `postfix_configs`, in order to allow using
distribution-supplied defaults for these files.

## saslauthd

If you need to authenticate users against an LDAP directory, this roles can install and enable
saslauthd.
This can be enabled by setting `postfix_sasl_enabled: True`.

If you need bind credentials or other additional configuration options, you need to set
`postfix_saslauthd_options`.
This dict will be parsed and written to the saslauthd config file.
The default is:

```yml
postfix_saslauthd_options:
  ldap_servers: "{{ postfix_ldap_servers }}"
  ldap_search_base: "{{ postfix_ldap_search_base }}"
  ldap_auth_method: bind
```

## OpenDKIM

If you want to add DKIM signing/checking, this role can install and configure OpenDKIM.
This can be enabled by setting `postfix_opendkim_enabled: True`.

The default config shown below, sets OpenDKIM up in verifier mode, listening at `localhost:12301`:

```
Syslog                  yes
SyslogSuccess           yes
Mode                    v

# In Debian, opendkim runs as user "opendkim". A umask of 007 is required when
# using a local socket with MTAs that access the socket as a non-privileged
# user (for example, Postfix). You may need to add user "postfix" to group
# "opendkim" in that case.
UserID                  opendkim
UMask                   007

# Socket for the MTA connection (required).
Socket                  inet:12301@localhost
PidFile                 /run/opendkim/opendkim.pid

# The trust anchor enables DNSSEC. In Debian, the trust anchor file is provided
# by the package dns-root-data.
TrustAnchorFile         /usr/share/dns/root.key
```

If you have different requirements, you can simply override the config by setting
`postfix_opendkim_config`.

### OpenDKIM Additional Configuration

The `postfix_opendkim_additional_configs` variable is used to configure additional OpenDKIM
configuration files under `/etc/opendkim/`.
It works identically to `postfix_configs`, i.e., it maps file names to file contents and file names
are extended by `.cf`.
Any surplus config files `/etc/opendkim/*.cf` are removed by this role (without exception).

The main reason to have this separated from `postfix_configs` is that changes in `postfix_configs`
trigger a restart of the Postfix daemon while changes in `postfix_opendkim_additional_configs`
instead trigger a restart of the OpenDKIM daemon.

### DKIM Private Keys

The `postfix_opendkim_keys` variable can be used to copy DKIM private keys to the host.
This variable takes is a dict where each key-value pair describes a private key.
For each key, the value is simply passed to the Ansible
[copy module](https://docs.ansible.com/ansible/latest/modules/copy_module.html), and the file is
placed at the destination `/etc/opendkim/keys/{{ key }}.private` with appropriate permissions for
OpenDKIM.

For example, the following configuration places a DKIM private key under
`/etc/opendkim/keys/my-key.private`:

```yml
postfix_opendkim_keys:
  my-key:
    content: |
      -----BEGIN RSA PRIVATE KEY-----
      [...]
      -----END RSA PRIVATE KEY-----
```

## License

This work is licensed under the [MIT License](./LICENSE).

## Author Information

- [Sven Feyerabend (SF2311)](https://github.com/SF2311) _sven.feyerabend at stuvus.uni-stuttgart.de_
