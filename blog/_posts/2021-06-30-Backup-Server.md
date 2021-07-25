---
layout: post
title: Cloud backup-server
subtitle: How to setup a backup-server to your cloud
categories: Cloud
tags: [ZFS, Mail]
---
Any correction, improvement, or suggestion send [here](https://github.com/Igortorrente/Blog).

This is the follow-up of the series of posts that started [here]({% post_url 2021-06-27-Cloud-Posts %}).

In this post, we will configure the backup-server. This server will periodically compare the snapshot
between it and the cloud server and make incremental backups of the difference.

In this case, the backup will the active side. It will start the communication with the server to initiate the backup
process. This decision was made because the backup-server will be under a NAT
[<sup>[1]</sup>](https://en.wikipedia.org/wiki/Network_address_translation). But if you have a backup-server with a
valid IP, you take the reverse path.

Almost all steps will be the same as the Cloud server, so will only put a reference to the
[Cloud server]({% post_url 2021-06-28-Cloud-server %}) section.


## Pre-requisites:

- Minimal installation of Debian/Ubuntu.
- Internet connection.
- At least 3 identical spare hard drives.
- A configured sudo user.
- A encrypted `/` and `/root`.

## Hardware configuration [TODO]

## Packages

``` console
$ sudo apt update && sudo apt dist-upgrade -y && sudo apt autoremove -y && sudo apt install software-properties-common -y
$ sudo apt-add-repository contrib
$ sudo apt update
$ sudo apt install sanoid ssh debsums unattended-upgrades needrestart apt-listchanges htop net-tools wget curl dpkg-dev linux-headers-$(uname -r) linux-image-amd64 zfs-dkms zfsutils-linux -y
```

## Unattended-upgrades

Follow the steps under [Unattended-upgrades]({% post_url 2021-06-28-Cloud-server %}#h-unattended-upgrades).

## ZFS

In the backup-server, we will also use the ZFS to enjoy all of its benefits (And also because the backup only
works between ZFS zpools).

In the backup server, we will use only the raidz instead of the `raidz2`. This choice lowers the number of disks
that we need (from 4 to 3) and increases the disk space available compared to the `raidz2`(with the same amount of disks).
The drawback is that we can only lose one disk before we lose data (against two of the `raidz2`).

But considering that is a backup server, I think this is a fair trade.

### Kernel module

Load the kernel module.

``` console
$ sudo modprobe zfs
```

### Configuring

List the disks
[<sup>[2]</sup>](https://wiki.debian.org/ZFS#Creating_the_Pool)
[<sup>[3]</sup>](https://wiki.archlinux.org/title/ZFS#Identify_disks).

`ls -l /dev/disk/by-id/` OR `ls -l /dev/disk/by-path/`

Find the disks sector size.

``` console
$ sudo fdisk -l
```

Now we will create our first zpool with spare disks (If the disks sectors are 4k add `-o ashift=12` bellow, if 8k add
`-o ashift=13`)[6][7]

```console
$ sudo zpool create -o autoexpand=on -o autoreplace=on -m /var/server_data server_data raidz <disk1> <disk2> <disk3> ... <diskN>
```

Check the state. You should see a similar result as in the
[Cloud server]({% post_url 2021-06-28-Cloud-server %}#h-configuring).

``` console
$ sudo zpool status
```

Set the default ext4/XFS atime semantics to ZFS.

``` console
$ sudo zfs set atime=on caedrium_backup
$ sudo zfs set relatime=on caedrium_backup
```

### Scrubbing

Same step as [Scrubbing section]({% post_url 2021-06-28-Cloud-server %}#h-scrubbing).

### Configuring Backup

Follow the steps in [Configuring Backup section]({% post_url 2021-06-28-Cloud-server %}#h-configuring-backup).

Using the Cron, we will make a `raw`[<sup>[4]</sup>](https://github.com/openzfs/zfs/issues/7966) backup every
4 hours with the max bandwidth at 1 MegaByte/s. And we will access the Cloud server using the `backup-user` as defined
[here]({% post_url 2021-06-28-Cloud-server %}#h-backup-user).

``` console
$ sudo crontab -e
```

``` diff
--- /tmp/crontab	2021-07-26 15:30:48.890516624 -0300
+++ /tmp/crontab	2021-07-26 15:30:55.466017258 -0300
@@ -1,4 +1,4 @@
+ \* \*/4 \* \* \* /usr/sbin/syncoid --recursive --skip-parent --no-privilege-elevation --sendoptions=w --source-bwlimit 1M backup-user@<Server-IP>:server_data backup_pool
 
```

## Service emails

We will use all the infrastructure created in the Cloud server to inform us of the state of the backup-server. You
can create a special email account to the backup-server if you want to. But here I will use the same as the main server.

- [system-bot@**\<Mail-Hostname>**{: style="color: #f87e19" }] - System notification bot.
- [no-reply@**\<Mail-Hostname>**{: style="color: #f87e19" }] - To be used by several services.

### Unattended-Upgrade

The same as [Unattended-upgrades]({% post_url 2021-06-28-Cloud-server %}#h-unattended-upgrade) in the
[Service emails](#h-service-emails) section.

### ZFS mail bot

Surprise, surprose, same steps as [ZFS mail bot]({% post_url 2021-06-28-Cloud-server %}#h-zfs-mail-bot) in the
[Service emails](#h-service-emails) section.

## Maintenance

The following procedures are also useful to our backup-server.

- [System binary corruption]({% post_url 2021-06-28-Cloud-server %}#h-system-binary-corruption).
- [Disk failure]({% post_url 2021-06-28-Cloud-server %}#h-disk-failure).
- [Useful commands]({% post_url 2021-06-28-Cloud-server %}#h-useful-commands).
