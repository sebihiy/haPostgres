## Valentine-Infra - PostgreSQL Replication on HA  [![Build Status](https://travis-ci.org/ANXS/postgresql.svg?branch=master)](https://travis-ci.org/ANXS/postgresql)

#### Ansible Playbook

The main.yaml playbook used the role postgresql. 

```
$ cat main.yaml
- hosts: postgres_cluster
  become: yes
  vars_files:
    - vars/custom.yml
  roles:
    - postgresql
```

To test playbook we have added host group postgres_cluster on inventory.yaml

```
$ cat inventory.yaml
all:
  children:
    postgres_cluster:
      hosts:
        PGS01:
        PGS02:
        PGS03:
      vars:
        ansible_user: "admin"
```

#### Custom Variable File

In the custom variable file vars/custom.yaml, we will define the following things:

  - PostgreSQL version to install and encoding to use
  - Modifying the PostgreSQL configuration to enable replication, we will modify the parameters like wal_level, max_wal_senders,max_replication_slots, hot_standby, archive_mode, archive_command.
  - Creating the necessary users and database.
  - Modifying pg_hba.conf file to allow the necessary connection from the application and the repmgr replication
  - Some repmgr related variables

```
$ cat vars/custom.yaml
# Basic settings
postgresql_version: 11
postgresql_encoding: "UTF-8"
postgresql_locale: "fr_FR.UTF-8"
postgresql_ctype: "fr_FR.UTF-8"
postgresql_admin_user: "postgres"
postgresql_default_auth_method: "peer"
postgresql_service_enabled: false # should the service be enabled, default is true
postgresql_cluster_name: "main"
postgresql_cluster_reset: false
postgresql_listen_addresses: "*"
postgresql_wal_level: "replica"
postgresql_max_wal_senders: 10
postgresql_max_replication_slots: 10
postgresql_wal_keep_segments: 100
postgresql_hot_standby: on
postgresql_archive_mode: on
postgresql_archive_command: "/bin/true"
postgresql_shared_preload_libraries:
  - repmgr

## access postgres cluster  pg_hba 
postgresql_pg_hba_custom:
  - { type: "host", database: "replication", user: "repmgr", address: "192.168.1.235/32", method: "trust" }
  - { type: "host", database: "replication", user: "repmgr", address: "192.168.1.236/32", method: "trust" }
  - { type: "host", database: "replication", user: "repmgr", address: "192.168.1.237/32", method: "trust" }
  - { type: "host", database: "replication", user: "repmgr", address: "127.0.0.1/32", method: "md5" }
  - { type: "host", database: "repmgr", user: "repmgr", address: "192.168.1.235/32", method: "trust" }
  - { type: "host", database: "repmgr", user: "repmgr", address: "192.168.1.236/32", method: "trust" }
  - { type: "host", database: "repmgr", user: "repmgr", address: "192.168.1.237/32", method: "trust" }
  - { type: "host", database: "all", user: "chkuser", address: "127.0.0.1/32", method: "md5" }
  - { type: "host", database: "all", user: "chkuser", address: "::1/128", method: "md5" }
  - { type: "host", database: "all", user: "chkuser", address: "localhost", method: "md5" }
  - { type: "host", database: "all", user: "chkuser", address: "{{ ansible_default_ipv4.address }}/32", method: "md5" }

# Manage replication with repmgr (optional)
postgresql_ext_install_repmgr: yes
repmgr_target_group: "postgres_cluster"
repmgr_user: repmgr
repmgr_database: repmgr
check_user: chkuser

postgresql_users:
  - name: "{{repmgr_user}}"
    pass: "pass"
  - name: "{{check_user}}"
    pass: "pass"
    encrypted: yes  # if password should be encrypted, postgresql >= 10 does only accepts encrypted passwords

postgresql_databases:
  - name: "{{repmgr_database}}"
    owner: "{{repmgr_user}}"
    encoding: "UTF-8"

postgresql_user_privileges:
  - name: "{{repmgr_user}}"
    db: "{{repmgr_database}}"
    priv: "ALL"
    role_attr_flags: "SUPERUSER,REPLICATION,CREATEROLE,CREATEDB"
  - name: "{{check_user}}"
    db: "postgres"
    priv: "ALL"
    role_attr_flags: "SUPERUSER,CREATEROLE,CREATEDB"

```

#### Running Ansible Playbook

We have now created the following Ansible artifacts and we are ready to run the Ansible playbook
  - roles/requirements.yml, to push postgresql role in project.
  - vars/custom.yaml, Ansible variable file.
  - inventory.yaml, Ansible inventory file.
  - main.yaml, Ansible playbook file

```

$ ansible-playbook -Ki inventory.yaml  main.yaml
TASK [postgresql : PostgreSQL Check  | Add Script] *******************************************************************************************************************************************************************************************
ok: [PGS03]
ok: [PGS02]
ok: [PGS01]

TASK [postgresql : PostgresSQL Check | add port for postgreschk] *****************************************************************************************************************************************************************************
changed: [PGS03]
changed: [PGS01]
changed: [PGS02]

TASK [postgresql : PostgresSQL Check | enable and start xinetd service] **********************************************************************************************************************************************************************
ok: [PGS03]
ok: [PGS02]
ok: [PGS01]

PLAY RECAP ***********************************************************************************************************************************************************************************************************************************
PGS01                      : ok=48   changed=5    unreachable=0    failed=0    skipped=49   rescued=0    ignored=0   
PGS02                      : ok=48   changed=5    unreachable=0    failed=0    skipped=49   rescued=0    ignored=0   
PGS03                      : ok=53   changed=5    unreachable=0    failed=0    skipped=44   rescued=0    ignored=0   

````

#### Feedback, bug-reports, requests, ...

Are [welcome](https://github.com/ANXS/postgresql/issues)!
