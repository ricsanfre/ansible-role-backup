Ansible Role: Backup to S3 Storage Server using restic
=========

This role install and configure [restic] (https://restic.net/) in a linux server using as storage backend a S3 object storage server (i.e. Minio)
The backup procedure will be executed and scheduled using a systemd service and timer.

Requirements
------------

None

Role Variables
--------------

Available variables are listed below along with default values (see `defaults\main.yaml`)

- Restic version to be installed and wheter to force the installation in case that a previous installation is found

  ```yml
  # force restic install even when it is already installed
  restic_force_install: false
  # restic version
  restic_version: 0.12.1
  ```
- Restic installation details

  Restic UNIX user/group
  ```yml
  restic_group: root
  restic_user: root
  ```
  Restic binary location
  ```yml
  restic_path: /usr/local/bin/restic
  ```
  Restic installation directories to place backup service configuration (`restic_etc_dir`), and CA TLS certificates (`restic_cert_dir`)
  ```yml
  restic_etc_dir: /etc/minio
  minio_ca_dir: "{{ restic_etc_dir }}/ssl"
  ```
  Restic backend S3 repository details (`restic_repository`), and access credentials (`restic_aws_access_key_id` and `restic_aws_secret_access_key`)
  ```yml
  restic_repository: "s3:https://127.0.0.1:9090/restic"
  restic_aws_access_key_id: restic
  restic_aws_secret_access_key: supers1cret0
  ```
  Restic respository password
  ```yml
  restic_password: mysupers1cret0
  ```
  
  Whether to use CA SSL certificate to validate connection to S3 storage (`restic_use_ca_cert`) or not. Needed when self-signed certificates or custom CA certificates are used in S3 server. CA certificate content must be loaded into variable `restic_ca_cert`.
  ```yml
  restic_use_ca_cert: false
  # custom CA certificate content
  restic_ca_cert: ""
  ```
  This can be done using `set_fact` ansible task with a lookup filter. See playbook example below
  ```yml
  - name: Load tls key and cert
    set_fact:
      restic_ca_cert: "{{ lookup('file','certificates/CA.pem') }}"
  ```

- Directories list to back up

  `restic_backup_dirs` is a list of dictionaries. Each item of the list is a directory to  include in the backup
  Each dictionary item has a `path` and an `exclude` property (which defaults to nothing). The `exclude` property is the literal argument passed to restic when executing the backup (example: `restic backup /root --exclude .cache --exclude .local`).

  ```yml
  # restic backup directories
  restic_backups_dirs:
    - path: '/etc'
    - path: '/var/log'
    - path: '/root'
      exclude: '--exclude .cache' 
  ```

### Execute pre and post backup commands

Execute `restic check` command before executing backup
```yml
# Run restic check as pre backup task
restic_check: true
```

Execute `restic forget` and `restic prune` commands after executing the backup for purging old backups
```yml
# Run restic forget as post backup task (restic forget --keep-within <keep_within>)
restic_forget: true
restic_forget_keep_within: 30d

# Run restic prune as post backup tasks
restic_prune: true
```

### Sytemd service and timer

A `restic-backup.service` service will be created with all the parameters defined above. The service is of type `oneshot` and will be triggered periodically with `restic-backup.timer`.

The timer is configurable as follows:

- `restic_systemd_timer_on_calender`: defines the `OnCalendar` directive (`*-*-* 03:00:00`)
- `restic_systemd_timer_randomized_delay_sec`: Delay the timer by a random amount of time between 0 and the specified time value. (`0`)

See the [systemd.timer](https://www.freedesktop.org/software/systemd/man/systemd.timer.html) documentation for more information.

You can see the logs of the backup with `journalctl`.

    journalctl -xefu restic-backup

The backup process can be also triggered manually running the command

    systemctl start restic-backup


Testing
------------

After installation the backup process can be tested following this procedure:

1) Trigger manually the backup process

    systemctl start restic-backup

2) Check logs restic-backup service

    journalctl -u restic-backup
  
   Output should be like:

    ```
    Dec 22 17:01:59 server systemd[1]: Starting Restic backup...
    Dec 22 17:01:59 server bash[1887]: Fatal: unable to open config file: Stat: The specified key does not ex
    ist.
    Dec 22 17:01:59 server bash[1887]: Is there a repository at the following location?
    Dec 22 17:01:59 server bash[1887]: s3:https://10.11.0.1:9091/restic
    Dec 22 17:01:59 server bash[1886]: Repo s3:https://10.11.0.1:9091/restic not initilialized. Initializing
    it ...
    Dec 22 17:02:01 server bash[1896]: created restic repository df54412d4c at s3:https://10.11.0.1:9091/rest
    ic
    Dec 22 17:02:01 server bash[1896]: Please note that knowledge of your password is required to access
    Dec 22 17:02:01 server bash[1896]: the repository. Losing your password means that your data is
    Dec 22 17:02:01 server bash[1896]: irrecoverably lost.
    Dec 22 17:02:01 server restic[1906]: using temporary cache in /tmp/restic-check-cache-052310176
    Dec 22 17:02:02 server restic[1906]: create exclusive lock for repository
    Dec 22 17:02:02 server restic[1906]: load indexes
    Dec 22 17:02:02 server restic[1906]: check all packs
    Dec 22 17:02:02 server restic[1906]: check snapshots, trees and blobs
    Dec 22 17:02:02 server restic[1906]: no errors were found
    Dec 22 17:02:02 server restic[1906]: [0:00]          0 snapshots
    Dec 22 17:02:02 server restic[1916]: open repository
    Dec 22 17:02:02 server restic[1916]: lock repository
    Dec 22 17:02:03 server restic[1916]: load index files
    Dec 22 17:02:03 server restic[1916]: no parent snapshot found, will read all files
    Dec 22 17:02:03 server restic[1916]: start scan on [/etc]
    Dec 22 17:02:03 server restic[1916]: start backup on [/etc]
    Dec 22 17:02:03 server restic[1916]: scan finished in 0.262s: 213 files, 536.395 KiB
    Dec 22 17:02:03 server restic[1916]: Files:         213 new,     0 changed,     0 unmodified
    Dec 22 17:02:03 server restic[1916]: Dirs:          105 new,     0 changed,     0 unmodified
    Dec 22 17:02:03 server restic[1916]: Data Blobs:    198 new
    Dec 22 17:02:03 server restic[1916]: Tree Blobs:     79 new
    Dec 22 17:02:03 server restic[1916]: Added to the repo: 725.919 KiB
    Dec 22 17:02:03 server restic[1916]: processed 213 files, 536.395 KiB in 0:00
    Dec 22 17:02:03 server restic[1916]: snapshot 0047e386 saved
    Dec 22 17:02:03 server restic[1926]: open repository
    Dec 22 17:02:03 server restic[1926]: lock repository
    Dec 22 17:02:03 server restic[1926]: load index files
    Dec 22 17:02:03 server restic[1926]: no parent snapshot found, will read all files
    Dec 22 17:02:03 server restic[1926]: start scan on [/var/log]
    Dec 22 17:02:03 server restic[1926]: start backup on [/var/log]
    Dec 22 17:02:03 server restic[1926]: scan finished in 0.239s: 10 files, 345.640 KiB
    Dec 22 17:02:03 server restic[1926]: Files:          10 new,     0 changed,     0 unmodified
    Dec 22 17:02:03 server restic[1926]: Dirs:            5 new,     0 changed,     0 unmodified
    Dec 22 17:02:03 server restic[1926]: Data Blobs:      8 new
    Dec 22 17:02:03 server restic[1926]: Tree Blobs:      4 new
    Dec 22 17:02:03 server restic[1926]: Added to the repo: 350.619 KiB
    Dec 22 17:02:03 server restic[1926]: processed 10 files, 345.640 KiB in 0:00
    Dec 22 17:02:03 server restic[1926]: snapshot 285d0864 saved
    Dec 22 17:02:03 server restic[1936]: open repository
    Dec 22 17:02:04 server restic[1936]: lock repository
    Dec 22 17:02:04 server restic[1936]: load index files
    Dec 22 17:02:04 server restic[1936]: no parent snapshot found, will read all files
    Dec 22 17:02:04 server restic[1936]: start scan on [/root]
    Dec 22 17:02:04 server restic[1936]: start backup on [/root]
    Dec 22 17:02:04 server restic[1936]: scan finished in 0.233s: 2 files, 3.190 KiB
    Dec 22 17:02:04 server restic[1936]: Files:           2 new,     0 changed,     0 unmodified
    Dec 22 17:02:04 server restic[1936]: Dirs:            3 new,     0 changed,     0 unmodified
    Dec 22 17:02:04 server restic[1936]: Data Blobs:      2 new
    Dec 22 17:02:04 server restic[1936]: Tree Blobs:      3 new
    Dec 22 17:02:04 server restic[1936]: Added to the repo: 4.852 KiB
    Dec 22 17:02:04 server restic[1936]: processed 2 files, 3.190 KiB in 0:00
    Dec 22 17:02:04 server restic[1936]: snapshot 7d55b179 saved
    Dec 22 17:02:05 server restic[1946]: Applying Policy: keep all snapshots within 30d of the newest
    Dec 22 17:02:05 server restic[1946]: keep 1 snapshots:
    Dec 22 17:02:05 server restic[1946]: ID        Time                 Host        Tags        Reasons     P
    aths
    Dec 22 17:02:05 server restic[1946]: --------------------------------------------------------------------
    ----
    Dec 22 17:02:05 server restic[1946]: 7d55b179  2021-12-22 17:02:03  server                  within 30d  /
    root
    Dec 22 17:02:05 server restic[1946]: --------------------------------------------------------------------
    ----
    Dec 22 17:02:05 server restic[1946]: 1 snapshots
    Dec 22 17:02:05 server restic[1946]: keep 1 snapshots:
    Dec 22 17:02:05 server restic[1946]: ID        Time                 Host        Tags        Reasons     P
    aths
    Dec 22 17:02:05 server restic[1946]: --------------------------------------------------------------------
    ----
    Dec 22 17:02:05 server restic[1946]: 0047e386  2021-12-22 17:02:02  server                  within 30d  /
    etc
    Dec 22 17:02:05 server restic[1946]: --------------------------------------------------------------------
    ----
    Dec 22 17:02:05 server restic[1946]: 1 snapshots
    Dec 22 17:02:05 server restic[1946]: keep 1 snapshots:
    Dec 22 17:02:05 server restic[1946]: ID        Time                 Host        Tags        Reasons     P
    aths
    Dec 22 17:02:05 server restic[1946]: --------------------------------------------------------------------
    -------
    Dec 22 17:02:05 server restic[1946]: 285d0864  2021-12-22 17:02:03  server                  within 30d  /
    var/log
    Dec 22 17:02:05 server restic[1946]: --------------------------------------------------------------------
    -------
    Dec 22 17:02:05 server restic[1946]: 1 snapshots
    Dec 22 17:02:06 server restic[1955]: loading indexes...
    Dec 22 17:02:06 server restic[1955]: loading all snapshots...
    Dec 22 17:02:06 server restic[1955]: finding data that is still in use for 3 snapshots
    Dec 22 17:02:06 server restic[1955]: [0:00] 100.00%  3 / 3 snapshots
    Dec 22 17:02:06 server restic[1955]: searching used packs...
    Dec 22 17:02:06 server restic[1955]: collecting packs for deletion and repacking
    Dec 22 17:02:06 server restic[1955]: [0:00] 100.00%  8 / 8 packs processed
    Dec 22 17:02:06 server restic[1955]: to repack:            0 blobs / 0 B
    Dec 22 17:02:06 server restic[1955]: this removes          0 blobs / 0 B
    Dec 22 17:02:06 server restic[1955]: to delete:            0 blobs / 0 B
    Dec 22 17:02:06 server restic[1955]: total prune:          0 blobs / 0 B
    Dec 22 17:02:06 server restic[1955]: remaining:          294 blobs / 1.065 MiB
    Dec 22 17:02:06 server restic[1955]: unused size after prune: 0 B (0.00% of remaining size)
    Dec 22 17:02:06 server restic[1955]: done
    Dec 22 17:02:06 server systemd[1]: restic-backup.service: Succeeded.
    Dec 22 17:02:06 server systemd[1]: Finished Restic backup.
    ```
