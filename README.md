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

- Directories list to be backed up

  `restic_backup_dirs` is a list of dictionaries. Each item of the list is a directory to  include in the backup
  Each dictionary item has a `path` and an `exclude` (which defaults to nothing). The `exclude` property is a list of exclude patterns to be passed as `--exclude` argument passed to restic when executing the backup (example: `restic backup /root --exclude .cache --exclude .ignore`).

  ```yml
  # restic backup directories
  restic_backups_dirs:
    - path: '/etc'
    - path: '/var/log'
    - path: '/root'
      exclude:
        - pattern: '.cache'
        - pattern: '.ignore' 
  ```

- Restic additional flags

  `restic_flags` additional restic commands flags to be included in the execution of all commands. Playbook automatically add --cacert flag if `restic_use_ca_cert` is set to true.
  
  ```yml
  restic_flags: ""
  ```

- Restic logs
 
  `restic_logs` restic scripts log file. 
  
  ```yml
  restic_log: /var/log/restic.log
  ```
### Restic repository cleaning tasks

A specific systemd service will be configured to execute checking and purging activities. This service will be independent from the backup service, since it is only needed to be executed from one server and to avoid mutual locking it need to be scheduled differently.

Cleaning systemd service will execute a script containing the following restic commands

- `restic check`
- `restic forget --keep-within <data_retention_days>`
- `restic prune`

`restic_clean_service` indicates whether to install or not the restic cleaning service and `restic_forget_keep_within` retention data polocy (`--keep-within` forget parameter)

```yml
restic_clean_service: true
et as post backup task (restic forget --keep-within <keep_within>)
restic_forget_keep_within: 30d
```

### Sytemd services and timers

Two systemd services of type `oneshot` will be created that will be triggered periodically with its corresponding systemd timers.

For executing backup process: `restic-backup.service` and `restic-backup.timer` are created.
For executing cleaning process: `restic-clean.service` and `restic-clean.timer` are created

The timers are configurable as follows:

- `restic_backup_systemd_timer_on_calender` and `restic_clean_systemd_timer_on_calender`: defines the `OnCalendar` directive (`*-*-* 03:00:00`)
- `restic_backup_systemd_timer_randomized_delay_sec` and `restic_clean_systemd_timer_randomized_delay_sec`: Delay the timer by a random amount of time between 0 and the specified time value. (`0`)

See the [systemd.timer](https://www.freedesktop.org/software/systemd/man/systemd.timer.html) documentation for more information.

You can see the logs of the backup/cleaning with `journalctl`.

    journalctl -xefu restic-backup
    journalctl -xefu restic-clean

Logs are also stored in a file indicated by `restic_log`

The backup/cleaning process can be also triggered manually running the command

    systemctl start restic-backup
    systemctl start restic-clean


Testing
--------

Ansible Playbook based on the configuration creates the scripts that are launched by the systemd services. Those scripts are stored in `restic_etc_dir` (/etc/restic)

- `restic-repo-init.sh`: Script for initializing the restic repo. Executed by ansible playbook once
- `restic-backup.sh`: script executed by `restic-backup` systemd service
- `restic-clean.sh`: script executed by `restic-clean` systemd service
- `restic-wrapper.sh`: restic wrapper script used by the rest of the scripts. This script load the repository variables stored in `/etc/restic/restic.conf` and if necessary pass the '--cacert' parameter to all restic commands.

After installation the backup process can be tested following this procedure:

1) Trigger manually the backup process

       systemctl start restic-backu
    
   Or

       /etc/restic/restic-backup.sh

