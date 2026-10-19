SSHD config update
=========

This role can be used to update an sshd config, e. g. early in the
startup process before the actual services runs.

[![Build Status](https://github.com/Rheinwerk/ansible-role-update_sshd_config/actions/workflows/ci.yml/badge.svg)](https://github.com/Rheinwerk/ansible-role-update_sshd_config/actions/workflows/ci.yml)

Notice that it will not start the service and expects the program to be
installed already.

Requirements
------------

The target machine must have scollector installed.

Role Variables
--------------

Every setting is one variable with a default in `defaults/main.yml`; set only the ones that
differ. The defaults are the CenterDevice values and read the consumer's `CD_ENV` contract.

| Variable | Default | Reads |
|---|---|---|
| `_sshd_krl` | KRL from `cd.<account>.static/<env>/ssh_key_revocation_lists/revoked_keys.krl` to `/etc/ssh/revoked_keys.krl`, fetched by `/usr/local/sbin/update-ssh-krl` once and then `hourly` via cron | `CD_ENV.iaas.account`, `CD_ENV.iaas.env` |
| `_sshd_principals` | `%u_principals` files; `cd-admin` gets the six env/account/name/hostgroup principals | `CD_ENV.iaas.env`, `CD_ENV.iaas.account`, `CD_ENV.iaas.name`, `CD_ENV.instance.hostgroup` |

`_sshd_krl` keys: `s3_key`, `filename`, `update_command` (called as `<update_command> <s3_key> <filename>`),
`cron_special_time`. `_sshd_principals` keys: `filename_pattern` (the `AuthorizedPrincipalsFile`),
`mappings` (user to list of principals; every entry becomes a line `<user>_<entry>` next to
`<user>_super`). An empty mapping (`{}`) skips that part of the role.

Until v0.4.0 the role took one `_sshd` dict with the keys `krl` and `principals` instead.

Dependencies
------------

None.


Example Playbook
----------------

Only the principals differ from the defaults here; the KRL settings come from the role.

    - hosts: servers
      vars:
        _sshd_principals:
          filename_pattern: "%u_principals"
          mappings:
            "ec2-user":
              - "mapping1"
              - "mapping2"
      roles:
         - { role: update_sshd_config, tags: [ 'sshd' ] }

License
-------

Please see LICENSE.

Author Information
------------------

Original author is [Daniel Schneller](https://github.com/dschneller) as member of the [Rheinwerk](https://github.com/Rheinwerk) project.