2) Check the restic snapshots

   - Load environment variables into shell console

         export $(grep -v '^#' /etc/restic/restic.conf | xargs -d '\n')
   - List backup snapshots

         restic --cacert /etc/restic/ssl/CA.pem snapshots
      Output should be like:
      
      ```
      repository df54412d opened successfully, password is correct
      ID        Time                 Host        Tags        Paths
      ---------------------------------------------------------------
      0047e386  2021-12-22 17:02:02  server                  /etc
      285d0864  2021-12-22 17:02:03  server                  /var/log
      7d55b179  2021-12-22 17:02:03  server                  /root
      ---------------------------------------------------------------
      3 snapshots
      ```    
     
Dependencies
------------

None

Example Playbook
----------------

The following playbook install restic, and schedule backup of `/etc`, `/var/log` and `/root` directories. As backend uses a S3 repository (`s3:https://10.11.0.1:9091/restic`) and to validate SSL certificates from S3 server installs a CA cert load from `certicates/CA.pem`.

```yml
- name: Converge | Configure backup
  hosts: server
  become: true
  gather_facts: true
  pre_tasks:
    - name: Load tls key and cert
      set_fact:
        restic_ca_cert: "{{ lookup('file','certificates/CA.pem') }}"
  roles:
    - role: ricsanfre.backup
      restic_repository: "s3:https://10.11.0.1:9091/restic"
      restic_aws_access_key_id: restic
      restic_aws_secret_access_key: supers1cret0
      restic_use_ca_cert: true
      restic_backups_dirs:
        - path: '/etc'
        - path: '/var/log'
        - path: '/root'
          exclude: '--exclude .cache'
```


License
-------

MIT

Author Information
------------------

Created by Ricardo Sanchez (ricsanfre) taking main idea of using systemd timer as a backup schedurle and base code from the repository (https://github.com/angristan/ansible-restic) by [angristan](https://github.com/angristan).
Code updated to work properly with S3 repository (i.e. Minio), solving some issues with the initialization of the repo and the use of custom CA/selfsigned certificates in the communication.
Additionally restic installation process is architecture agnostic (supporting x86 and ARM architectures). Original code supported only x86 architectures.