2) Check logs restic-backup service

       journalctl -u restic-backup
   Or

       tail -f /var/log/restic.log

    
   Output should be like:

    ```
    -- Logs begin at Tue 2021-12-28 10:38:54 UTC, end at Tue 2021-12-28 10:53:57 UTC. --
    Dec 28 10:50:01 server systemd[1]: Starting Restic backup...
    Dec 28 10:50:01 server restic-backup.sh[2751]: Dec 28 2021 10:50:01 UTC: restic-backup started
    Dec 28 10:50:01 server restic-backup.sh[2755]: -------------------------------------------------------------------
    ------------
    Dec 28 10:50:02 server restic-backup.sh[2757]: no parent snapshot found, will read all files
    Dec 28 10:50:02 server restic-backup.sh[2757]: Files:         219 new,     0 changed,     0 unmodified
    Dec 28 10:50:02 server restic-backup.sh[2757]: Dirs:          105 new,     0 changed,     0 unmodified
    Dec 28 10:50:02 server restic-backup.sh[2757]: Added to the repo: 730.675 KiB
    Dec 28 10:50:02 server restic-backup.sh[2757]: processed 219 files, 538.536 KiB in 0:00
    Dec 28 10:50:02 server restic-backup.sh[2757]: snapshot 8bc0e3ea saved
    Dec 28 10:50:03 server restic-backup.sh[2769]: no parent snapshot found, will read all files
    Dec 28 10:50:03 server restic-backup.sh[2769]: Files:          11 new,     0 changed,     0 unmodified
    Dec 28 10:50:03 server restic-backup.sh[2769]: Dirs:            5 new,     0 changed,     0 unmodified
    Dec 28 10:50:03 server restic-backup.sh[2769]: Added to the repo: 351.917 KiB
    Dec 28 10:50:03 server restic-backup.sh[2769]: processed 11 files, 346.573 KiB in 0:00
    Dec 28 10:50:03 server restic-backup.sh[2769]: snapshot bd7c3e4f saved
    Dec 28 10:50:03 server restic-backup.sh[2781]: no parent snapshot found, will read all files
    Dec 28 10:50:03 server restic-backup.sh[2781]: Files:           2 new,     0 changed,     0 unmodified
    Dec 28 10:50:03 server restic-backup.sh[2781]: Dirs:            3 new,     0 changed,     0 unmodified
    Dec 28 10:50:03 server restic-backup.sh[2781]: Added to the repo: 4.852 KiB
    Dec 28 10:50:03 server restic-backup.sh[2781]: processed 2 files, 3.190 KiB in 0:00
    Dec 28 10:50:03 server restic-backup.sh[2781]: snapshot dc362721 saved
    Dec 28 10:50:03 server restic-backup.sh[2792]: -------------------------------------------------------------------
    ------------
    Dec 28 10:50:03 server restic-backup.sh[2794]: Dec 28 2021 10:50:03 UTC: restic.sh finished
    Dec 28 10:50:03 server systemd[1]: restic-backup.service: Succeeded.
    Dec 28 10:50:03 server systemd[1]: Finished Restic backup.
    
    ```
3) Check the restic snapshots

   - List backup snapshots

         /etc/restic/restic_wrapper.sh snapshots
      
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
4) Trigger manually the clean process

       systemctl start restic-clean
    
   Or

       /etc/restic/restic-repo-clean.sh

