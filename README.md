[![Build Status](https://travis-ci.com/FiaasCo/borgbackup.svg?branch=master)](https://travis-ci.com/FiaasCo/borgbackup)

# Borg backup role
This role installs Borg backup on borgbackup\_servers and clients. The role contains a wrapper-script 'borg-backup' to ease the usage on the client. Supported options include borg-backup info | init | list | backup | mount. Automysqlbackup will run as pre-backup command if it's installed.
The role supports both self hosted and offsite backup-storage such as rsync.net and hetzner storage box as Borg server.

It's possible to configure append-only repositories to secure the backups against deletion from the client.

Ansible 2.4 or higher is required to run this role.

## Required variables
Define a group borgbackup\_servers in your inventory with one or multiple hosts. The group borgbackup\_management is only necessary if you want to enable append-only mode and prune the backups from a secured hosts.
```
[borgbackup_servers]
backup1.fiaas.co

[borgbackup_management]
supersecurehost
```

Define group- or hostvars for your backup endpoints and retention:
```
borgbackup_servers:
  - id: fiaas
    fqdn: backup1.fiaas.co
    user: borgbackup
    type: normal
    home: /backup/
    pool: repos
    options: ""
  - id: rsync
    fqdn: yourhost.rsync.net
    user: userid
    type: rsync.net
    home: ""
    pool: repos
    options: "--remote-path=borg1"
  - id: hetzner
    fqdn: username.your-storagebox.de
    user: username
    type: hetzner
    home: ""
    pool: repos
    options: ""


borgbackup_retention:
  hourly: 12
  daily: 7
  weekly: 4
  monthly: 6
  yearly: 1
```
*WARNING: the trailing / in item.home is required.*

Define a borg\_passphrase for every host.
host\_vars\client1:
```
borgbackup_passphrase: Ahl9EiNohr5koosh1Wohs3Shoo3ooZ6p
```

Per default the role creates a cronjob in /etc/cron.d/borg-backup running as root every day on a random hour between 0 and 5am on a random minute. Override the defaults if necessary:
```
borgbackup_client_user: root
borgbackup_cron_day: "*"
borgbackup_cron_minute: "{{ 59|random }}"
borgbackup_cron_hour: "{{ 5|random }}"
```
Override borgbackup\_client\_user where required, for example if you have a laptop with an encrypted homedir you'll have to run the backup as the user of that homedir.

Set borgbackup\_appendonly: True in host or group vars if you want append-only repositories. In that case it's possible to define a hostname in borgbackup\_management\_station where a borg prune script will be configured. Only the management station will have permission to prune old backups for (all) clients. This will generate serve with --append-only ssh key options.
If you set borgbackup\_appendonly\_repoconfig to True, this will also disable the possibility to remove backups from the management station. (Or at least: it's not possible to remove them till you reconfigure the repository and this is currently not supported in the prune script)
Be aware of the limitations of append-only mode: [pruned backups appear to be removed, but are only removed in the transaction log till something writes in normal mode to the repository](https://github.com/borgbackup/borg/issues/3504))

*Make sure to check the configured defaults for this role, which contains the list of default locations being backed up in backup\_include.* Override this in your inventory where required.

## Installing Borg from Package
Borg can be installed from a package by setting the variable:
```
borgbackup_install_from_pkg: true
```

On CentOS/RedHat the EPEL repo must be present for this to succeed. To install the EPEL repo using the
[`geerlingguy.repo-epel`](https://galaxy.ansible.com/geerlingguy/repo-epel) role, set:
```
borgbackup_install_epel: true
```

## Usage

Configure Borg on the server and on a client:
```
ansible-playbook -i inventory/test backup.yml -l backup1.fiaas.co
ansible-playbook -i inventory/test backup.yml -l client1.fiaas.co
```

## Further reading
* [Borg documentation](https://borgbackup.readthedocs.io/en/stable/)
* [Append only mode information](http://borgbackup.readthedocs.io/en/stable/usage/notes.html#append-only-mode)
