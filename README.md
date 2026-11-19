SSHD config update
=========

This role can be used to update an sshd config, e. g. early in the
startup process before the actual services runs.

[![Build Status](https://github.com/Rheinwerk/ansible-role-update_sshd_config/actions/workflows/ci.yml/badge.svg)](https://github.com/Rheinwerk/ansible-role-update_sshd_config/actions/workflows/ci.yml)

Requirements
------------

- Debian (bookworm, trixie) with systemd: the role starts the `ssh` unit and installs a systemd timer
- openssh-server installed on the target; the role does not install it, it only edits
  `/etc/ssh/sshd_config` and starts the `ssh` service if it is not running. It does not reload a
  running sshd, so the changes take effect on the next sshd start
- the KRL `update_command` (default `/usr/local/sbin/update-ssh-krl`) present and executable on the
  target, unless `_sshd_krl` is `{}`; the role runs it during the play, so it must exit 0
- ansible-core 2.18 or newer

Role Variables
--------------

Every setting is one variable with a default in `defaults/main.yml`; set only the ones that
differ. The defaults are the CenterDevice values and read the consumer's `CD_ENV` contract.

| Variable | Default | Reads |
|---|---|---|
| `_sshd_krl` | KRL from `cd.<account>.static/<env>/ssh_key_revocation_lists/revoked_keys.krl` to `/etc/ssh/revoked_keys.krl`, fetched by `/usr/local/sbin/update-ssh-krl` once and then `hourly` by the systemd timer `ssh-krl-update.timer` | `CD_ENV.iaas.account`, `CD_ENV.iaas.env` |
| `_sshd_principals` | `%u_principals` files; `cd-admin` gets the six env/account/name/hostgroup principals | `CD_ENV.iaas.env`, `CD_ENV.iaas.account`, `CD_ENV.iaas.name`, `CD_ENV.instance.hostgroup` |

`_sshd_krl` keys: `s3_key`, `filename`, `update_command` (called as `<update_command> <s3_key> <filename>`),
`on_calendar` (a systemd `OnCalendar` expression, e.g. `hourly`). `_sshd_principals` keys:
`filename_pattern` (the `AuthorizedPrincipalsFile` below `/etc/ssh`; the role writes each user's file
under the same name with `%u` replaced by the user name, as sshd expands it), `mappings` (user to
list of principals; every entry becomes a line `<user>_<entry>` next to `<user>_super`). An empty
mapping (`{}`) skips that part of the role.

The role validates both variables before it changes anything. `meta/argument_specs.yml` rejects wrong
types and unknown keys; `tasks/assert.yml` (tag `always`) requires for a non-empty dict:

- `_sshd_krl`: every key set as a string; `filename` and `update_command` absolute paths; `filename`
  without whitespace (it ends up unquoted as `RevokedKeys` in `sshd_config`); `s3_key` and
  `update_command` without control characters. The initial run passes the three values as separate
  arguments, and `ExecStart` quotes each one and escapes `%`, `$`, `"` and `\`, so whitespace and
  systemd specifiers reach the command unchanged.
- `_sshd_principals`: `filename_pattern` a file name below `/etc/ssh` (no slash, no whitespace) with
  exactly one `%u` and no other `%` token (without `%u` all users would share one file, and the role
  does not resolve sshd's other tokens); `mappings` at least one user, user names matching
  `[a-z_][a-z0-9_-]*`, each with a flat list of principal strings

Since v1.0.0 an empty `mappings` with a `filename_pattern` fails instead of setting an
`AuthorizedPrincipalsFile` without any principals file.

Until v1.0.0 the principals files were always written as `/etc/ssh/<user>_principals`, whatever
`filename_pattern` said, so only the default `%u_principals` worked.

Until v0.4.0 the role took one `_sshd` dict with the keys `krl` and `principals` instead.

Until v0.5.0 the KRL update ran from root's crontab (key `cron_special_time`). The role now installs
`ssh-krl-update.service` and `ssh-krl-update.timer` instead; it does not remove an existing cron entry,
so use it on new instances. The service runs with systemd's PATH, which includes
`/usr/local/bin` (cron's PATH is `/usr/bin:/bin`), and a failed update shows up in the journal and in
`systemctl --failed`.

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
