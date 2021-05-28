# Borg backup role
This role installs Borg backup on borgbackup\_servers and clients. The role contains a wrapper-script 'borg-backup' to ease the usage on the client. Supported options include borg-backup info | init | list | backup | mount. Automysqlbackup will run as pre-backup command if it's installed.
The role supports both self hosted and offsite backup-storage such as rsync.net and hetzner storage box as Borg server.

It's possible to configure append-only repositories to secure the backups against deletion from the client.

Ansible 2.4 is required to run this role.

## Required variables
Define a group borgbackup\_servers in your inventory with one or multiple hosts. The group borgbackup\_management is only necessary if you want to enable append-only mode and prune the backups from a secured hosts.
```
[borgbackup_servers]
backup1.fiaas.co

[borgbackup_management]
supersecurehost
```

Define group- or hostvars for your backup endpoints and retention (e.g. the following could live in `hosts/all`):

Here is an example where each ansible hosts stores backups without encryption under `borgbackup.server` with the user the same as the inventory name, and a home dir under /backup. It also installs (offline) a specific borgbackup version (useful for older hosts):
```
borgbackup_version: 1.1.0
borgbackup_encryption_mode: none
borgbackup_include:
  - /etc
  - /var/spool/cron
borgbackup_exclude: []
borgbackup_pre_commands: []
borgbackup_post_commands: []
borgbackup_servers:
  - fqdn: borgbackup.server
    shortname: borgbackup
    user: "{{inventory_hostname}}"
    type: ssh
    home: "/backup/{{inventory_hostname}}/"
    pool: "{{inventory_hostname}}"
    options: ""
borgbackup_passphrase: ""
borgbackup_retention:
  hourly: -1
  daily: 7
  weekly: 4
  monthly: 6
  yearly: -1
```
Simpler example, with a single user/repo:
```
borgbackup_servers:
  - fqdn: backup1.fiaas.co
    user: borgbackup
    type: normal
    home: /backup/
    pool: repos
    options: ""
  - fqdn: yourhost.rsync.net
    user: userid
    type: rsync.net
    home: ""
    pool: repos
    options: "--remote-path=borg1"
  - fqdn: username.your-storagebox.de
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

Set borgbackup\_appendonly: True in host or group vars if you want append-only repositories. In that case it's possible to define a hostname in borgbackup\_management\_station where a borg prune script will be configured. Only the management station will have permission to prune old backups for (all) clients.

*Make sure to check the configured defaults for this role, which contains the list of default locations being backed up in backup_include.* Override this in your inventory where required.

To back up a local mysql instance you can add:
```
mysqluser: root
mysqlpass: mysecretmysqlpassword
mysqlskip: 
  - appdb.unimportant_table
  - otherdb.big_table_with_logs
```

If your mysql root account doesn't have a password, change `mysqlpass: ""`. mysqlskip is a list of database.table names to skip from the backup. 

## Usage

Configure Borg on the server and on a client:
```
ansible-playbook -i inventory/test backup.yml -l backup1.fiaas.co
ansible-playbook -i inventory/test backup.yml -l client1.fiaas.co
```

## Further reading
* https://borgbackup.readthedocs.io/en/stable/
