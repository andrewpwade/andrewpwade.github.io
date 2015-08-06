---
title: Making a time machine with rsnapshot
layout: post
---

I backup my desktop's root filesystem (on LVM) to a RAID 1 array which
contains an encrypted filesystem (LUKS), to which hourly, daily,
weekly and monthly snapshots are transferred using
[rsnapshot](http://www.rsnapshot.org/).

Using a [wrapper script](https://github.com/andrewpwade/scripts/blob/master/rsnapbackup) I wrote, after each snapshot is taken,
symlinks named after the date of the backup are created to make
restoration easier.

rsnapshot is nifty because it makes use of hard links to reduce disk
usage. This enables multiple full backups, each of which only requires
disk space for what changed between each backup. There are no
incremental backups to restore in order.

Also, it utilises LVM's snapshot facility to ensure consistent backups.

Tools required:

 - mdadm
 - rsnapshot
 - cryptsetup
<p></p>

### Create RAID and encrypted device ###

I'll assume you have 2 or more partitions formatted as type 'fd' (Linux RAID autodetect). In this example I'll use /dev/sdb1 and /dev/sdc1 to form /dev/md0:

~~~
mdadm --create /dev/md0 --level=1 --raid-disks=2 /dev/sd[cb]1
~~~

Verify the RAID with `cat /proc/mdstat`. Afer it has finished syncing, it should show something similar to `md0 : active raid1 sdb1[2] sdc1[1]`

Next, set up LUKS. Here, I'm choosing to set it up on the whole device rather than partition:

~~~
cryptsetup --verbose --verify-passphrase luksFormat /dev/md0
~~~

Optionally back up the LUKS header somewhere safe:

~~~
sudo cryptsetup luksHeaderBackup --header-backup-file /somewhere/super/safe/backup_luks_header /dev/md0
~~~

Now activate the encrypted device and make a filesystem:

~~~
cryptsetup luksOpen /dev/md0 backup01
mkfs.ext4 /dev/mapper/backup01
~~~

And mount it:

~~~
mkdir -p /mnt/backup01
mount /dev/mapper/backup01 /mnt/backup01
~~~

Optionally, adjust `/etc/mdadm/mdadm.conf` so that the new array is started on boot. The command `mdadm --detail --scan` will output the required configuration that needs to be appended to mdadm.conf, but be careful, as it outputs configuration for all arrays.

### Rsnapshot ###

Install it:

~~~
sudo apt-get update
sudo apt-get install rsnapshot
~~~

Edit `/etc/rsnapshot.conf` and adjust the following settings:

 - snapshot_root
 - no_create_root (value "1")
 - retain
 - exclude (if necessary)
 - link_dest (value "1")
 - linux_lvm_mountpath
 - backup (e.g. "backup lvm://vg/root/ root/")

 These are the settings I use:

~~~
snapshot_root	/mnt/backup01/rsnapshot/
no_create_root	1
retain		hourly	6
retain		daily	7
retain		weekly	4
retain		monthly	6
exclude		/dev
exclude		/proc
exclude		/sys
exclude		/tmp
exclude		/run
exclude		/mnt
exclude		/media
exclude		/lost+found
exclude		/usr/share/**
exclude		/home/andrew/Downloads/**
exclude		/home/andrew/.local/share/**
exclude		/home/andrew/.cache/**
link_dest	1
linux_lvm_mountpath	/mnt/snapshot/
backup	lvm://xubuntu-vg/root/	xubuntu-vg-root/
~~~
 
Test the configuration:

~~~
sudo rsnapshot configtest
...
Syntax OK
~~~

Download the wrapper script I mentioned at the beginning ([rsnapbackup](https://github.com/andrewpwade/scripts/blob/master/rsnapbackup)) and perform a backup.

~~~
sudo rsnapbackup -l hourly
~~~

A backup should exist under `/mnt/backup01/rsnapshot/hourly.0/` and `/mnt/backup01/rsnapshot/dates/` should contain a symlik to `hourly.0`