5) Check logs restic-clean service

       journalctl -u restic-clean
   Or

       tail -f /var/log/restic.log
   
   Output is like:

    ```
    -- Logs begin at Tue 2021-12-28 10:38:54 UTC, end at Tue 2021-12-28 10:53:57 UTC. --
    Dec 28 10:52:20 server systemd[1]: Starting Restic check and purge...
    Dec 28 10:52:20 server restic-repo-clean.sh[2807]: Dec 28 2021 10:52:20 UTC: restic-repo-clean started
    Dec 28 10:52:20 server restic-repo-clean.sh[2811]: ---------------------------------------------------------------
    ----------------
    Dec 28 10:52:20 server restic-repo-clean.sh[2813]: using temporary cache in /tmp/restic-check-cache-652678859
    Dec 28 10:52:21 server restic-repo-clean.sh[2813]: create exclusive lock for repository
    Dec 28 10:52:21 server restic-repo-clean.sh[2813]: load indexes
    Dec 28 10:52:21 server restic-repo-clean.sh[2813]: check all packs
    Dec 28 10:52:21 server restic-repo-clean.sh[2813]: check snapshots, trees and blobs
    Dec 28 10:52:21 server restic-repo-clean.sh[2813]: no errors were found
    Dec 28 10:52:21 server restic-repo-clean.sh[2813]: [0:00] 100.00%  3 / 3 snapshots
    Dec 28 10:52:22 server restic-repo-clean.sh[2825]: Applying Policy: keep all snapshots within 30d of the newest
    Dec 28 10:52:22 server restic-repo-clean.sh[2825]: keep 1 snapshots:
    Dec 28 10:52:22 server restic-repo-clean.sh[2825]: ID        Time                 Host        Tags        Reasons
        Paths
    Dec 28 10:52:22 server restic-repo-clean.sh[2825]: ---------------------------------------------------------------
    ---------
    Dec 28 10:52:22 server restic-repo-clean.sh[2825]: dc362721  2021-12-28 10:50:03  server                  within 3
    0d  /root
    Dec 28 10:52:22 server restic-repo-clean.sh[2825]: ---------------------------------------------------------------
    ---------
    Dec 28 10:52:22 server restic-repo-clean.sh[2825]: 1 snapshots
    Dec 28 10:52:22 server restic-repo-clean.sh[2825]: keep 1 snapshots:
    Dec 28 10:52:22 server restic-repo-clean.sh[2825]: ID        Time                 Host        Tags        Reasons
        Paths
    Dec 28 10:52:22 server restic-repo-clean.sh[2825]: ---------------------------------------------------------------
    ---------
    Dec 28 10:52:22 server restic-repo-clean.sh[2825]: 8bc0e3ea  2021-12-28 10:50:01  server                  within 3
    0d  /etc
    Dec 28 10:52:22 server restic-repo-clean.sh[2825]: ---------------------------------------------------------------
    ---------
    Dec 28 10:52:22 server restic-repo-clean.sh[2825]: 1 snapshots
    Dec 28 10:52:22 server restic-repo-clean.sh[2825]: keep 1 snapshots:
    Dec 28 10:52:22 server restic-repo-clean.sh[2825]: ID        Time                 Host        Tags        Reasons
        Paths
    Dec 28 10:52:22 server restic-repo-clean.sh[2825]: ---------------------------------------------------------------
    ------------
    Dec 28 10:52:22 server restic-repo-clean.sh[2825]: bd7c3e4f  2021-12-28 10:50:02  server                  within 3
    0d  /var/log
    Dec 28 10:52:22 server restic-repo-clean.sh[2825]: ---------------------------------------------------------------
    ------------
    Dec 28 10:52:22 server restic-repo-clean.sh[2825]: 1 snapshots
    Dec 28 10:52:22 server restic-repo-clean.sh[2836]: loading indexes...
    Dec 28 10:52:22 server restic-repo-clean.sh[2836]: loading all snapshots...
    Dec 28 10:52:22 server restic-repo-clean.sh[2836]: finding data that is still in use for 3 snapshots
    Dec 28 10:52:22 server restic-repo-clean.sh[2836]: [0:00] 100.00%  3 / 3 snapshots
    Dec 28 10:52:22 server restic-repo-clean.sh[2836]: searching used packs...
    Dec 28 10:52:22 server restic-repo-clean.sh[2836]: collecting packs for deletion and repacking
    Dec 28 10:52:22 server restic-repo-clean.sh[2836]: [0:00] 100.00%  8 / 8 packs processed
    Dec 28 10:52:22 server restic-repo-clean.sh[2836]: to repack:            0 blobs / 0 B
    Dec 28 10:52:22 server restic-repo-clean.sh[2836]: this removes          0 blobs / 0 B
    Dec 28 10:52:22 server restic-repo-clean.sh[2836]: to delete:            0 blobs / 0 B
    Dec 28 10:52:22 server restic-repo-clean.sh[2836]: total prune:          0 blobs / 0 B
    Dec 28 10:52:22 server restic-repo-clean.sh[2836]: remaining:          301 blobs / 1.071 MiB
    Dec 28 10:52:22 server restic-repo-clean.sh[2836]: unused size after prune: 0 B (0.00% of remaining size)
    Dec 28 10:52:22 server restic-repo-clean.sh[2836]: done
    Dec 28 10:52:22 server restic-repo-clean.sh[2847]: ---------------------------------------------------------------
    ----------------
    Dec 28 10:52:22 server restic-repo-clean.sh[2849]: Dec 28 2021 10:52:22 UTC: restic-repo-clean finished
    Dec 28 10:52:22 server systemd[1]: restic-clean.service: Succeeded.
    Dec 28 10:52:22 server systemd[1]: Finished Restic check and purge.
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
          exclude:
            - pattern: '.cache'
            - pattern: '.ignore'
```


License
-------

MIT

Author Information
------------------

Created by Ricardo Sanchez (ricsanfre) highly inspired by this [project](https://github.com/angristan/ansible-restic) by [angristan](https://github.com/angristan).

Code completely refactored and updated to work properly with S3 repository (i.e. Minio) and with ARM architecture.

Changes and improvements:
- Restic backup and purge activities are splitted into two different systemd services that can be scheduled independetly to avoid locking issues.
- S3 backend issues solved: repo initialization  and the use of custom CA/selfsigned certificates in the communication.
- Restic installation installation process is architecture agnostic (supporting x86 and ARM architectures). Original code supported only x86 architectures.
