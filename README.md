# Alpine Remote Backup

## Introduction

Alpine remote backup (RBU) is a set of scripts based on LBU
(Alpine Local Backup). It consist of a main script which need to be run from
cron and a set of profiles which can be used as pre/post scripts for LBU.
The idea of profiles is to simplify the backup process and to make sure
services inside Alpine infrastructure are setup in a similar way. Some of the
features/profiles of RBU are:

* Dump databases to a cache directory
* Synchronise backups to remote location
* Copy backup status log to remote location
* Send status notifications over mqtt

The alpine-backup script itself takes care of capturing the logs, creating the
snapshot, and updating the status for monitoring. Other tasks such uploading the
backup should be done through profiles. See the *Minimal Setup* section for
details.

## Prerequisite

* POSIX shell
* LBU (default on Alpine installation)
* rsync `apk add rsync`
* mosquitto-pub `apk add mosquitto-clients`

## Usage

### Configuration

RBU needs a configuration file named `rbu.conf`. This configuration should
(by default) be put in the lbu configuration directory `/etc/lbu`. Inside this
configuration should be a list of exported configuration variables which are
needed for lbu and its pre/post scripts to function properly.

### Profiles

Profiles are a set of LBU pre/post scripts. To use these scripts you should
symlink (or copy) them from the profile directory to one of the following dirs:

* `/etc/lbu/pre-package.d` (scripts run before LBU creates its backup)
* `/etc/lbu/post-package.d` (scripts run after LBU created its backup)

These profiles/scripts will be run by busybox run-parts which means they cannot
have an extension and need to be executable. If you need to run these scripts
in a specific order you should prefix them with a number like `10-scriptname`.

### Minimal setup

The `profiles/` directory contains some default profiles which can be used to
setup basic back-ups.

At the minimum, you want to add the `backup-sync` script in the `post-package.d`
directory to actually upload the backup archive to the remote server in the
homedir of the user provided in `rbu.conf`.

`backup-list` can be placed in the `pre-package.d` directory to generate a list
of files that are being backed up, which will added to the log.

### Encryption

To make sure the backups are safe we use openssl encryption option provided by
LBU. Please make sure you provide a secure password in `rbu.conf` and make sure
you have a backup of that password in case you will need it when restoring.

### Include non etc files

By default LBU will only backup files located in the `/etc` directory and have
been modified. To include other files you need to `lbu_include` files and
directories. Exclusion is possible via `lbu_exclude`. Please check
[Alpine wiki](http://wiki.alpinelinux.org/wiki/Alpine_local_backup) for more
information regarding LBU usage.

### Backup server

Backups will be kept in the specified backup location in rbu.conf. After the
backup is successful these backups will be rsynced to the remote backup server.
To be able to automatically rsync backups a ssh key needs to be generated per
backup. For generating a Ed25519 key exec: `ssh-keygen -t ed25519` and copy the
public key to the backup server.

### Enable daily backups

Copy the provided `alpine-backup` to `/etc/periodic/daily` to make the backup
run every night.

### Retention

LBU is configured to keep 7 days of backups. If you also want a monthly backup,
enable the backup-archive profile in `post-package.d` and make sure that it's
run after the `backup-sync` profile. This will copy the last created backup at
the first of each month to the `month/` directory..

## Backup monitoring

Backups are are automatically monitored by Alpine monitoring system. After the
backup has finished it will send a message with metadata to our mqtt server
which will be picked up by Alpine monitoring system.

## NOTE

Older version of LBU have a bug which prevents the cleanup of older local
encrypted backups. see:
https://git.alpinelinux.org/cgit/alpine-conf/commit/?id=cd395b
