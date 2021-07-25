---
layout: post
title: How to host a alternative to the Google/Microsoft services
subtitle: A guide to install and configure a small cloud infrastructure
categories: Cloud
tags: [Nextcloud, Roundcube, Matrix, Mail, Debian, ZFS]
---
Any correction, improvement, or suggestion send [here](https://github.com/Igortorrente/Blog).

This post is a continuation of the [Cloud post series]({% post_url 2021-06-27-Cloud-Posts %}).
Here, I we will setup our server with all the services described in the first post.

## Pre-requisites:

- Minimal installation of Debian/Ubuntu.
- Internet connection.
- At least 5 identical spare hard drives.
- A configured sudo `<user>`.
- Port forwarding for web server (80, 443) and email (25, 587, 110, 995, 143, 993) in the router[<sup>[1]</sup>](https://docs.iredmail.org/network.ports.html).
- A encrypted `/` and `/root`.
- A valid domain.
- ... (TBD)

{: .box-note}
**Note:** This tutorial was tested on Debian 10 and 11, but it should work on ubuntu and derivates.

## Hardware recomendation [TODO]

## Packages

### Update

Update the packages.

``` console
$ sudo apt update && sudo apt dist-upgrade -y && sudo apt autoremove -y
```

### Install

The ZFS is under the `contrib` in the Debian repositories, so we need to enable it and install all the necessary packages.

``` console
$ sudo apt install software-properties-common -y
$ sudo apt-add-repository contrib
$ sudo apt update
$ sudo apt install certbot lnav sanoid ufw debsums unattended-upgrades needrestart apt-listchanges htop net-tools wget curl nginx unzip postgresql redis-server php-fpm php php-curl php-pgsql php-fdomdocument php-apcu php-bcmath php-cli php-common php-curl php-gd php-gmp php-imagick php-intl php-mbstring php-zip php-xml php-fpm php-pear php-json php-soap php-ldap php-imap php-pspell php-redis dpkg-dev linux-headers-$(uname -r) linux-image-amd64 zfs-dkms zfsutils-linux -y
```

## SSH

In the future, we will create a user called `backup-user` that will be responsible for backup that data out of our
server (check [Backup user](#h-backup-user) section).<br>
But we don't want to allow anyone to login as this user, so we need to change the some ssh server configuration to improve
the security.

``` diff
--- /etc/ssh/sshd_config	2021-07-26 14:47:11.081050657 -0300
+++ /etc/ssh/sshd_config	2021-07-26 15:16:47.689863093 -0300
@@ -31,11 +31,20 @@
 # Authentication:
 
 #LoginGraceTime 2m
-#PermitRootLogin prohibit-password
+PermitRootLogin no
 #StrictModes yes
 #MaxAuthTries 6
 #MaxSessions 10
 
+AllowUsers <user> backup-user
+
+Match User backup-user
+        Banner none
+        PasswordAuthentication no
+        AllowTcpForwarding no
+        ForceCommand export PATH=$PATH:/usr/zfs-bin; bash -c "$SSH_ORIGINAL_COMMAND"
+Match all
+
 #PubkeyAuthentication yes
 
 # Expect .ssh/authorized_keys2 to be disregarded by default in future.
@@ -56,7 +65,7 @@
 
 # To disable tunneled clear text passwords, change to no here!
 #PasswordAuthentication yes
-#PermitEmptyPasswords no
+PermitEmptyPasswords no
 
 # Change to yes to enable challenge-response passwords (beware issues with
 # some PAM modules and threads)
@@ -86,9 +95,9 @@
 UsePAM yes
 
 #AllowAgentForwarding yes
-#AllowTcpForwarding yes
+AllowTcpForwarding yes
 #GatewayPorts no
-X11Forwarding yes
+X11Forwarding no
 #X11DisplayOffset 10
 #X11UseLocalhost yes
 #PermitTTY yes
@@ -97,7 +106,7 @@
 #TCPKeepAlive yes
 #PermitUserEnvironment no
 #Compression delayed
-#ClientAliveInterval 0
+ClientAliveInterval 180
 #ClientAliveCountMax 3
 #UseDNS no
 #PidFile /var/run/sshd.pid
 ```

Restart the service.

``` console
$ sudo systemctl restart ssh.service
```

## Unattended-upgrades

A good security measure is to automate the package update procedure. And to perform that we will use the
Unattended-upgrades[<sup>[2]</sup>](https://wiki.debian.org/UnattendedUpgrades).<br>
But we need change some configurations to fit server requirements.

``` diff
--- /etc/apt/apt.conf.d/50unattended-upgrades	2021-07-26 14:49:15.217300443 -0300
+++ /etc/apt/apt.conf.d/50unattended-upgrades	2021-07-26 14:50:47.065457281 -0300
@@ -85,7 +85,7 @@
 // big step back from the 30 minutes allowed for InstallOnShutdown previously.
 // Users enabling InstallOnShutdown mode are advised to increase
 // InhibitDelayMaxSec even further, possibly to 30 minutes.
-//Unattended-Upgrade::InstallOnShutdown "false";
+Unattended-Upgrade::InstallOnShutdown "false";
 
 // Send email to this address for problems or packages upgrades
 // If empty or unset then no email is sent, make sure that you
@@ -101,22 +101,22 @@
 
 // Remove unused automatically installed kernel-related packages
 // (kernel images, kernel headers and kernel version locked tools).
-//Unattended-Upgrade::Remove-Unused-Kernel-Packages "true";
+Unattended-Upgrade::Remove-Unused-Kernel-Packages "false";
 
 // Do automatic removal of newly unused dependencies after the upgrade
-//Unattended-Upgrade::Remove-New-Unused-Dependencies "true";
+Unattended-Upgrade::Remove-New-Unused-Dependencies "false";
 
 // Do automatic removal of unused packages after the upgrade
 // (equivalent to apt-get autoremove)
-//Unattended-Upgrade::Remove-Unused-Dependencies "false";
+Unattended-Upgrade::Remove-Unused-Dependencies "false";
 
 // Automatically reboot *WITHOUT CONFIRMATION* if
 //  the file /var/run/reboot-required is found after the upgrade
-//Unattended-Upgrade::Automatic-Reboot "false";
+Unattended-Upgrade::Automatic-Reboot "false";
 
 // Automatically reboot even if there are users currently logged in
 // when Unattended-Upgrade::Automatic-Reboot is set to true
-//Unattended-Upgrade::Automatic-Reboot-WithUsers "true";
+Unattended-Upgrade::Automatic-Reboot-WithUsers "false";
 
 // If automatic reboot is enabled and needed, reboot at the specific
 // time instead of immediately
```

Restart the service to apply the new configuration.

``` console
$ sudo systemctl restart unattended-upgrades.service
```

## ZFS

I choose the ZFS because it is one of the best(if not the best) File-System/Volume-Manager available for Linux.
With a lot of engineering time(sorry btrfs), has robust error detection and correction, and encryption.
All required features to our cloud.

And also has very nice features like snapshot, incremental backups, and others. In few words, the ZFS is the best
option considering our requirements.

But there's a caveat, we need ECC memory and at least 4 spare disks if we want to use the `raidz2`.

`raidz2` is the ZFS implementation of the [RAID6](https://en.wikipedia.org/wiki/Standard_RAID_levels#RAID_6),
and it comes with all benefits and drawbacks of the RAID5. With the `raidz2` we can lose two disks **without**
losing **any** data.

But we lose a considerable space in our `zpool`. You can calculate the final space with this
[formula](https://en.wikipedia.org/wiki/Standard_RAID_levels#Comparison):

$$1 - \frac{2}{N}$$

$N$ being the number of disks.

Considering that data integrity is a requirement, this tradeoff is pretty reasonable.

### Kernel module

Load the ZFS kernel module.

``` console
$ sudo modprobe zfs
```

### Configuring

The `zfs-mount.service` of Debian 11 doesn't load the encryption keys at the boot
[<sup>[3]</sup>](https://github.com/jimsalterjrs/sanoid/issues/648), so we need to change it.

``` diff
--- /etc/systemd/system/zfs.target.wants/zfs-mount.service	2021-07-26 14:56:50.446013082 -0300
+++ /etc/systemd/system/zfs.target.wants/zfs-mount.service	2021-07-26 14:51:23.241515683 -0300
@@ -11,7 +11,6 @@
 [Service]
 Type=oneshot
 RemainAfterExit=yes
+ExecStart=/bin/sh -c "/sbin/zfs load-key -a; exit 0;"
 ExecStart=/sbin/zfs mount -a
 
 [Install]
```

Restart the `zfs-mount.service`.

``` console
$ sudo systemctl daemon-reload
$ sudo systemctl restart zfs-mount.service
```

List the disks
[<sup>[4]</sup>](https://wiki.debian.org/ZFS#Creating_the_Pool)
[<sup>[5]</sup>](https://wiki.archlinux.org/title/ZFS#Identify_disks).

`$ ls -l /dev/disk/by-id/` OR  `$ ls -l /dev/disk/by-path/`

Find the disks sector size:

``` console
$ sudo fdisk -l
```

Now we will create our first zpool with spare disks (If the disks sectors are 4k add `-o ashift=12` below,
if 8k add `-o ashift=13`)
[<sup>[6]</sup>](https://wiki.debian.org/ZFS#Basic_Configuration)
[<sup>[7]</sup>](https://zfsonlinux.org/manpages/0.8.1/man8/zpool.8.html#lbAG).

``` console
$ sudo zpool create -o autoexpand=on -o autoreplace=on -m /var/server_data server_data raidz2 disk1 disk2 diskN... spare diskN1 diskN2
```

Check the state.

``` console
$ sudo zpool status
```

Output example:

``` console
$ sudo zpool create -o autoexpand=on -o autoreplace=on -m /var/server_data server_data raidz2 ata-ST3000DM001-9YN166_S1F0KDGY ata-ST3000DM001-9YN166_S1F0JKRR ata-ST3000DM001-9YN166_S1F0KBP8 ata-ST3000DM001-9YN166_S1F0JTM1 spare ata-ST3000DM001-9YN166_S1FKJTSS1
$ sudo zpool status
pool: server_data
state: ONLINE
config:
        NAME                                 STATE     READ WRITE CKSUM
        server_data                          ONLINE       0     0     0
          raidz2-0                           ONLINE       0     0     0
            ata-ST3000DM001-9YN166_S1F0KDGY  ONLINE       0     0     0
            ata-ST3000DM001-9YN166_S1F0JKRR  ONLINE       0     0     0
            ata-ST3000DM001-9YN166_S1F0KBP8  ONLINE       0     0     0
            ata-ST3000DM001-9YN166_S1F0JTM1  ONLINE       0     0     0
        spares
            ata-ST3000DM001-9YN166_S1FKJTSS1 AVAILABLE
errors: No known data errors
```

The ZFS allows us two methods to decrypt the disks. One is `prompt`, that will ask the password at the boot time, other
is `raw`, that gets a 32 bytes string.

To this guide, we will use the `raw` that will be stored in the encrypted `/root`.

First, generate keys to encrypt our datasets[<sup>[8]</sup>](https://wiki.archlinux.org/title/ZFS#Native_encryption).

``` console
$ sudo dd if=/dev/random of=/root/zfs_dataset_postgres_pass bs=1 count=32
$ sudo dd if=/dev/random of=/root/zfs_dataset_mail_pass bs=1 count=32
$ sudo dd if=/dev/random of=/root/zfs_dataset_nextcloud_pass bs=1 count=32
$ sudo dd if=/dev/random of=/root/zfs_dataset_log_pass bs=1 count=32
$ sudo dd if=/dev/random of=/root/zfs_dataset_matrix_pass bs=1 count=32
$ sudo chmod 600 /root/zfs_dataset_nextcloud_pass /root/zfs_dataset_mail_pass /root/zfs_dataset_postgres_pass /root/zfs_dataset_log_pass /root/zfs_dataset_matrix_pass
```

Copy all these keys to a safe place (like KeepassXC), and then create the datasets.

``` console
$ sudo zfs create -o encryption=on -o keyformat=raw -o keylocation=file:///root/zfs_dataset_postgres_pass server_data/postgres
$ sudo zfs create -o encryption=on -o keyformat=raw -o keylocation=file:///root/zfs_dataset_mail_pass server_data/mail
$ sudo zfs create -o encryption=on -o keyformat=raw -o keylocation=file:///root/zfs_dataset_nextcloud_pass server_data/nextcloud
$ sudo zfs create -o encryption=on -o keyformat=raw -o keylocation=file:///root/zfs_dataset_log_pass server_data/log
$ sudo zfs create -o encryption=on -o keyformat=raw -o keylocation=file:///root/zfs_dataset_matrix_pass server_data/matrix
```

Check to see if everything is correct.

``` console
$ sudo zfs list -t filesystem
NAME                    USED  AVAIL     REFER  MOUNTPOINT
server_data             958K  38.4G     35.9K  /var/server_data
server_data/log         111K  38.4G      111K  /var/server_data/log
server_data/mail        111K  38.4G      111K  /var/server_data/mail
server_data/matrix      111K  38.4G      111K  /var/server_data/matrix
server_data/nextcloud   111K  38.4G      111K  /var/server_data/nextcloud
server_data/postgres    111K  38.4G      111K  /var/server_data/postgres
```

Add a little optimization to the `block_size` of the postgres dataset
[<sup>[9]</sup>](https://wiki.archlinux.org/title/ZFS#Databases).

``` console
$ sudo zfs set recordsize=8K server_data/postgres
```

And another little tweak to bring the default ext4/XFS `atime` semantics to ZFS. We are applying to the `zpool`, but the
datasets will inherit this characteristic from the `server_data`.

``` console
$ sudo zfs set atime=on server_data
$ sudo zfs set relatime=on server_data
```

### Scrubbing

A quote directly from the arch wiki[<sup>[10]</sup>](https://wiki.archlinux.org/title/ZFS#Scrubbing) summarizes
well what is scrubbing.

> Whenever data is read and ZFS encounters an error, it is silently repaired when possible, rewritten back to disk
> and logged so you can obtain an overview of errors on your pools. There is no fsck or equivalent tool for ZFS.
> Instead, ZFS supports a feature known as scrubbing. This traverses through all the data in a pool and verifies
> that all blocks can be read.

As recommended in the Arch wiki, we will add new a systemd service
[<sup>[11]</sup>](https://wiki.archlinux.org/title/ZFS#Start_with_a_service_or_timer)
to do the scrubbing once a month.

{% highlight toml wl linenos %}
/etc/systemd/system/zfs-scrub@.timer

[Unit]
Description=Monthly zpool scrub on %i

[Timer]
OnCalendar=monthly
AccuracySec=1h
Persistent=true

[Install]
WantedBy=multi-user.target
{% endhighlight %}

{% highlight toml wl linenos %}
/etc/systemd/system/zfs-scrub@.service

[Unit]
Description=zpool scrub on %i

[Service]
Nice=19
IOSchedulingClass=idle
KillSignal=SIGINT
ExecStart=/usr/sbin/zpool scrub %i

[Install]
WantedBy=multi-user.target
{% endhighlight %}

Reload, enable, start, and check the status of the `zfs-scrub` service.

``` console
$ sudo systemctl daemon-reload
$ sudo systemctl enable zfs-scrub@server_data.timer --now
$ sudo systemctl status zfs-scrub@server_data.timer
```

### Configuring Backup

Would be nice to have the snapshot and backup process happening automatically. Fortunately, tools like the
[sanoid and syncoid](https://github.com/jimsalterjrs/sanoid) exist. The job of sanoid is to take and manage
ZFS snapshots based on the retention policy establlished in the config, and syncoid is to synchronize
(over ssh for example) ZFS datasets between two(maybe more)
ZFS Pools using the ZFS feature of [send](https://openzfs.github.io/openzfs-docs/man/8/zfs-send.8.html?highlight=send)
and [recv](https://openzfs.github.io/openzfs-docs/man/8/zfs-recv.8.html) to achieve incremental backups
between ZFS Pools.

One killer feature of the ZFS to our usecase, the use of `zfs send --raw` by the syncoid, this allows us to send
encrypted incremental backups to a backup server. This means that the backup server only will contain an encrypted
version of our datasets since we send it encrypted and didn't send the decryption keys. And therefore we don't
need to care about the safety of the backup-server.

#### Sanoid

Create the sanoid configuration folder(Because Debian doesn't create it for us `¯\_(ツ)_/¯`).

``` console
$ sudo mkdir /etc/sanoid/
```

Fill the configuration file with the retention policy. I choose values that makes sense for my zpool space, but you can
change the `template_production` as you wish.

{% highlight toml wl linenos %}
/etc/sanoid/sanoid.conf

[server_data/log]
use_template = production
[server_data/mail]
use_template = production
[server_data/matrix]
use_template = production
[server_data/nextcloud]
use_template = production
[server_data/postgres]
use_template = production

#############################
# templates below this line #
#############################

[template_production]
frequently = 0
hourly = 36
daily = 30
monthly = 6
yearly = 1
autosnap = yes
autoprune = yes
{% endhighlight %}

Reload the sanoide service.

``` console
$ sudo systemctl restart sanoid.service
```

#### Backup user

The syncoid uses the ssh to perform the backup and requires a user with permission to access the ZFS binaries.
Currently, the only user with this privilege is the `root`. And for security reasons, it should not be accessible
via ssh. So we will try to mitigate that by adding a new user - called `backup-user` - and some permission over
the ZFS datasets and allowing access to only the ZFS binaries to this new user.

``` console
$ sudo useradd --user-group --create-home backup-user
```

Delegate some ZFS "power" to our new (non-super) user
[<sup>[12]</sup>](https://github.com/jimsalterjrs/sanoid/issues/522).

``` console
$ sudo zfs allow -u backup-user compression,create,destroy,mount,bookmark,hold,mountpoint,receive,rollback,send,snapshot server_data
```

To improve safety we will create a new folder to the zfs binaries, so the `backup-user` will not have access to
all binaries from the `/usr/sbin`.

``` console
$ sudo mkdir /usr/zfs-bin
$ sudo ln -s /usr/sbin/{zdb,zed,zfs,zfs_ids_to_path,zgenhostid,zhack,zpool,zstream,zstreamdump,zvol_wait,fsck.zfs,mount.zfs} /usr/zfs-bin
```

Setup the ssh files and folders.

``` console
$ sudo -u backup-user mkdir /home/backup-user/.ssh
$ sudo -u backup-user chmod 700 /home/backup-user/.ssh
$ sudo -u backup-user ssh-keygen -t ed25519
$ sudo -u backup-user cp /home/backup-user/.ssh/id_ed25519.pub /home/backup-user/.ssh/authorized_keys
$ sudo -u backup-user chmod 600 /home/backup-user/.ssh/authorized_keys
```

Safely copy the private key `id_ed25519` to the backup server.


**The rest of the configuration [backup-server]({% post_url 2021-06-30-Backup-Server %}) post, so don't forget to check it out.**{: style="color: red" }

### /var/log

Logs are very important, so would be good to store them in the ZFS Dataset to preserve their integrity. This is what this
section is about.

We want to move the `/var/log` to the log dataset at our `zpool`. To do that we need to enter in the single-user mode
by using this command[<sup>[13]</sup>](https://www.suse.com/support/kb/doc/?id=000018399):

``` console
$ sudo init 1
```

Enter the root password and copy all the data to the `server_data/log` dataset.

```console
# cp -apx /var/log/* /var/server_data/log
```

Create a backup of the `/var/log`

```console
# mv /var/log/ /var/log.old
```

Create the symlink from our ZFS log dataset to `/var/log`.

```console
# ln -s /var/server_data/log /var/log
```

Check to see if everything is good.

```console
# ls -l /var/
```

And now reboot the system.

```console
# reboot now
```

## UPS Configuration [TODO]

## Hostname

This step is necessary before we install the [iRedMail](#h-iredmail).

Add the **\<Hostname>**{: style="color: #d2929d" }, **\<Mail-Hostname>**{: style="color: #f87e19" },
and all domains that you will have like this
[<sup>[14]</sup>](https://docs.iredmail.org/install.iredmail.on.debian.ubuntu.html#preparations).

``` plain
/etc/hosts

127.0.0.1 <Mail-Hostname> localhost
127.0.1.1 <Hostname> <Domain 1> <Domain 2> ... <Domain N>
```

``` plain
/etc/hostname

<Hostname>
```

**Reboot** to apply the changes.

## PostgreSQL

I choose PostgreSQL because it is the only one compatible with all our services(IredMail, Nextcloud, Synapse
Matrix server). And that is it, there wasn't much thought in this decision.

Additionaly, we want to move the database folder from the `/usr/lib/` to the ZFS `postgres` dataset to enjoy all ZFS features.

The way that will be done is by initing a new 'Database cluster' under `/var/server_data/postgres`,
deleting the default 'Database cluster' at `/var/lib/postgresql/<version>/main`, and add a synlink from the
`/var/server_data/postgres` to `/var/lib/postgresql/<version>/main`.

### Postgress database cluster

Change the `/server_data/postgres` ownership and init the 'Database cluster'
[<sup>[15]</sup>](https://postgrespro.com/docs/postgresql/13/creating-cluster).

``` console
$ sudo chown postgres:postgres /var/server_data/postgres
$ sudo -u postgres /usr/lib/postgresql/<version>/bin/initdb -D /var/server_data/postgres
```

Add a syslink to the default folder.

``` console
$ sudo systemctl stop postgresql.service postgresql@<version>-main.service
$ sudo rm -rf /var/lib/postgresql/<version>/main
$ sudo ln -s /var/server_data/postgres /var/lib/postgresql/<version>/main
```

Check if everything is fine.

``` console
$ ls -l /var/lib/postgresql/<version>/
```

Start the Postgresql services.

``` console
$ sudo systemctl start postgresql.service postgresql@<version>-main.service
```

### Database setup

Now that we moved the 'Database cluster' to a ZFS dataset, we will create the users and databases to our Nextcloud and
Synapse services and configure the access to each database.

Setup the synapse database and user
[<sup>[16]</sup>](https://github.com/matrix-org/synapse/blob/develop/INSTALL.md#using-postgresql).

``` console
$ sudo -u postgres bash
# createuser --pwprompt <Matrix-Postgres-user>
Enter password for new role: <Matrix-database-pass>
Enter it again:  <Matrix-database-pass>
# createdb --encoding=UTF8 --locale=C --template=template0 --owner=<Matrix-Postgres-user> <Matrix-database-Name>
# exit
```

Now the nextcloud database and user[<sup>[17]</sup>](https://youtu.be/WbAAMskEQqg?t=315).

``` sql
$ sudo -u postgres psql
CREATE USER <Nextcloud-Postgres-user> WITH PASSWORD '<Nextcloud-database-pass>';
CREATE DATABASE <Nextcloud-database-Name> TEMPLATE template0 ENCODING 'UNICODE';
ALTER DATABASE <Nextcloud-database-Name> OWNER TO <Nextcloud-Postgres-user>;
GRANT ALL PRIVILEGES ON DATABASE <Nextcloud-database-Name> TO <Nextcloud-Postgres-user>;
\du          # Show users
\l           # Show the dbs
\q           # Exits
```

## IRedMail

The [IRedMail](https://www.iredmail.org/) is basically a shell script that installs and configures the
Postfix(SMTP server), Dovecote(IMAP/POP3 server), Roundcube WebMail, database, Nginx, and Fail2Ban.
It also brings a program called iRedAdmin, to manages the accounts, domains e other things.

There's another similar program called [Mail-in-a-box](https://mailinabox.email/) that looks good too.
But in this tutorial, we will stick with the IRedMail.

Access the [iredmail](https://www.iredmail.org/download.html) download page, copy the download link, and:

``` console
$ wget https://github.com/iredmail/iRedMail/archive/<iredmail-version>.tar.gz
```

Unzip.

``` console
$ tar -xvf <iredmail-version>.tar.gz
```

Init the installer.

``` console
$ chmod +x iRedMail-<iredmail-version>/iRedMail.sh
$ cd iRedMail-<iredmail-version>
$ sudo bash iRedMail.sh
```

In the wizard[<sup>[18]</sup>](https://docs.iredmail.org/install.iredmail.on.debian.ubuntu.html#screenshots-of-installation):

- `/path/to/zfs/mail`
- Choose nginx.
- Choose PostgreSQL.
- Put the **\<Nextcloud-SU-Pass>**{: style="color black" }.
- **\<Hostname>**{: style="color: #d2929d" }.
- Put the **\<Mail-Admin-Pass>**{: style="color: #00b0f0" }.
- Only select iRedAdmin and Fail2Ban.
- Yes to nftables rules with you want to use it.

**Reboot** to apply the changes.

Now you should be able to access https://**\<Hostname>**{: style="color: #d2929d" }/iredadmin.

### Disabling iRedMail backup

Since we have the ZFS+sanoid we don't need the iRedMail backup, so we can disable it.

``` console
$ sudo crontab -e
```
``` diff
--- /tmp/crontab	2021-07-26 15:30:48.890516624 -0300
+++ /tmp/crontab	2021-07-26 15:30:55.466017258 -0300
@@ -1,4 +1,4 @@
-1   3   *   *   *   /bin/bash /var/server_data/mail/backup/backup_pgsql.sh
+#1   3   *   *   *   /bin/bash /var/server_data/mail/backup/backup_pgsql.sh
 
 # iRedAPD: Clean up expired tracking records hourly.
 1   *   *   *   *   python3 /opt/iredapd/tools/cleanup_db.py >/dev/null
```

And now we need restart the `cron.service`.

``` console
$ sudo systemctl restart cron.service
```

### Changing the password scheme

In the future, we will integrate the passwords of Nextcloud and email. By allowing Nextcloud to manage the passwords
of the emails. But the app `user_sql_raw`[<sup>[19]</sup>](https://github.com/PanCakeConnaisseur/user_backend_sql_raw)
doesn't support salted SHA512 (SSHA512)
[<sup>[20]</sup>](https://doc.dovecot.org/configuration_manual/authentication/password_schemes/#salting),
we need to choose a scheme that is compatible with both, the app and the iRedAdmin.
At least for now, the Blowfish crypt (bcrypt) is the best option
[<sup>[21]</sup>](https://doc.dovecot.org/configuration_manual/authentication/password_schemes/#what-scheme-to-use).

``` diff
--- /opt/www/iredadmin/settings.py	2021-07-26 15:39:27.206712536 -0300
+++ /opt/www/iredadmin/settings.py	2021-07-26 15:39:38.758691159 -0300
@@ -111,7 +111,7 @@
 # Place your custom settings below, you can override all settings in this file
 # and libs/default_settings.py here.
 #
-DEFAULT_PASSWORD_SCHEME = 'SSHA512'
+DEFAULT_PASSWORD_SCHEME = 'BCRYPT'
 mlmmjadmin_api_auth_token = 'automaticaly-generated-token'
 fail2ban_enabled = True
 fail2ban_db_host = '127.0.0.1'
```

Restart the daemons.

``` console
$ sudo systemctl restart iredadmin.service iredapd.service
```

The rest of the configuration will happen in the [Configuring](#h-configuring-1).

### Hardening the Postfix SSL

The title says it all, we need to change some settings on `/etc/postfix/main.cf`. This change is based on the
[Mozilla SSL Configuration Generator](https://ssl-config.mozilla.org/#server=postfix&version=3.5.6-1&config=intermediate&openssl=1.1.1d&guideline=5.6).

``` diff
--- /etc/postfix/main.cf	2021-07-26 15:42:33.320486981 -0300
+++ /etc/postfix/main.cf	2021-07-26 15:43:18.653315534 -0300
@@ -98,19 +98,20 @@
 smtpd_tls_CApath = /etc/ssl/certs
 
 #
-# Disable SSLv2, SSLv3
+# Disable SSLv2, SSLv3 !TLSv1 !TLSv1.1
 #
-smtpd_tls_protocols = !SSLv2 !SSLv3
-smtpd_tls_mandatory_protocols = !SSLv2 !SSLv3
-smtp_tls_protocols = !SSLv2 !SSLv3
-smtp_tls_mandatory_protocols = !SSLv2 !SSLv3
-lmtp_tls_protocols = !SSLv2 !SSLv3
-lmtp_tls_mandatory_protocols = !SSLv2 !SSLv3
+smtpd_tls_protocols = !SSLv2 !SSLv3 !TLSv1 !TLSv1.1
+smtpd_tls_mandatory_protocols = !SSLv2 !SSLv3 !TLSv1 !TLSv1.1
+smtp_tls_protocols = !SSLv2 !SSLv3 !TLSv1 !TLSv1.1
+smtp_tls_mandatory_protocols = !SSLv2 !SSLv3 !TLSv1 !TLSv1.1
+lmtp_tls_protocols = !SSLv2 !SSLv3 !TLSv1 !TLSv1.1
+lmtp_tls_mandatory_protocols = !SSLv2 !SSLv3 !TLSv1 !TLSv1.1
+smtpd_tls_mandatory_ciphers = medium
 
 #
 # Fix 'The Logjam Attack'.
 #
-smtpd_tls_exclude_ciphers = aNULL, eNULL, EXPORT, DES, RC4, MD5, PSK, aECDH, EDH-DSS-DES-CBC3-SHA, EDH-RSA-DES-CDC3-SHA, KRB5-DE5, CBC3-SHA
+tls_medium_cipherlist = ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384
 smtpd_tls_dh512_param_file = /etc/ssl/dh512_param.pem
 smtpd_tls_dh1024_param_file = /etc/ssl/dh2048_param.pem
 
@@ -232,7 +233,7 @@
 #
 #smtpd_sasl_auth_enable = yes
 #smtpd_sasl_security_options = noanonymous
-#smtpd_tls_auth_only = yes
+smtpd_tls_auth_only = yes
 
 # hostname
 myhostname = <Hostname>
```

Restart the Postfix service.

``` console
$ sudo systemctl restart postfix.service
```

### Shared DKIM Key

DKIM keys are used to validate the domain against the DNS TXT record. This section is useful if we want more
than one domain but only don't wanna generate more DKIM keys. If you wanna separate keys follow
[this](https://docs.iredmail.org/sign.dkim.signature.for.new.domain.html#generate-new-dkim-key-for-new-mail-domain) tutorial.

To share a `dkim key` with the **\<Mail-Hostname>**{: style="color: #f87e19" }, we need to change the `amavis` config.

``` diff
--- /etc/amavis/conf.d/50-user	2021-07-26 15:44:49.363163091 -0300
+++ /etc/amavis/conf.d/50-user	2021-07-26 15:45:10.391615210 -0300
@@ -503,6 +503,7 @@
 
 # Add dkim_key here.
 dkim_key('<Mail-Hostname>', 'dkim', '/var/lib/dkim/<Mail-Hostname>.pem');
+dkim_key('<Hostname>', 'dkim', '/var/lib/dkim/<Mail-Hostname>.pem');
 
 @dkim_signature_options_bysender_maps = ({
     # 'd' defaults to a domain of an author/sender address,
```

And restart the `amavis.service`

``` console
$ sudo systemctl restart amavis
```

### Autodicover and Autoconfig

Several email clients use autoconfig(Mozilla Thunderbird) and/or autodiscover(Microsoft protocol) instead of an
SRV records to discover the mail server configuration. To make the life of the end-user easier, we need to add a tiny
PHP script to our server to allow these methods of automatic server configuration discovery.

As described at Microsoft documentation
[<sup>[22]</sup>](https://docs.microsoft.com/pt-br/Exchange/architecture/client-access/autodiscover)
the exchange client request a custom xml at `https://<SMTP-address-domain>/autodiscover/autodiscover.xml`,
and if it fails, it tries at `https://autodiscover.<SMTP-address-domain>/autodiscover/autodiscover.xml`.
The \<SMTP-address-domain> is based on the record at DNS. And we need to format the xml based on this request.
To do that we will use the project outlook-autodiscover-nginx project to do that
[<sup>[23]</sup>](https://github.com/Earl0fPudding/outlook-autodiscover-nginx).
(This project is pretty small and, as far as my PHP goes, this project looks to be safe)

Clone the repository.

``` console
$ git clone https://github.com/Earl0fPudding/outlook-autodiscover-nginx.git autoconfig
$ sudo chown www-data:www-data --recursive autoconfig
$ sudo mv autoconfig /var/www
```

Change the `config.xml` with the infomations of your mail server.

``` diff
--- a/var/www/autoconfig/config.xml
+++ b/var/www/autoconfig/config.xml
@@ -1,30 +1,30 @@
 <?xml version="1.0" encoding="UTF-8"?>
 <config>
-  <domain>example.com</domain>
-  <displayname>Example Mail</displayname>
-  <displayshortname>Example</displayshortname>
-  <usePop>false</usePop> <!-- true if you use POP3, otherwise false -->
+  <domain> <Hostname> </domain>
+  <displayname> YOUR-Mail</displayname>
+  <displayshortname>Mail</displayshortname>
+  <usePop>true</usePop> <!-- true if you use POP3, otherwise false -->
   <pop>
-    <server>mail.example.com</server> <!-- POP3 server IP or address -->
-    <port>110</port> <!-- 110 for unencrypted, 995 for encypted trafic -->
+    <server> <Mail-Hostname> </server> <!-- POP3 server IP or address -->
+    <port>995</port> <!-- 110 for unencrypted, 995 for encypted trafic -->
     <loginname>email</loginname> <!-- email for entire email+domain, username for the part before the @ character -->
-    <encryption>none</encryption> <!-- none, ssl or starttls -->
+    <encryption>ssl</encryption> <!-- none, ssl or starttls -->
     <password>unencrypted</password> <!-- unencrypted or encrypted -->
   </pop>
   <useImap>true</useImap> <!-- true if you use IMAP, otherwise false -->
   <imap>
-    <server>mail.example.com</server> <!-- IMAP server IP or address -->
+    <server> <Mail-Hostname> </server> <!-- IMAP server IP or address -->
     <port>993</port> <!-- 143 for unencrypted, 993 for encypted trafic (IMAPS) -->
     <loginname>email</loginname> <!-- email for entire email+domain, username for the part before the @ character -->
     <encryption>ssl</encryption> <!-- none, ssl or starttls -->
     <password>unencrypted</password> <!-- unencrypted or encrypted -->
   </imap>
   <smtp>
-    <server>mail.example.com</server> <!-- SMTP server IP or address -->
+    <server> <Mail-Hostname> </server> <!-- SMTP server IP or address -->
     <port>587</port> <!-- 25 for unencrypted SMTP, 465 for encypted SMTP or 587 for submission -->
     <loginname>email</loginname> <!-- email for entire email+domain, username for the part before the @ character -->
     <encryption>starttls</encryption> <!-- none, ssl or starttls -->
     <password>unencrypted</password> <!-- unencrypted or encrypted -->
   </smtp>
-  <autodiscoverAddress>https://autoconfig.example.com</autodiscoverAddress> <!-- the socket/virtual host which provides /autoconfig/autoconfig.xml -->
+  <autodiscoverAddress>https://autoconfig.<Hostname></autodiscoverAddress> <!-- the socket/virtual host which provides /autoconfig/autoconfig.xml -->
 </config>
```

Change the permission to read and execute only.

``` console
$ sudo chmod 555 --recursive /var/www/autoconfig
```

The web server configuration will happen in the [autoconfig](#h-autoconfig).

**(Optional)**

To have a fallback of the `autoconfig.php`, get the xml from the
`https://autoconfig.<Hostname>/mail/config-v1.1.xml`
 and put in the path `/var/www/<root-autoconfig>/.well-known/autoconfig/mail/config-v1.1.xml`.

Currently `/var/www/html/.well-known/autoconfig/mail/config-v1.1.xml`.

## Roundcube webmail

IMHO the Roundcube is the best of all open source webmail that I could find. I played with the
[rainloop](https://www.rainloop.net/), [afterlogic](https://afterlogic.com/), and the
[official nextcloud mail app](https://apps.nextcloud.com/apps/mail). But the Roundcube is the most complete,
and it is bundled in the iRedMail.

But I prefer to use the Debian package because it comes with more plugins, so we will follow this path here.
If you want to install it through the IredMail installer, skip to the [carddav](#h-carddav).
But may be interesting to check out the [Roundcube Configuration](#h-roundcube-configuration) too.

### Install

Install the package:

``` console
$ sudo apt install roundcube-pgsql roundcube-plugins roundcube-plugins-extra roundcube gnupg -y
```

Use the **\<Roundcube-database-Pass>**{: style="color: purple" } in the installation process.
Or leave it blank to a random password be generated by the installer.

### Roundcube Configuration

This is the Roundcube's `config.inc.php` against the default config, use it as a base to yours.

**(Attention, change the lines 159 and 199)**{: style="color: red" }

{% highlight php wl linenos %}
/etc/roundcube/config.inc.php

<?php

/*
+-----------------------------------------------------------------------+
| Local configuration for the Roundcube Webmail installation.           |
|                                                                       |
| This is a sample configuration file only containing the minimum       |
| setup required for a functional installation. Copy more options       |
| from defaults.inc.php to this file to override the defaults.          |
|                                                                       |
| This file is part of the Roundcube Webmail client                     |
| Copyright (C) 2005-2013, The Roundcube Dev Team                       |
|                                                                       |
| Licensed under the GNU General Public License version 3 or            |
| any later version with exceptions for skins & plugins.                |
| See the README file for a full license statement.                     |
+-----------------------------------------------------------------------+
*/

$config = array();

/* Do not set db_dsnw here, use dpkg-reconfigure roundcube-core to configure database ! */
include_once("/etc/roundcube/debian-db-roundcube.php");


// ----------------------------------
// LOGGING/DEBUGGING
// ----------------------------------
$config['debug_level'] = 1;

// Log SQL queries
$config['sql_debug'] = true;

// Log IMAP conversation
$config['imap_debug'] = true;

// Log LDAP conversation
$config['ldap_debug'] = true;

// Log SMTP conversation
$config['smtp_debug'] = true;

// log driver:  'syslog', 'stdout' or 'file'.
$config['log_driver'] = 'file';


// ----------------------------------
// IMAP
// ----------------------------------

// The IMAP host chosen to perform the log-in.
// Leave blank to show a textbox at login, give a list of hosts
// to display a pulldown menu or set one host as string.
// To use SSL/TLS connection, enter hostname with prefix ssl:// or tls://
// Supported replacement variables:
// %n - hostname ($_SERVER['SERVER_NAME'])
// %t - hostname without the first part
// %d - domain (http hostname $_SERVER['HTTP_HOST'] without the first part)
// %s - domain name after the '@' from e-mail address provided at login screen
// For example %n = mail.domain.tld, %t = domain.tld
$config['default_host'] = '127.0.0.1';

// TCP port used for IMAP connections
$config['default_port'] = 143;

// IMAP authentication method (DIGEST-MD5, CRAM-MD5, LOGIN, PLAIN or null).
// Use 'IMAP' to authenticate with IMAP LOGIN command.
// By default the most secure method (from supported) will be selected.
$config['imap_auth_type'] = 'LOGIN';

// If you know your imap's folder delimiter, you can specify it here.
// Otherwise it will be determined automatically
$config['imap_delimiter'] = '/';
// Required if you're running PHP 5.6 or later

// IMAP socket context options
// See http://php.net/manual/en/context.ssl.php
// The example below enables server certificate validation
//$config['imap_conn_options'] = array(
//  'ssl'         => array(
//     'verify_peer'  => true,
//     'verify_depth' => 3,
//     'cafile'       => '/etc/openssl/certs/ca.crt',
//   ),
// );
$config['imap_conn_options'] = array(
    'ssl' => array(
        'verify_peer'  => false,
        'verify_peer_name' => false,
    ),
);


// ----------------------------------
// SMTP
// ----------------------------------

// SMTP server host (for sending mails).
// Enter hostname with prefix tls:// to use STARTTLS, or use
// prefix ssl:// to use the deprecated SSL over SMTP (aka SMTPS)
// Supported replacement variables:
// %h - user's IMAP hostname
// %n - hostname ($_SERVER['SERVER_NAME'])
// %t - hostname without the first part
// %d - domain (http hostname $_SERVER['HTTP_HOST'] without the first part)
// %z - IMAP domain (IMAP hostname without the first part)
// For example %n = mail.domain.tld, %t = domain.tld
$config['smtp_server'] = 'tls://127.0.0.1';

// SMTP port (default is 25; use 587 for STARTTLS or 465 for the
// deprecated SSL over SMTP (aka SMTPS))
$config['smtp_port'] = 587;

// SMTP username (if required) if you use %u as the username Roundcube
// will use the current username for login
$config['smtp_user'] = '%u';

// SMTP password (if required) if you use %p as the password Roundcube
// will use the current user's password for login
$config['smtp_pass'] = '%p';

// SMTP AUTH type (DIGEST-MD5, CRAM-MD5, LOGIN, PLAIN or empty to use
// best server supported one)
$config['smtp_auth_type'] = 'LOGIN';

// SMTP socket context options
// See http://php.net/manual/en/context.ssl.php
// The example below enables server certificate validation, and
// requires 'smtp_timeout' to be non zero.
// $config['smtp_conn_options'] = array(
//   'ssl'         => array(
//     'verify_peer'  => true,
//     'verify_depth' => 3,
//     'cafile'       => '/etc/openssl/certs/ca.crt',
//   ),
// );
$config['smtp_conn_options'] = array(
    'ssl' => array(
        'verify_peer'      => false,
        'verify_peer_name' => false,
    ),
);


// ----------------------------------
// CACHE(S)
// ----------------------------------

// Use these hosts for accessing Redis.
// Currently only one host is supported. Cluster support may come in a future release.
// You can pass 4 fields, host, port (optional), database (optional) and password (optional).
// Unset fields will be set to the default values host=127.0.0.1, port=6379.
// Examples:
//     array('localhost:6379');
//     array('192.168.1.1:6379:1:secret');
//     array('unix:///var/run/redis/redis-server.sock:1:secret');
$config['redis_hosts'] = array('unix:///var/run/redis/redis-server.sock:<Redis-Password>');


// ----------------------------------
// SYSTEM
// ----------------------------------

// Enables possibility to log in using email address from user identities
$config['user_aliases'] = true;

// Name your service. This is displayed on the login screen and in the window title
$config['product_name'] = 'Caedrium Webmail powered by Roundcube';

// Enforce connections over https
// With this option enabled, all non-secure connections will be redirected.
// It can be also a port number, hostname or hostname:port if they are
// different than default HTTP_HOST:443
$config['force_https'] = true;

// Allow browser-autocompletion on login form.
// 0 - disabled, 1 - username and host only, 2 - username, host, password
$config['login_autocomplete'] = 2;

// check client IP in session authorization
$config['ip_check'] = true;

// provide an URL where a user can get support for this Roundcube installation
// PLEASE DO NOT LINK TO THE ROUNDCUBE.NET WEBSITE HERE!
$config['support_url'] = '';

// Use user's identity as envelope sender for 'return receipt' responses,
// otherwise it will be rejected by iRedAPD plugin `reject_null_sender`.
// According to RFC2298, return receipt envelope sender address must be empty.
// If this option is true, Roundcube will use user's identity as envelope sender for MDN responses.
$config['mdn_use_from'] = true;

// this key is used to encrypt the users imap password which is stored
// in the session record (and the client cookie if remember password is enabled).
// please provide a string of exactly 24 chars.
// YOUR KEY MUST BE DIFFERENT THAN THE SAMPLE VALUE FOR SECURITY REASONS
$config['des_key'] = '8HjBAnpzqXgrquTFx4uh5WW9'; // READ THE COMMENT ABOVE

// Encryption algorithm. You can use any method supported by OpenSSL.
// Default is set for backward compatibility to DES-EDE3-CBC,
// but you can choose e.g. AES-256-CBC which we consider a better choice.
$config['cipher_method'] = 'AES-256-CBC';

// Add this user-agent to message headers when sending
$config['useragent'] = 'Roundcube Webmail'; // Hide version number

// Automatically add this domain to user names for login
// Only for IMAP servers that require full e-mail addresses for login
// Specify an array with 'host' => 'domain' values to support multiple hosts
// Supported replacement variables:
// %h - user's IMAP hostname
// %n - hostname ($_SERVER['SERVER_NAME'])
// %t - hostname without the first part
// %d - domain (http hostname $_SERVER['HTTP_HOST'] without the first part)
// %z - IMAP domain (IMAP hostname without the first part)
//$config['username_domain'] = '<Mail-Hostname>';

// Absolute path to a local mime.types mapping table file.
// This is used to derive mime-types from the filename extension or vice versa.
// Such a file is usually part of the apache webserver. If you don't find a file named mime.types on your system,
$config['mime_types'] = '/etc/mime.types';

// Message size limit. Note that SMTP server(s) may use a different value.
// This limit is verified when user attaches files to a composed message.
// Size in bytes (possible unit suffix: K, M, G)
$config['max_message_size'] = '15M';

// Logo image replacement. Specifies location of the image as:
// - URL relative to the document root of this Roundcube installation
// - full URL with http:// or https:// prefix
// - URL relative to the current skin folder (when starts with a '/')
//
// An array can be used to specify different logos for specific template files
// The array key specifies the place(s) the logo should be applied to and
// is made up of (up to) 3 parts:
// - skin name prefix (always with colon, can be replaced with *)
// - template name (or * for all templates)
// - logo type - it is used for logos used on multiple templates
//   the available types include '[favicon]' for favicon, '[print]' for logo on all print
//   templates (e.g. messageprint, contactprint) and '[small]' for small screen logo in supported skins
//
// Example config for skin_logo
/*
   array(
     // show the image /images/logo_login_small.png for the Login screen in the Elastic skin on small screens
     "elastic:login[small]" => "/images/logo_login_small.png",
     // show the image /images/logo_login.png for the Login screen in the Elastic skin
     "elastic:login" => "/images/logo_login.png",
     // show the image /images/logo_small.png in the Elastic skin
     "elastic:*[small]" => "/images/logo_small.png",
     // show the image /images/larry.png in the Larry skin
     "larry:*" => "/images/larry.png",
     // show the image /images/logo_login.png on the login template in all skins
     "login" => "/images/logo_login.png",
     // show the image /images/logo_print.png for all print type logos in all skins
     "[print]" => "/images/logo_print.png",
   );
*/
$config['skin_logo'] = null;


// ----------------------------------
// PLUGINS
// ----------------------------------

// List of active plugins (in plugins/ directory)
$config['plugins'] = array('managesieve', 'zipdownload','emoticons',
                           'html5_notifier', 'identicon', 'fail2ban',
                           'vcard_attachments', 'thunderbird_labels',
                           'newmail_notifier','newmail_notifier',
                           'keyboard_shortcuts','contextmenu',
                           'subscriptions_option');

// ----------------------------------
// USER INTERFACE
// ----------------------------------

// Default language
$config['language'] = 'pt_BR';

// use this format for date display (date or strftime format)
$config['date_format'] = 'd/m/Y';

// use this format for detailed date/time formatting (derived from date_format and time_format)
$config['date_long'] = 'H:i d/m/Y';

// Disable spellchecking
// Debian: spellshecking needs additional packages to be installed, or calling external APIs
//         see defaults.inc.php for additional informations
$config['enable_spellcheck'] = false;

// automatically create the above listed default folders on user login
$config['create_default_folders'] = true;

// if in your system 0 quota means no limit set this option to true 
$config['quota_zero_as_unlimited'] = true;

// Set the spell checking engine. Possible values:
// - 'googie'  - the default (also used for connecting to Nox Spell Server, see 'spellcheck_uri' setting)
// - 'pspell'  - requires the PHP Pspell module and aspell installed
// - 'enchant' - requires the PHP Enchant module
// - 'atd'     - install your own After the Deadline server or check with the people at http://www.afterthedeadline.com before using their API
// Since Google shut down their public spell checking service, the default settings
// connect to http://spell.roundcube.net which is a hosted service provided by Roundcube.
// You can connect to any other googie-compliant service by setting 'spellcheck_uri' accordingly.
$config['spellcheck_engine'] = 'pspell';

// ----------------------------------
// USER PREFERENCES
// ----------------------------------

// skin name: folder from skins/
$config['skin'] = 'elastic';

// Use this charset as fallback for message decoding
$config['default_charset'] = 'UTF-8';

// sort contacts by this col (preferably either one of name, firstname, surname)
//$config['addressbook_sort_col'] = 'name';

// save compose message every 300 seconds (5min)
$config['draft_autosave'] = 60;

// Default messages listing mode. One of 'threads' or 'list'.
$config['default_list_mode'] = 'threads';

// 0 - Do not expand threads
// 1 - Expand all threads automatically
// 2 - Expand only threads with unread messages
$config['autoexpand_threads'] = 2;

// If true all folders will be checked for recent messages
$config['check_all_folders'] = true;

// Default font size for composed HTML message.
$config['default_font_size'] = '12pt';

// Enables display of email address with name instead of a name (and address in title)
$config['message_show_email'] = true;

// Interface layout. Default: 'widescreen'.
//  'widescreen' - three columns
//  'desktop'    - two columns, preview on bottom
//  'list'       - two columns, no preview
$config['layout'] = 'widescreen';   // three columns

// Set true if deleted messages should not be displayed
// This will make the application run slower
//$config['skip_deleted'] = true;
{% endhighlight %}

### Plugins Configuration

Because we installed the Roundcube manually, we have to configure some plugins manually too.

#### ZipDownload

{% highlight php wl linenos %}
/etc/roundcube/plugins/zipdownload/config.inc.php

// Zip attachments// Only show the link when there are more than this many attachments
// -1 to prevent downloading of attachments as zip
$config['zipdownload_attachments'] = 1;

// Zip selection of mail messages
// This option enables downloading of multiple messages as one zip archive.
// The number or string value specifies maximum total size of all messages
// in the archive (not the size of the archive itself).
$config['zipdownload_selection'] = '50MB';

// Charset to use for filenames inside the zip
$config['zipdownload_charset'] = 'ISO-8859-1';
 ?>
{% endhighlight %}

####  Thunderbird labels

{% highlight php wl linenos %}
/etc/roundcube/plugins/thunderbird_labels/config.inc.php

 <?php
// whether to globally enable thunderbird labels
$rcmail_config['tb_label_enable'] = true;
// add labels to contextmenu (if contextmenu plugin is present)
$rcmail_config['tb_label_enable_contextmenu'] = true;
// enable kb shortcuts (1-5)
$rcmail_config['tb_label_enable_shortcuts'] = true;
// users can modify labels
$rcmail_config['tb_label_modify_labels'] = true;
// style for UI: 'bullets' or 'thunderbird'
$rcmail_config['tb_label_style'] = "bullets";
// custom hidden flags
$rcmail_config['tb_label_hidden_flags'] = array();
 ?>
{% endhighlight %}

####  Managesieve

{% highlight php wl linenos %}
/etc/roundcube/plugins/managesieve/config.inc.php

 <?php
// managesieve server port. When empty the port will be determined automatically
// using getservbyname() function, with 4190 as a fallback.
$config['managesieve_port'] = 4190;

// managesieve server address, default is localhost.
// Replacement variables supported in host name:
// %h - user's IMAP hostname
// %n - http hostname ($_SERVER['SERVER_NAME'])
// %d - domain (http hostname without the first part)
// For example %n = mail.domain.tld, %d = domain.tld
$config['managesieve_host'] = "127.0.0.1";

// authentication method. Can be CRAM-MD5, DIGEST-MD5, PLAIN, LOGIN, EXTERNAL
// or none. Optional, defaults to best method supported by server.
$config['managesieve_auth_type'] = "LOGIN";

// Optional managesieve authentication identifier to be used as authorization proxy.
// Authenticate as a different user but act on behalf of the logged in user.
// Works with PLAIN and DIGEST-MD5 auth.
$config['managesieve_auth_cid'] = null;
// Optional managesieve authentication password to be used for imap_auth_cid
$config['managesieve_auth_pw'] = null;

// use or not TLS for managesieve server connection
// Note: tls:// prefix in managesieve_host is also supported
$config['managesieve_usetls'] = true;

// Connection scket context options
// See http://php.net/manual/en/context.ssl.php
// The example below enables server certificate validation
//$config['managesieve_conn_options'] = array(
//  'ssl'         => array(
//     'verify_peer'  => true,
//     'verify_depth' => 3,
//     'cafile'       => '/etc/openssl/certs/ca.crt',
//   ),
// );
// Note: These can be also specified as an array of options indexed by hostname
$config['managesieve_conn_options'] = array("ssl" => array("verify_peer" => false, "verify_peer_name" => false));

// A file with default script content (eg. spam filter)
$config['managesieve_default'] = "";

// The name of the script which will be used when there's no user script
$config['managesieve_script_name'] = 'managesieve';

// Sieve RFC says that we should use UTF-8 endcoding for mailbox names,
// but some implementations does not covert UTF-8 to modified UTF-7.
// Defaults to UTF7-IMAP
$config['managesieve_mbox_encoding'] = 'UTF-8';

// I need this because my dovecot (with listescape plugin) uses
// ':' delimiter, but creates folders with dot delimiter
$config['managesieve_replace_delimiter'] = '';

// disabled sieve extensions (body, copy, date, editheader, encoded-character,
// envelope, environment, ereject, fileinto, ihave, imap4flags, index,
// mailbox, mboxmetadata, regex, reject, relational, servermetadata,
// spamtest, spamtestplus, subaddress, vacation, variables, virustest, etc.
// Note: not all extensions are implemented
$config['managesieve_disabled_extensions'] = array();

// Enables debugging of conversation with sieve server. Logs it into <log_dir>/sieve
$config['managesieve_debug'] = false;

// Enables features described in http://wiki.kolab.org/KEP:14
$config['managesieve_kolab_master'] = false;

// Script name extension used for scripts including. Dovecot uses '.sieve',
// Cyrus uses '.siv'. Doesn't matter if you have managesieve_kolab_master disabled.
$config['managesieve_filename_extension'] = '.sieve';

// List of reserved script names (without extension).
// Scripts listed here will be not presented to the user.
$config['managesieve_filename_exceptions'] = array();

// List of domains limiting destination emails in redirect action
// If not empty, user will need to select domain from a list
$config['managesieve_domains'] = array();

// Default list of entries in header selector
$config['managesieve_default_headers'] = "";

// Enables separate management interface for vacation responses (out-of-office)
// 0 - no separate section (default),
// 1 - add Vacation section,
// 2 - add Vacation section, but hide Filters section
$config['managesieve_vacation'] = 1;

// Enables separate management interface for setting forwards (redirect to and copy to)
// 0 - no separate section (default),
// 1 - add Forward section,
// 2 - add Forward section, but hide Filters section
$config['managesieve_forward'] = 0;

// Default vacation interval (in days).
// Note: If server supports vacation-seconds extension it is possible
// to define interval in seconds here (as a string), e.g. "3600s".
$config['managesieve_vacation_interval'] = 1;

// Some servers require vacation :addresses to be filled with all
// user addresses (aliases). This option enables automatic filling
// of these on initial vacation form creation.
$config['managesieve_vacation_addresses_init'] = 1;

// Sometimes you want to always reply with mail email address
// This option enables automatic filling of :from field on initial vacation form creation.
$config['managesieve_vacation_from_init'] = 1;

// Supported methods of notify extension. Default: 'mailto'
$config['managesieve_notify_methods'] = array('mailto');

// Enables scripts RAW editor feature
$config['managesieve_raw_editor'] = true;

// Disabled actions
// Prevent user from performing specific actions:
// list_sets, enable_disable_set, delete_set, new_set, download_set, new_rule, delete_rule
// Note: disabling list_sets removes the Filter sets widget from the UI and means
//       the set defined in managesieve_script_name will always be used (and activated)
$config['managesieve_disabled_actions'] = array();

// List of hosts that support managesieve.
// Activate managesieve for selected hosts only. If this is not set all hosts are allowed.
// Example: $config['managesieve_allowed_hosts'] = array('host1.mydomain.com','host2.mydomain.com');
$config['managesieve_allowed_hosts'] = null;
 ?>
{% endhighlight %}

####  Carddav

Cardav plugin will allow us to deeply integrate the Roundcube to the Nextcloud, by allowing us to
access the Nextcloud contacts inside the Rounducube. I also tried the
[roundcube-nextcloud_sql_addressbook](https://github.com/cweiske/roundcube-nextcloud_sql_addressbook)
plugin, but it didn't work for me :(.

The Carddav plugin isn't available in the Debian repo so we will need to install and configure it
[<sup>[24]</sup>](https://help.nextcloud.com/t/how-connect-nextcolud-address-book-to-roundcube-address-book-server-side-no-user-intervention/52882/4)
[<sup>[25]</sup>](https://github.com/mstilkerich/rcmcarddav).

``` console
$ wget https://github.com/mstilkerich/rcmcarddav/releases/download/<Version>/carddav-<Version>.tar.gz
$ tar -zxf carddav-<version>.tar.gz
$ sudo mv carddav /usr/share/roundcube/plugins
$ sudo mkdir /etc/roundcube/plugins/carddav
$ sudo ln -s /usr/share/roundcube/plugins/carddav /var/lib/roundcube/plugins
$ sudo cp /usr/share/roundcube/plugins/carddav/config.inc.php.dist
/etc/roundcube/plugins/carddav/config.inc.php
$ sudo ln -s /etc/roundcube/plugins/carddav/config.inc.php /usr/share/roundcube/plugins/carddav/.
```

And configure the plug-in.

``` diff
--- /etc/roundcube/plugins/carddav/config.inc.php	2021-07-26 16:02:20.213277523 -0300
+++ /etc/roundcube/plugins/carddav/config.inc.php	2021-07-26 16:04:35.917870027 -0300
@@ -32,30 +32,36 @@
 //// ** ADDRESSBOOK PRESETS
 
 // Each addressbook preset takes the following form:
-/*
+
 $prefs['<Presetname>'] = [
     // required attributes
-    'name'         =>  '<Addressbook Name>',
-    'url'          =>  '<CardDAV Discovery URL>',
+    'name'         =>  '<Your cloud name>',
 
-    // required attributes unless passwordless authentication is used (Kerberos)
-    'username'     =>  '<CardDAV Username>',
-    'password'     =>  '<CardDAV Password>',
-
-    // optional attributes
-    'active'       =>  <true or false>,
-    'readonly'     =>  <true or false>,
-    'refresh_time' => '<Refresh Time in Hours, Format HH[:MM[:SS]]>',
+    // %u: Replaced by the full roundcube username
+    // %l: Replaced by the local part of the roundcube username if it is an email address :
+    // (Example: Roundcube username theuser@example.com - %l is replaced with theuser)
+    // %d: Replaced by the domain part of the roundcube username if it is an email address
+    // (Example: Roundcube username theuser@example.com - %d is replaced with example.com)
+    // %V: Replaced by the roundcube username, with @ and . characters substituted by _.
+    // (Example: Roundcube username user.name@example.com - %V is replaced with user_name_example_com)
+    'url'          =>  'https://<Hostname>/nextcloud/remote.php/dav/addressbooks/users/%l/contacts/',
+    'username'     =>  '%l',
+    // will be substituted for the roundcube password
+    'password'     =>  '%p',
+
+     // optional attributes
+    'active'       =>  true,
+    'readonly'     =>  false,
+    'refresh_time' => '01:00:00',
 
     // attributes that are fixed (i.e., not editable by the user) and auto-updated for this preset
-    'fixed'        =>  [ < 0 or more of the other attribute keys > ],
+    'fixed'        =>  [ 'Username' ],
 
     // always require these attributes, even for addressbook view
     'require_always' => ['email'],
 
     // hide this preset from CardDAV preferences section so users can't even see it
-    'hide' => <true or false>,
+    'hide' => true,
 ];
-*/
 
 // vim: ts=4:sw=4:expandtab:fenc=utf8:ff=unix:tw=120
```

Add to the rouncube main config.

``` diff
--- /etc/roundcube/config.inc.php	2021-07-26 16:06:51.596870612 -0300
+++ /etc/roundcube/config.inc.php	2021-07-26 16:07:22.088749824 -0300
@@ -237,7 +237,7 @@
                            'vcard_attachments', 'thunderbird_labels',
                            'newmail_notifier','newmail_notifier',
                            'keyboard_shortcuts','contextmenu',
-                           'subscriptions_option');
+                           'subscriptions_option', 'carddav');
 
 
 // ----------------------------------
```

Your contacts tab should look like the image in the [Gallery](#h-gallery).

## Nginx

The IRedMail configures the nginx, and we will follow its configurations to some extent. But some changes need to be
made, and new files need to be created. Since we will use the subdomains to each service instead of subdirs.

### Iredadmin

{: .box-note}
**Note:** If you wanna expose the iRedAmin to the internet, comment the lines 12-15.

{% highlight nginx wl linenos %}
/etc/nginx/sites-available/iredadmin.conf

server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name admin.<hostname> www.admin.<hostname> localhost;

    root /var/www/html;
    index index.php index.html;

    # Access control
    allow 127.0.0.1;
    allow ::1;
    deny all;

    include /etc/nginx/templates/misc.tmpl;
    include /etc/nginx/templates/ssl.tmpl;
    include /etc/nginx/templates/iredadmin-subdomain.tmpl;
    include /etc/nginx/templates/php-catchall.tmpl;
    include /etc/nginx/templates/stub_status.tmpl;
}
{% endhighlight %}

### Roundcube

{% highlight nginx wl linenos %}
/etc/nginx/sites-available/roundcube.conf

server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name www.<Mail-Hostname> <Mail-Hostname>;

    root /var/www/html;
    index index.php index.html;

    include /etc/nginx/templates/misc.tmpl;
    include /etc/nginx/templates/ssl.tmpl;
    include /etc/nginx/templates/roundcube-subdomain.tmpl;
    include /etc/nginx/templates/php-catchall.tmpl;
    include /etc/nginx/templates/stub_status.tmpl;
}
{% endhighlight %}

### Matrix

We will not expose the matrix server directly. Instead, we will use a reverse proxy to expose it
[<sup>[26]</sup>](https://github.com/matrix-org/synapse/blob/develop/docs/reverse_proxy.md)
[<sup>[27]</sup>](https://github.com/matrix-org/synapse/blob/develop/INSTALL.md#client-well-known-uri).

{% highlight nginx wl linenos %}
/etc/nginx/sites-available/matrix.conf

server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;

    # For the federation port
    listen 8448 ssl http2 default_server;
    listen [::]:8448 ssl http2 default_server;

    server_name matrix.<Hostname>  www.matrix.<Hostname>;

    root /var/www/html;
    index index.html index.htm;

    include /etc/nginx/templates/misc.tmpl;
    include /etc/nginx/templates/ssl.tmpl;

    location /.well-known/matrix/client {
        return 200 '{ "m.homeserver": { "base_url": "https://matrix.<Hostname>" } }';
        default_type application/json;
        add_header Access-Control-Allow-Origin *;
    }

    location /.well-known/matrix/server {
      return 200 '{"m.server": "matrix.<Hostname>:443"}';
      add_header Content-Type application/json;
    }

    location ~* ^(\/_matrix|\/_synapse\/client) {
        proxy_pass http://localhost:8008;
        proxy_set_header X-Forwarded-For $remote_addr;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header Host $host;

        # Nginx by default only allows file uploads up to 1M in size
        # Increase client_max_body_size to match max_upload_size defined in homeserver.yaml
        client_max_body_size 100M;
    }

    location /_synapse/admin {
        allow 127.0.0.1;
        allow ::1;
        deny all;
    }
}
{% endhighlight %}

### Autoconfig

The config to our autodiscover/autoconfig service.

{% highlight nginx wl linenos %}
/etc/nginx/sites-available/autoconfig.conf

server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name autodiscover.<Hostname>  autoconfig.<Hostname>;

    root /var/www/autoconfig;
    index index.php index.html;

    location ~ /(?:a|A)utodiscover/(?:a|A)utodiscover.xml {
        fastcgi_split_path_info ^(.+\.php)(/.+)$;
        fastcgi_pass php-handler;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        try_files /autodiscover-xml.php =404;
    }

    location ~ /(?:a|A)utodiscover/(?:a|A)utodiscover.json {
        fastcgi_split_path_info ^(.+\.php)(/.+)$;
        fastcgi_pass php-handler;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        try_files /autodiscover-json.php =404;
    }

    location ~* ^/mail/config-v1.1.xml {
        fastcgi_split_path_info ^(.+\.php)(/.+)$;
        fastcgi_pass php-handler;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        try_files /autoconfig.php =404;
    }

    location ~ ^/(LICENSE|README.md|.git|autodiscover-json.php|autoconfig.php|config.xml)/.* { deny all; }

    include /etc/nginx/templates/misc.tmpl;
    include /etc/nginx/templates/ssl.tmpl;
}
{% endhighlight %}

### SSL

Here we are hardening the Nginx SSL template, again, based on the
[Mozilla SSL Configuration Generator](https://ssl-config.mozilla.org/#server=nginx&version=1.17.7&config=modern&openssl=1.1.1d&guideline=5.6).

This template will be used in all of our subdomains. All the Nginx that contains the line
`include /etc/nginx/templates/ssl.tmpl;` is using this template.

{: .box-warning}
**Warning:** This configuration may not work with old browsers.

``` diff
--- /etc/nginx/templates/ssl.tmpl	2021-07-26 16:10:46.172714064 -0300
+++ /etc/nginx/templates/ssl.tmpl	2021-07-26 16:10:49.964724285 -0300
@@ -1,9 +1,26 @@
-ssl_protocols TLSv1.2;
+ssl_protocols TLSv1.3;
 
 # Fix 'The Logjam Attack'.
-ssl_ciphers EECDH+AESGCM:EDH+AESGCM:AES256+EECDH:AES256+EDH;
-ssl_prefer_server_ciphers on;
 ssl_dhparam /etc/ssl/dh2048_param.pem;
+ssl_session_timeout 1d;
+ssl_session_cache shared:MozSSL:10m;  # about 40000 sessions
+ssl_session_tickets off;
+ssl_prefer_server_ciphers off;
+
+# OCSP stapling
+ssl_stapling on;
+ssl_stapling_verify on;
+
+# verify chain of trust of OCSP response using Root CA and Intermediate certs
+ssl_trusted_certificate /etc/ssl/certs/iRedMail.crt;
+
+# HSTS settings
+# WARNING: Only add the preload option once you read about
+# the consequences in https://hstspreload.org/. This option
+# will add the domain to a hardcoded list that is shipped
+# in all major browsers and getting removed from this list
+# could take several months.
+#add_header Strict-Transport-Security "max-age=15768000; includeSubDomains; preload;" always;
 
 # To use your own ssl cert (e.g. "Let's Encrypt"), please create symbol link to
 # ssl cert/key used below, so that we can manage this config file with Ansible.
```

### Roundcube-subdomain

{: .box-note}
**Note:** Skip this if you installed the RoundCube using the iRedMail installer.

``` diff
--- /etc/nginx/templates/roundcube-subdomain.tmpl	2021-07-26 16:11:42.156898885 -0300
+++ /etc/nginx/templates/roundcube-subdomain.tmpl	2021-07-26 16:12:37.205146825 -0300
@@ -14,13 +14,13 @@
 location ~ ^/plugins/enigma/home($|/.*) { deny all; }
 
 location / {
-    root    /opt/www/roundcubemail;
+    root    /var/lib/roundcube;
     index   index.php index.html;
     include /etc/nginx/templates/hsts.tmpl;
 }
 
 location ~ \.php$ {
-    root /opt/www/roundcubemail;
+    root    /var/lib/roundcube;
     include /etc/nginx/templates/fastcgi_php.tmpl;
-    fastcgi_param SCRIPT_FILENAME /opt/www/roundcubemail$fastcgi_script_name;
+    fastcgi_param SCRIPT_FILENAME /var/lib/roundcube$fastcgi_script_name;
 }
```

### 00-default-ssl

Our setup will use subdomains instead of subdirs. Therefore, this config is useless for us.

``` diff
--- /etc/nginx/sites-available/00-default-ssl.conf	2021-07-26 16:11:56.556957696 -0300
+++ /etc/nginx/sites-available/00-default-ssl.conf	2021-07-26 16:12:53.341230946 -0300
@@ -2,20 +2,3 @@
 # Note: This file must be loaded before other virtual host config files,
 #
 # HTTPS
-server {
-    listen 443 ssl http2;
-    listen [::]:443 ssl http2;
-    server_name _;
-
-    root /var/www/html;
-    index index.php index.html;
-
-    include /etc/nginx/templates/misc.tmpl;
-    include /etc/nginx/templates/ssl.tmpl;
-    include /etc/nginx/templates/iredadmin.tmpl;
-    include /etc/nginx/templates/roundcube.tmpl;
-    include /etc/nginx/templates/sogo.tmpl;
-    include /etc/nginx/templates/netdata.tmpl;
-    include /etc/nginx/templates/php-catchall.tmpl;
-    include /etc/nginx/templates/stub_status.tmpl;
-}
```

### Nextcloud

This is based on the
[official Nginx configuration](https://docs.nextcloud.com/server/latest/admin_manual/installation/nginx.html)
at Nextcloud docs with few adjustments. The adjustments include the additions of lines 58-66 and modifications
of lines 92 to fix a certbot error, 124 and 130 to fix the load problem with the Roundcube page inside the
[Plugin](#h-roundcube-integration).

{% highlight nginx wl linenos %}
/etc/nginx/sites-available/nextcloud.conf

upstream php-handler {
    server 127.0.0.1:9999;
    #server unix:/var/run/php/php7.4-fpm.sock;
}

server {
    listen 443      ssl http2;
    listen [::]:443 ssl http2;
    server_name <Hostname> www.<Hostname>;

    include /etc/nginx/templates/ssl.tmpl;

    # set max upload size
    client_max_body_size 512M;
    fastcgi_buffers 64 4K;

    # Enable gzip but do not remove ETag headers
    gzip on;
    gzip_vary on;
    gzip_comp_level 4;
    gzip_min_length 256;
    gzip_proxied expired no-cache no-store private no_last_modified no_etag auth;
    gzip_types application/atom+xml application/javascript application/json application/ld+json application/manifest+json application/rss+xml application/vnd.geo+json application/vnd.ms-fontobject application/x-font-ttf application/x-web-app-manifest+json application/xhtml+xml application/xml font/opentype image/bmp image/svg+xml image/x-icon text/cache-manifest text/css text/plain text/vcard text/vnd.rim.location.xloc text/vtt text/x-component text/x-cross-domain-policy;

    # Pagespeed is not supported by Nextcloud, so if your server is built
    # with the `ngx_pagespeed` module, uncomment this line to disable it.
    #pagespeed off;

    # HTTP response headers borrowed from Nextcloud `.htaccess`
    add_header Referrer-Policy                      "no-referrer"   always;
    add_header X-Content-Type-Options               "nosniff"       always;
    add_header X-Download-Options                   "noopen"        always;
    add_header X-Frame-Options                      "SAMEORIGIN"    always;
    add_header X-Permitted-Cross-Domain-Policies    "none"          always;
    add_header X-Robots-Tag                         "none"          always;
    add_header X-XSS-Protection                     "1; mode=block" always;

    # Remove X-Powered-By, which is an information leak
    fastcgi_hide_header X-Powered-By;

    # Path to the root of your installation
    root /var/www/nextcloud;

    # Specify how to handle directories -- specifying `/index.php$request_uri`
    # here as the fallback means that Nginx always exhibits the desired behaviour
    # when a client requests a path that corresponds to a directory that exists
    # on the server. In particular, if that directory contains an index.php file,
    # that file is correctly served; if it doesn't, then the request is passed to
    # the front-end controller. This consistent behaviour means that we don't need
    # to specify custom rules for certain paths (e.g. images and other assets,
    # `/updater`, `/ocm-provider`, `/ocs-provider`), and thus
    # `try_files $uri $uri/ /index.php$request_uri`
    # always provides the desired behaviour.
    index index.php index.html /index.php$request_uri;

    location /roundcube/ {
        proxy_set_header X-Forwarded-Host $host:$server_port;
        proxy_set_header X-Forwarded-Server $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        add_header Front-End-Https on;

        proxy_ssl_protocols TLSv1.3;
        proxy_pass https://<Mail-Hostname>/;
    }

    # Rule borrowed from `.htaccess` to handle Microsoft DAV clients
    location = / {
        if ( $http_user_agent ~ ^DavClnt ) {
            return 302 /remote.php/webdav/$is_args$args;
        }
    }

    location = /robots.txt {
        allow all;
        log_not_found off;
        access_log off;
    }

    # Make a regex exception for `/.well-known` so that clients can still
    # access it despite the existence of the regex rule
    # `location ~ /(\.|autotest|...)` which would otherwise handle requests
    # for `/.well-known`.
    location ^~ /.well-known {
        # The following 6 rules are borrowed from `.htaccess`

        location = /.well-known/carddav     { return 301 /remote.php/dav/; }
        location = /.well-known/caldav      { return 301 /remote.php/dav/; }
        # Anything else is dynamically handled by Nextcloud
        location ^~ /.well-known            { return 301 /index.php$uri; }
        location /.well-known/acme-challenge/ { allow all; }

        try_files $uri $uri/ =404;
    }

    # Rules borrowed from `.htaccess` to hide certain paths from clients
    location ~ ^/(?:build|tests|config|lib|3rdparty|templates|data)(?:$|/)  { return 404; }
    location ~ ^/(?:\.|autotest|occ|issue|indie|db_|console)              { return 404; }

    # Ensure this block, which passes PHP files to the PHP process, is above the blocks
    # which handle static assets (as seen below). If this block is not declared first,
    # then Nginx will encounter an infinite rewriting loop when it prepends `/index.php`
    # to the URI, resulting in a HTTP 500 error response.
    location ~ \.php(?:$|/) {
        fastcgi_split_path_info ^(.+?\.php)(/.*)$;
        set $path_info $fastcgi_path_info;

        try_files $fastcgi_script_name =404;

        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_param PATH_INFO $path_info;
        fastcgi_param HTTPS on;

        fastcgi_param modHeadersAvailable true;         # Avoid sending the security headers twice
        fastcgi_param front_controller_active true;     # Enable pretty urls
        fastcgi_pass php-handler;

        fastcgi_intercept_errors on;
        fastcgi_request_buffering off;
    }

    location ~ (!/roundcube/)\.(?:css|js|svg|gif)$ {
        try_files $uri /index.php$request_uri;
        expires 6M;         # Cache-Control policy borrowed from `.htaccess`
        access_log off;     # Optional: Don't log access to assets
    }

    location ~ (!/roundcube/)\.woff2?$ {
        try_files $uri /index.php$request_uri;
        expires 7d;         # Cache-Control policy borrowed from `.htaccess`
        access_log off;     # Optional: Don't log access to assets
    }

    location / {
        try_files $uri $uri/ /index.php$request_uri;
    }
}
{% endhighlight %}

Add symlink to sites-enable.

``` console
$ sudo ln -s /etc/nginx/sites-available/*.conf /etc/nginx/sites-enabled/
```

Test and apply the configuration (Ignore the warnings related with the `ssl_stapling` for now).

``` console
$ sudo nginx -t
$ sudo nginx -s reload
```

## PHP

The default PHP configurations are not enough for Nextcloud. Nextcloud will warns us about those problems, but
we will ajust them before the Nextcloud complains
[<sup>[28]</sup>](https://wiki.learnlinux.tv/index.php/Nextcloud_-_Complete_Setup_Guide#Configure_PHP).

### php.ini

``` diff
--- /etc/php/7.4/fpm/php.ini	2021-07-26 16:19:31.832540865 -0300
+++ /etc/php/7.4/fpm/php.ini	2021-07-26 16:29:13.975733282 -0300
@@ -385,7 +385,7 @@
 ; Maximum execution time of each script, in seconds
 ; http://php.net/max-execution-time
 ; Note: This directive is hardcoded to 0 for the CLI SAPI
-max_execution_time = 30
+max_execution_time = 360
 
 ; Maximum amount of time each script may spend parsing request data. It's a good
 ; idea to limit this time on productions servers in order to eliminate unexpectedly
@@ -406,7 +406,7 @@
 
 ; Maximum amount of memory a script may consume
 ; http://php.net/memory-limit
-memory_limit = 256M;
+memory_limit = 1024M;
 
 ;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;
 ; Error handling and logging ;
@@ -691,7 +691,7 @@
 ; Its value may be 0 to disable the limit. It is ignored if POST data reading
 ; is disabled through enable_post_data_reading.
 ; http://php.net/post-max-size
-post_max_size = 12M;
+post_max_size = 200M;
 
 ; Automatically add files before PHP document.
 ; http://php.net/auto-prepend-file
@@ -843,7 +843,7 @@
 
 ; Maximum allowed size for uploaded files.
 ; http://php.net/upload-max-filesize
-upload_max_filesize = 10M;
+upload_max_filesize = 200M;
 
 ; Maximum number of files that can be uploaded via a single request
 max_file_uploads = 20
@@ -959,7 +959,7 @@
 [Date]
 ; Defines the default timezone used by the date functions
 ; http://php.net/date.timezone
-date.timezone = GMT
+date.timezone = America/Sao_Paulo
 
 ; http://php.net/date.default-latitude
 ;date.default_latitude = 31.7667
@@ -1766,20 +1766,20 @@
 
 [opcache]
 ; Determines if Zend OPCache is enabled
-;opcache.enable=1
+opcache.enable=1
 
 ; Determines if Zend OPCache is enabled for the CLI version of PHP
 ;opcache.enable_cli=0
 
 ; The OPcache shared memory storage size.
-;opcache.memory_consumption=128
+opcache.memory_consumption=128
 
 ; The amount of memory for interned strings in Mbytes.
-;opcache.interned_strings_buffer=8
+opcache.interned_strings_buffer=8
 
 ; The maximum number of keys (scripts) in the OPcache hash table.
 ; Only numbers between 200 and 1000000 are allowed.
-;opcache.max_accelerated_files=10000
+opcache.max_accelerated_files=10000
 
 ; The maximum percentage of "wasted" memory until a restart is scheduled.
 ;opcache.max_wasted_percentage=5
@@ -1797,14 +1797,14 @@
 ; How often (in seconds) to check file timestamps for changes to the shared
 ; memory storage allocation. ("1" means validate once per second, but only
 ; once per request. "0" means always validate)
-;opcache.revalidate_freq=2
+opcache.revalidate_freq=1
 
 ; Enables or disables file search in include_path optimization
 ;opcache.revalidate_path=0
 
 ; If disabled, all PHPDoc comments are dropped from the code to reduce the
 ; size of the optimized code.
-;opcache.save_comments=1
+opcache.save_comments=1
 
 ; Allow file existence override (file_exists, etc.) performance feature.
 ;opcache.enable_file_override=0
@@ -1945,3 +1945,8 @@
 
 ; List of headers files to preload, wildcard patterns allowed.
 ;ffi.preload=
+
+[redis]
+redis.session.locking_enabled=1
+redis.session.lock_retries=-1
+redis.session.lock_wait_time=10000
```

### www.conf[<sup>[29]</sup>](https://docs.nextcloud.com/server/latest/admin_manual/installation/source_installation.html#php-fpm-configuration-notes)

``` diff
--- /etc/php/7.4/fpm/pool.d/www.conf	2021-07-26 16:19:40.032627874 -0300
+++ /etc/php/7.4/fpm/pool.d/www.conf	2021-07-26 16:29:30.827962476 -0300
@@ -28,3 +28,11 @@
 ;
 access.log = /var/log/php-fpm/php-fpm.log
 slowlog = /var/log/php-fpm/slow.log
+
+env[HOSTNAME] = $HOSTNAME
+env[PATH] = /usr/local/bin:/usr/local/sbin:/usr/bin:/bin:/usr/games
+env[TMP] = /tmp
+env[TMPDIR] = /tmp
+env[TEMP] = /tmp
+
+clear_env = no
```

Now we reload the php and enable some modules:

``` console
$ sudo systemctl restart php<version>-fpm.service
$ sudo phpenmod bcmath gmp imagick intl
```

## Redis

Nextcloud supports three programs of in-memory caching (and it complains if you don't enable any of those).
Redis, Memcached, and APCu.

I chose Redis for two reasons:

- It is more compatible with all the software that we want to host.
- And the [Nextcloud high performance file server requires it](https://github.com/nextcloud/notify_push#requirements).

And we will configure it and some other things now.

### Redis.conf

``` diff
--- /etc/redis/redis.conf	2021-07-26 16:19:48.492718235 -0300
+++ /etc/redis/redis.conf	2021-07-26 16:32:35.962525354 -0300
@@ -105,8 +105,8 @@
 # incoming connections. There is no default, so Redis will not listen
 # on a unix socket when not specified.
 #
-# unixsocket /var/run/redis/redis-server.sock
-# unixsocketperm 700
+unixsocket /var/run/redis/redis-server.sock
+unixsocketperm 770
 
 # Close the connection after a client is idle for N seconds (0 to disable)
 timeout 0
@@ -787,7 +787,7 @@
 # AUTH <password> as usually, or more explicitly with AUTH default <password>
 # if they follow the new protocol: both will work.
 #
-# requirepass foobared
+requirepass <Redis-Password>
 
 # Command renaming (DEPRECATED).
 #
@@ -857,7 +857,7 @@
 # limit for maxmemory so that there is some free RAM on the system for replica
 # output buffers (but this is not needed if the policy is 'noeviction').
 #
-# maxmemory <bytes>
+maxmemory 1073741824
 
 # MAXMEMORY POLICY: how Redis will select what to remove when maxmemory
 # is reached. You can select one from the following behaviors:
```

### Kernel and system config

Change the `transparent_hugepage` page to opt-in
[<sup>[30]</sup>](https://stackoverflow.com/a/64945381)
[<sup>[31]</sup>](https://redis.io/topics/admin).

``` diff
--- /etc/default/grub	2021-07-26 16:19:57.024809984 -0300
+++ /etc/default/grub	2021-07-26 16:34:14.155912305 -0300
@@ -7,7 +7,7 @@
 GRUB_TIMEOUT=5
 GRUB_DISTRIBUTOR=`lsb_release -i -s 2> /dev/null || echo Debian`
 GRUB_CMDLINE_LINUX_DEFAULT="quiet"
-GRUB_CMDLINE_LINUX=""
+GRUB_CMDLINE_LINUX="transparent_hugepage=madvise"
 
 # Uncomment to enable BadRAM filtering, modify to suit your needs
 # This works with Linux (no patch required) and with any kernel that obtains
```

Set Linux kernel over-commit memory setting to 1.

``` diff
--- /etc/sysctl.conf	2021-07-26 16:20:07.112919234 -0300
+++ /etc/sysctl.conf	2021-07-26 16:34:29.060124193 -0300
@@ -66,3 +66,4 @@
 # for what other values do
 #kernel.sysrq=438
 
+vm.overcommit_memory = 1
```

Add the web-server user to the redis group, update grub and reboot to all take effect.

``` console
$ sudo usermod -a -G redis www-data
$ sudo update-grub
$ sudo systemctl reboot
```

## Generating certificates

This step is necessary if you don’t want to see your browser screaming because your site doesn't have
a verified certificate. I chose the [let's encrypt](https://letsencrypt.org/en-us/), because
it is free and extremely easy to setup.

First do the [Installing](#h-installing) step in the Nextcloud section.

After Install the Nextcloud, check if your domain configuration are working. The command below should return the IP
of your server.

``` console
$ dig short -t a <Mail-Hostname> <Hostname> matrix.<Hostname>
```

Run the certboot in dry-run mode to test if everything is correct.

``` console
$ sudo certbot certonly --webroot --dry-run -w /var/www/html -d <Mail-Hostname> -w /var/www/nextcloud -d <Hostname> -w /var/www/html -d matrix.<Hostname> -w /var/www/autoconfig -d autodiscover.<Hostname> -w /var/www/autoconfig -d autoconfig.<Hostname>
```

Run without the `—dry-run`. Check the folder. It should contain the certificates and the keys.

``` console
$ sudo ls /etc/letsencrypt/live/
$ sudo ls /etc/letsencrypt/live/<Mail-Hostname>
```

Allow access to non-root(postgress/Postfix/Dovecot).

``` console
$ sudo chmod 0755 /etc/letsencrypt/{live,archive}
```

Now we have to change the certificate in each service, this is the easiest way is to replace the iRedMail certificate
with the lets encrypt.

``` console
$ sudo rm -f /etc/ssl/private/iRedMail.key /etc/ssl/certs/iRedMail.crt
$ sudo ln -s /etc/letsencrypt/live/<Mail-Hostname>/privkey.pem /etc/ssl/private/iRedMail.key
$ sudo ln -s /etc/letsencrypt/live/<Mail-Hostname>/fullchain.pem /etc/ssl/certs/iRedMail.crt
```

Check if everything is correct.

``` console
$ ls /etc/ssl/certs -l | grep iRed
-rw------- 1 postgres postgres 2147 Jun 20 02:05 iRedMail_CA_PostgreSQL.pem
lrwxrwxrwx 1 root root 53 Jun 20 13:49 iRedMail.crt -> /etc/letsencrypt/live/<Mail-Hostname>/fullchain.pem
$ sudo ls /etc/ssl/private -l | grep iRed
lrwxrwxrwx 1 root root 51 Jun 20 13:50 iRedMail.key -> /etc/letsencrypt/live/<Mail-Hostname>/privkey.pem
-rw------- 1 postgres postgres 3272 Jun 20 02:05 iRedMail_PostgreSQL.key
```

Create a group to the doveadm access the certificate private-key. It is called in the PHP script, and the PHP
runs as `www-data` user, and that is why we are adding the `www-data` group.

``` console
$ sudo groupadd cert-priv-key
$ sudo gpasswd -M root,www-data cert-priv-key
$ sudo chown root:cert-priv-key /etc/letsencrypt/archive/<Mail-Hostname>/privkey1.pem
$ sudo chmod 440 /etc/letsencrypt/archive/<Mail-Hostname>/privkey1.pem
```

Add a cronjob to renew the certificate.

``` console
$ sudo crontab -u root -e
\* \* \* \* \* /bin/console /usr/local/bin/fail2ban_banned_db unban_db
```

``` diff
--- crontab	2021-07-26 16:40:16.291860027 -0300
+++ crontab	2021-07-26 16:41:01.485357931 -0300
@@ -17,3 +17,6 @@
 1   *   *   *   *   python3 /opt/www/iredadmin/tools/delete_mailboxes.py
 # Fail2ban: Unban IP addresses pending for removal (stored in SQL db).
 * * * * * /bin/bash /usr/local/bin/fail2ban_banned_db unban_db
+
+# Let's encrypt certificates
+1   3   *   *   *   certbot renew --post-hook '/usr/sbin/service postfix restart; /usr/sbin/service nginx restart; /usr/sbin/service dovecot restart'
```

Restart all the services.

``` console
$ sudo systemctl restart postfix.service dovecot.service nginx.service postgresql.service
```

## Nextcloud

Here we will install and configure the main piece of software of our cloud, the Nextcloud. I will not explain
what is Nextcloud because there are great resources out there. So let’s jump to the installation process.

### Installing

Copy the download link At [Nextcloud](https://nextcloud.com/install/#instructions-server) site, unzip and install the
nextcloud[<sup>[32]</sup>](https://wiki.learnlinux.tv/index.php/Nextcloud_-_Complete_Setup_Guide#Organize_Apache_files):

``` console
$ wget https://download.nextcloud.com/server/releases/nextcloud-<Nextcloud-version>.zip
$ unzip nextcloud-<Nextcloud-version>.zip
$ sudo chown -R www-data:www-data nextcloud
$ sudo mv nextcloud /var/www/
```

### Configuring

Change the `/var/server_data/nextcloud` ownership.

``` console
$ sudo chown www-data:www-data /var/server_data/nextcloud
```

Open the address cloud https://**\<Hostname>**{: style="color: #d2929d" }/ in our browser, and if everything is working
you should have the initial setup interface.

![Nextcloud setup screen image](/assets/images/cloud-server/nextcloud_setup_screen.png)

{: .box-note}
**Note:** I usually don't select the 'Install recommended apps' box because it comes with undesired stuff and takes much
more time to install, but it's up to you the decision.

**Servername:** **\<Nextcloud-Admin>**{: style="color: red" }<br>
**Password:** **\<Nextcloud-Admin-Pass>**{: style="color: blue" }<br>
**Data Folder:**`/path/to/zfs/nextcloud/data`<br>
**Database User:** **\<Nextcloud-Postgres-user>**{: style="color: #ffc000" }<br>
**Database Password:** **\<Nextcloud-database-pass>**{: style="color: purple" }<br>
**Database Name:** **\<Nextcloud-database-Name>**{: style="color: green" }<br>

Here we have to change the config file to fix some issues
[<sup>[33]</sup>](https://docs.nextcloud.com/server/21/admin_manual/configuration_server/caching_configuration.html#id2),
add the following lines to the`$CONFIG = array`:

{: .box-note}
**Note:** The **\<vmailadmin-password>** will be in the file`iRedMail-<version>/iRedMail.tips`in the`PostgreSQL:`section.
{: .box-warning}

``` php
/var/www/nextcloud/config/config.php

  'memcache.local' => '\\OC\\Memcache\\Redis',
  'memcache.distributed' => '\\OC\\Memcache\\Redis',
  'memcache.locking' => '\\OC\\Memcache\\Redis',
  'redis' => 
  array (
    'host' => '/var/run/redis/redis-server.sock',
    'port' => 0,
    'dbindex' => 0,
    'password' => '<Redis-Password>',
    'timeout' => 1.5,
  ),
  'default_phone_region' => 'BR',
  'logdateformat' => 'F d, Y H:i:s',
  'trusted_proxies' => 
  array (
    0 => '127.0.0.1',
    1 => '::1',
  ),
  'app_install_overwrite' => array (0 => 'user_backend_sql_raw'),
  'user_backend_sql_raw' => 
  array (
    'db_type' => 'postgresql',
    'db_host' => 'localhost',
    'db_port' => '5432',
    'db_name' => 'vmail',
    'db_user' => 'vmailadmin',
    'db_password' => '<vmailadmin-password>',
    'hash_algorithm_for_new_passwords' => 'bcrypt',
    'queries' => 
    array (
      'user_exists' => 'SELECT EXISTS(SELECT 1 FROM mailbox WHERE username = (:username || \'@<Hostname>\'))',
      'get_users' => 'SELECT split_part(username, \'@\', 1) FROM mailbox WHERE (username ILIKE :search) OR (name ILIKE :search)',
      'get_password_hash_for_user' => 'SELECT split_part(password, \'{CRYPT}\', 2) FROM mailbox WHERE username = (:username || \'@<Hostname>\')',
      'set_password_hash_for_user' => 'UPDATE mailbox SET password =  \'{CRYPT}\' || :new_password_hash WHERE username = (:username || \'@<Hostname>\')',
      'get_display_name' => 'SELECT name FROM mailbox WHERE username = (:username || \'@<Hostname>\')',
      'set_display_name' => 'UPDATE mailbox SET name = :new_display_name WHERE username = (:username || \'@<Hostname>\')',
    ),
  ),
);
```

Reload the Nginx.

``` console
$ sudo nginx -s reload
```

### Email and Nextcloud password integration

Download the app `user_backend_sql_raw`[<sup>[34]</sup>](https://apps.nextcloud.com/apps/user_backend_sql_raw)
from the "Nextcloud store" to enable the integration.

In the current configuration, we use the username to the Nextcloud. And the username + domain to login into the email.

To change the behavior to use the username + domain in both just use the commented lines and comment on the other lines.

|             **Current behavior**           |          **Username + domain behavior**        |
| ------------------------------------------ | ---------------------------------------------- |
| Email user: joanadarc@\<Mail-Hostname\&gt; | Email user: joanadarc@\<Mail-Hostname\&gt;     |
| Nextcloud user: joanadarc                  | Nextcloud user: joanadarc@\<Mail-Hostname\&gt; |

### Setup Fail2Ban

Let's configure the fail2ban on our Nextcloud instance
[<sup>[35]</sup>](https://docs.nextcloud.com/server/21/admin_manual/installation/harden_server.html#setup-fail2ban),
add the following files.

{% highlight toml wl linenos %}
{% raw %}
/etc/fail2ban/filter.d/nextcloud.conf

[Definition]
_groupsre = (?:(?:,?\s*"\w":(?:"[^"]"|\w))*)
failregex = ^\{%(_groupsre)s,?\s*"remoteAddr":"<HOST>"%(_groupsre)s,?\s*"message":"Login failed:
            ^\{%(_groupsre)s,?\s*"remoteAddr":"<HOST>"%(_groupsre)s,?\s*"message":"Trusted domain error.
datepattern = ,?\s*"time"\s*:\s*"%%Y-%%m-%%d[T ]%%H:%%M:%%S(%%z)?"
{% endraw %}
{% endhighlight %}

{% highlight toml wl linenos %}
/etc/fail2ban/jail.d/nextcloud.local

[nextcloud]
backend = auto
enabled = true
port = 80,443
protocol = tcp
filter = nextcloud
maxretry = 3
bantime = 86400
findtime = 43200
logpath = /var/server_data/nextcloud/nextcloud.log
{% endhighlight %}

Bantime, and findtime in seconds, change it as needed.

Now we need restart to the fail2ban service.

``` console
$ sudo systemctl restart fail2ban.service
```

And test it.

``` console
$ sudo fail2ban-client status nextcloud
Status for the jail: nextcloud
|- Filter
| |- Currently failed: 0
| |- Total failed: 0
| `- File list: /var/server_data/nextcloud/nextcloud.log
`- Actions
|- Currently banned: 0
|- Total banned: 0
`- Banned IP list:
```

### Nextcloud background jobs

As recommended in several places
[<sup>[36]</sup>](https://docs.nextcloud.com/server/21/admin_manual/configuration_server/background_jobs_configuration.html#systemd)
we will move from Ajax to the Nextcloud background jobs, but we will use the systemd.

{% highlight toml wl linenos %}
etc/systemd/system/nextcloudcron.service

[Unit]
Description=Nextcloud cron.php job

[Service]
User=www-data
ExecStart=/usr/bin/php -f /var/www/nextcloud/cron.php
KillMode=process
{% endhighlight %}

{% highlight toml wl linenos %}
/etc/systemd/system/nextcloudcron.timer

[Unit]
Description=Run Nextcloud cron.php every 5 minutes

[Timer]
OnBootSec=5min
OnUnitActiveSec=5min
Unit=nextcloudcron.service

[Install]
WantedBy=timers.target
{% endhighlight %}

Enable the `nextcloudcron.timer`.

``` console
$ sudo systemctl enable --now nextcloudcron.timer
```

### Roundcube integration

There's an app to integrate the Roundcube to the Nextcloud, but, unfortunately, it isn't available in the
'Nextcloud store'. So we need to install it manually.

I create a fork with some fixes and new features. Hopefully, at the time you are reading this, all changes are already
merged.

My repo

``` console
$ git clone https://github.com/Igortorrente/Nextcloud-roundcube.git
```

Official repository

``` console
$ git clone https://github.com/flionet89/Nextcloud-roundcube
```

Move the code to the correct folder.

``` console
$ sudo chown www-data:www-data Nextcloud-roundcube
$ sudo mv Nextcloud-roundcube/roundcube /var/www/nextcloud/apps/.
```

Enable it in the in graphics interface under apps -> Disable apps -> RoundCube Mail.

![Roundcube plugin enable screen](/assets/images/cloud-server/nextcloud_roundcube_enable.png){: .center-block :}

Configure it with the config below.

![Roundcube config screen](/assets/images/cloud-server/nextcloud_roundcube_plugin.png){: .center-block :}

### [Nextcloud high performance file server](https://github.com/nextcloud/notify_push):

The high-performance backend improves the server-load and the notification delay to the client
[<sup>[37]</sup>](https://github.com/nextcloud/notify_push#about).
This is optional, but If you are running the Nextcloud in the aarch64, armv7, i386 or amd64 I totally
recommend you follow this part.

Install the client in the interface.

![Nextcloud client push](/assets/images/cloud-server/nextcloud_client_push.png){: .center-block :}

Add a new service to the systemd[<sup>[38]</sup>](https://github.com/nextcloud/notify_push#push-server).

{% highlight toml wl linenos %}
/etc/systemd/system/notify_push.service

[Unit]
Description = Push daemon for Nextcloud clients

[Service]
Environment = PORT=7867
ExecStart = /var/www/nextcloud/apps/notify_push/bin/x86_64/notify_push -- /var/www/nextcloud/config/config.php
User = www-data
Restart = always
 
[Install]
WantedBy = multi-user.target
{% endhighlight %}

Reload the daemon and enable the new service.

``` console
$ sudo systemctl daemon-reload
$ sudo systemctl enable notify_push.service --now
```

Change the [nextcloud.tmpl](#h-nextcloud) to have add a reverse proxy to the `notify_push`
[<sup>[39]</sup>](https://github.com/nextcloud/notify_push#nginx).

``` diff
--- /etc/nginx/sites-available/nextcloud.conf	2021-07-26 20:50:05.267899272 +0000
+++ /etc/nginx/sites-available/nextcloud.conf	2021-07-21 22:44:30.149339346 +0000
@@ -53,6 +53,15 @@
     # always provides the desired behaviour.
     index index.php index.html /index.php$request_uri;
 
+     location ^~ /push/ {
+        proxy_pass http://127.0.0.1:7867/;
+        proxy_http_version 1.1;
+        proxy_set_header Upgrade $http_upgrade;
+        proxy_set_header Connection "Upgrade";
+        proxy_set_header Host $host;
+        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
+    }
+
     location /roundcube/ {
         proxy_set_header X-Forwarded-Host $host:$server_port;
         proxy_set_header X-Forwarded-Server $host;
```

Reload Nginx and enable it.

``` console
$ sudo nginx -s reload
$ sudo -u www-data php /var/www/nextcloud/occ app:enable notify_push
$ sudo -u www-data php /var/www/nextcloud/occ notify_push:setup https://<Hostname>/push
```

If all goes well you should receive this messages:

``` console
✓ redis is configured
✓ push server is receiving redis messages
✓ push server can load mount info from database
✓ push server can connect to the Nextcloud server
✓ push server is a trusted proxy
✓ push server is running the same version as the app
configuration saved
```

## Service emails

One of the nicest features that I discovered while installing my cloud is that the Unattended-upgrades and
the ZFS-zed can send emails to you when a disk fails, a scrub finds errors, or with an upgrade report.
Even though this part is optional, I highly recommend it.

### Emails

Create a email to our S.O. inform us about errors, and another to be used in our service. Using the
[Accessing the IRedAdmin from external computer](#h-accessing-the-iredadmin-from-external-computer), access the
page to create new domain (<Hostname>) and add two emails.

- [system-bot@**\<Mail-Hostname>**{: style="color: #f87e19" }] - System notification bot.
- [no-reply@**\<Mail-Hostname>**{: style="color: #f87e19" }] - To be used by several services.

![iRedAdmin panel](/assets/images/cloud-server/iRedAdmin.png){: .center-block :}

### Unattended-Upgrade

Install a smtp client
[<sup>[40]</sup>](https://gist.github.com/roybotnik/b0ec2eda2bc625e19eaf#email-notification-configuration).

``` console
$ sudo apt install bsd-mailx -y
```

Add the config.

{% highlight toml wl linenos %}
/root/.mailc

set smtp-use-starttls
#set ssl-verify=ignore
set smtp=smtp://<Mail-Hostname>:587
set smtp-auth=login
set smtp-auth-user=system-bot@<Mail-Hostname>
set smtp-auth-password=<mail-password>
set from="system-bot@<Mail-Hostname>"
{% endhighlight %}

Configure the Unattended-upgrades by adding the email(s) that we want to receive the reports.

``` diff
--- /etc/apt/apt.conf.d/50unattended-upgrades	2021-07-26 14:50:47.065457281 -0300
+++ /etc/apt/apt.conf.d/50unattended-upgrades	2021-07-26 17:27:31.283157779 -0300
@@ -91,13 +91,13 @@
 // If empty or unset then no email is sent, make sure that you
 // have a working mail setup on your system. A package that provides
 // 'mailx' must be installed. E.g. "user@example.com"
-//Unattended-Upgrade::Mail "";
+Unattended-Upgrade::Mail "<user>@<Mail-Hostname>";
 
 // Set this value to one of:
 //    "always", "only-on-error" or "on-change"
 // If this is not set, then any legacy MailOnlyOnError (boolean) value
 // is used to chose between "only-on-error" and "on-change"
-//Unattended-Upgrade::MailReport "on-change";
+Unattended-Upgrade::MailReport "on-change";
 
 // Remove unused automatically installed kernel-related packages
 // (kernel images, kernel headers and kernel version locked tools).
```

### ZFS mail bot

Change the zed config to enable zfs email reports
[<sup>[41]</sup>](https://wiki.archlinux.org/title/ZFS#Monitoring_/_Mailing_on_Events).

``` diff
--- /etc/zfs/zed.d/zed.rc	2021-07-26 17:28:40.363602144 -0300
+++ /etc/zfs/zed.d/zed.rc	2021-07-26 17:29:31.999936172 -0300
@@ -15,14 +15,14 @@
 # Email will only be sent if ZED_EMAIL_ADDR is defined.
 # Disabled by default; uncomment to enable.
 #
-ZED_EMAIL_ADDR="root"
+ZED_EMAIL_ADDR="<user>@<Mail-Hostname>"
 
 ##
 # Name or path of executable responsible for sending notifications via email;
 #   the mail program must be capable of reading a message body from stdin.
 # Email will only be sent if ZED_EMAIL_ADDR is defined.
 #
-#ZED_EMAIL_PROG="mail"
+#ZED_EMAIL_PROG="mailx"
 
 ##
 # Command-line options for ZED_EMAIL_PROG.
@@ -31,7 +31,7 @@
 #   this should be protected with quotes to prevent word-splitting.
 # Email will only be sent if ZED_EMAIL_ADDR is defined.
 #
-#ZED_EMAIL_OPTS="-s '@SUBJECT@' @ADDRESS@"
+ZED_EMAIL_OPTS="-s '@SUBJECT@' @ADDRESS@"
 
 ##
 # Default directory for zed lock files.
@@ -54,7 +54,7 @@
 # Send notifications for 'ereport.fs.zfs.data' events.
 # Disabled by default, any non-empty value will enable the feature.
 #
-#ZED_NOTIFY_DATA=
+ZED_NOTIFY_DATA=1
 
 ##
 # Pushbullet access token.
```

Change the accesses to a very restrictive one.

``` console
$ sudo chmod 600 /root/.mailc
```

Restart the services to reload the configs.

``` console
$ sudo systemctl restart unattended-upgrades.service zfs-zed.service
```

Use `ZED_NOTIFY_VERBOSE=1 in the configuration and `zpool scrub <pool-name>` to test the zed config.

If you have any upgrade to do you can test it using.

``` console
$ sudo unattended-upgrade -v -d
```

### Nextcloud

Add the credential as shown below.

![Nextcloud email configuration screen](/assets/images/cloud-server/nextcloud_email_setup.png){: .center-block :}

## Matrix Server

The last piece of software is the matrix server. I chose the [Synapse](https://github.com/matrix-org/synapse)
mainly because it's the most complete implementation(in fact, it is the reference implementation), and my
cloud isn't that big to suffer from scalability issues, so it should be enough.

Maybe when you are reading it, [Dendrite](https://matrix.org/docs/projects/server/dendrite),
[Conduit](https://gitlab.com/famedly/conduit), or even the [Nextcloud talk](https://nextcloud.com/talk/),
are in good shape. But for now, the Synapse is the best option.

To install the synapse matrix server we need to add the official repo, update the apt, and install it
[<sup>[42]</sup>](https://github.com/matrix-org/synapse/blob/develop/INSTALL.md#debianubuntu).

``` console
$ sudo apt install -y lsb-release wget apt-transport-https
$ sudo wget -O /usr/share/keyrings/matrix-org-archive-keyring.gpg https://packages.matrix.org/debian/matrix-org-archive-keyring.gpg
$ echo "deb [signed-by=/usr/share/keyrings/matrix-org-archive-keyring.gpg] https://packages.matrix.org/debian/ $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/matrix-org.list
$ sudo apt update && sudo apt upgrade
$ sudo apt install matrix-synapse-py3 libxml2-dev -y
```

During installation, use the server as `matrix.<Hostname>`.

Now we will add the `.well-know` server file, but first create the folder.

``` console
$ sudo mkdir -p /var/www/html/.well-known/matrix/
```

Add the file.

{% highlight json wl linenos %}
/var/www/html/.well-known/matrix/server

{
    "m.server": "matrix.<Hostname>:443"
}
{% endhighlight %}

Change the `/etc/matrix-synapse/homeserver.yaml` using the diff below as a base configuration,
**and don't forget** to change the lines:

- [729] **user:**  **\<Matrix-Postgres-user>**{: style="color: #779d3c" }
- [730] **password:**  **\<Matrix-database-pass>**{: style="color: red" }
- [731] **database:**  **\<Matrix-database-Name>**{: style="color: #ff63b5" }
- [923] **media_store_path: &quot;/var/lib/matrix-synapse/media&quot;**
- [1132] **registration_shared_secret:** &quot;**\<Matrix-registration_shared_secret>**{: style="color: blue" }&quot;
- [2376] **smtp_user: &quot;service_email@<Mail-Hostname>&quot;**
- [2377] **smtp_pass: &quot;\<Service-Email-Password>&quot;**
- [2847] **password:**  **\<Redis-Password>**{: style="color: #ff4e00" }

``` diff
--- /etc/matrix-synapse/homeserver.yaml	2021-07-26 17:34:17.229803531 -0300
+++ /etc/matrix-synapse/homeserver.yaml	2021-07-26 17:43:45.301587996 -0300
@@ -67,7 +67,7 @@
 # Otherwise, it should be the URL to reach Synapse's client HTTP listener (see
 # 'listeners' below).
 #
-#public_baseurl: https://example.com/
+public_baseurl: https://matrix.<Hostname>/
 
 # Set the soft limit on the number of file descriptors synapse can use
 # Zero is used to indicate synapse should set the soft limit to the
@@ -729,10 +729,23 @@
 # see https://matrix-org.github.io/synapse/latest/postgres.html.
 #
 database:
-  name: sqlite3
+  name: psycopg2
   args:
-    database: /var/lib/matrix-synapse/homeserver.db
-
+    user: <Matrix-Postgres-user>
+    password: <Synapse-Postgres-user>
+    database: <Matrix-database-Name>
+    host: localhost
+    port: 5432
+    cp_min: 5
+    cp_max: 10
+    # seconds of inactivity after which TCP should send a keepalive message to the server
+    keepalives_idle: 10
+    # the number of seconds after which a TCP keepalive message that is not
+    # acknowledged by the server should be retransmitted
+    keepalives_interval: 10
+    # the number of TCP keepalives that can be lost before the client's connection
+    # to the server is considered dead
+    keepalives_count: 3
 
 ## Logging ##
 
@@ -919,7 +932,7 @@
 # 'false' by default: uncomment the following to enable it (and specify a
 # url_preview_ip_range_blacklist blacklist).
 #
-#url_preview_enabled: true
+url_preview_enabled: true
 
 # List of IP address CIDR ranges that the URL preview spider is denied
 # from accessing.  There are no defaults: you must explicitly
@@ -935,26 +948,26 @@
 # This must be specified if url_preview_enabled is set. It is recommended that
 # you uncomment the following list as a starting point.
 #
-#url_preview_ip_range_blacklist:
-#  - '127.0.0.0/8'
-#  - '10.0.0.0/8'
-#  - '172.16.0.0/12'
-#  - '192.168.0.0/16'
-#  - '100.64.0.0/10'
-#  - '192.0.0.0/24'
-#  - '169.254.0.0/16'
-#  - '192.88.99.0/24'
-#  - '198.18.0.0/15'
-#  - '192.0.2.0/24'
-#  - '198.51.100.0/24'
-#  - '203.0.113.0/24'
-#  - '224.0.0.0/4'
-#  - '::1/128'
-#  - 'fe80::/10'
-#  - 'fc00::/7'
-#  - '2001:db8::/32'
-#  - 'ff00::/8'
-#  - 'fec0::/10'
+url_preview_ip_range_blacklist:
+  - '127.0.0.0/8'
+  - '10.0.0.0/8'
+  - '172.16.0.0/12'
+  - '192.168.0.0/16'
+  - '100.64.0.0/10'
+  - '192.0.0.0/24'
+  - '169.254.0.0/16'
+  - '192.88.99.0/24'
+  - '198.18.0.0/15'
+  - '192.0.2.0/24'
+  - '198.51.100.0/24'
+  - '203.0.113.0/24'
+  - '224.0.0.0/4'
+  - '::1/128'
+  - 'fe80::/10'
+  - 'fc00::/7'
+  - '2001:db8::/32'
+  - 'ff00::/8'
+  - 'fec0::/10'
 
 # List of IP address CIDR ranges that the URL preview spider is allowed
 # to access even if they are specified in url_preview_ip_range_blacklist.
@@ -1001,7 +1014,7 @@
 
 # The largest allowed URL preview spidering size in bytes
 #
-#max_spider_size: 10M
+max_spider_size: 10M
 
 # A list of values for the Accept-Language HTTP header used when
 # downloading webpages during URL preview generation. This allows
@@ -1132,7 +1145,7 @@
 # If set, allows registration of standard or admin accounts by anyone who
 # has the shared secret, even if registration is otherwise disabled.
 #
-#registration_shared_secret: <PRIVATE STRING>
+registration_shared_secret: <PRIVATE STRING>
 
 # Set the number of bcrypt rounds used to generate password hash.
 # Larger numbers increase the work factor needed to generate the hash.
@@ -2267,24 +2280,24 @@
 email:
   # The hostname of the outgoing SMTP server to use. Defaults to 'localhost'.
   #
-  #smtp_host: mail.server
+  smtp_host: '<Mail-Hostname>'
 
   # The port on the mail server for outgoing SMTP. Defaults to 25.
   #
-  #smtp_port: 587
+  smtp_port: 587
 
   # Username/password for authentication to the SMTP server. By default, no
   # authentication is attempted.
   #
-  #smtp_user: "exampleusername"
-  #smtp_pass: "examplepassword"
+  smtp_user: "service_email@<Mail-Hostname>"
+  smtp_pass: "<Email-Password>"
 
   # Uncomment the following to require TLS transport security for SMTP.
   # By default, Synapse will connect over plain text, and will then switch to
   # TLS via STARTTLS *if the SMTP server supports it*. If this option is set,
   # Synapse will refuse to connect unless the server supports STARTTLS.
   #
-  #require_transport_security: true
+  require_transport_security: true
 
   # notif_from defines the "From" address to use when sending emails.
   # It must be set if email sending is enabled.
@@ -2296,12 +2309,12 @@
   # Note that the placeholder must be written '%(app)s', including the
   # trailing 's'.
   #
-  #notif_from: "Your Friendly %(app)s homeserver <noreply@example.com>"
+  notif_from: "Your Friendly %(app)s homeserver <noreply@example.com>"
 
   # app_name defines the default value for '%(app)s' in notif_from and email
   # subjects. It defaults to 'Matrix'.
   #
-  #app_name: my_branded_matrix_server
+  app_name: your_site_name_matrix_server
 
   # Uncomment the following to enable sending emails for messages that the user
   # has missed. Disabled by default.
@@ -2834,14 +2847,14 @@
 redis:
   # Uncomment the below to enable Redis support.
   #
-  #enabled: true
+  enabled: true
 
   # Optional host and port to use to connect to redis. Defaults to
   # localhost and 6379
   #
-  #host: localhost
-  #port: 6379
+  host: localhost
+  port: 6379
 
   # Optional password if configured on the Redis instance
   #
-  #password: <secret_password>
+  password: <Redis-Password>
```

Now reboot the machine to the synapse start correctly.

``` console
$ sudo systemctl reboot
```

Use [this](https://federationtester.matrix.org/) and [this](https://fed.mau.dev/) sites to test the federation setup.

And finally we will add a admin user [<sup>[43]</sup>](https://github.com/matrix-org/synapse/blob/master/INSTALL.md#registering-a-user).

``` console
$ sudo register_new_matrix_user -c /etc/matrix-synapse/homeserver.yaml http://localhost:8008
New user localpart [root]: <Nextcloud-admin-user>
Password: <Matrix-Admin-Password>
Confirm password: <Matrix-Admin-Password>
Make admin [no]: yes
Sending registration request...
Success!
```

## Firewall (UFW)

I’m using the UFW because it is as simple as my need for a firewall. You can skip this part if you need something more robust.

``` console
$ sudo ufw default deny incoming
$ sudo ufw default allow outgoing
$ sudo ufw allow OpenSSH
$ sudo ufw allow WWW
$ sudo ufw allow 'WWW Secure'
$ sudo ufw allow 'Dovecot IMAP'
$ sudo ufw allow 'Dovecot POP3'
$ sudo ufw allow 'Dovecot Secure IMAP'
$ sudo ufw allow 'Dovecot Secure POP3'
$ sudo ufw allow 'Nginx Full'
$ sudo ufw allow Postfix
$ sudo ufw allow 'Postfix SMTPS'
$ sudo ufw allow 'Postfix Submission'
$ sudo ufw enable
```

## Maintenance

In this section, I want to provide some maintenance procedures that can be useful for someone like me that
is running the cloud with the same setup as this tutorial.

### Accessing the IredAdmin from external computer

The IredAdmin is only accessible from the localhost of server, so we need to forward the port:

``` console
$ ssh -L 5001:localhost:443 caedrium-server
```

And then we access from our browser [https://localhost:5001/](https://localhost:5001/).

### System binary corruption

Because our root isn't covered by ZFS we need check the file for corruption in a periodic fashion
(One a month should be more than enough). To do that we use the **debsums** to check against the MD5 sum
in the Debian servers.

``` console
$ sudo debsums --changed --all
```

To fix the binary.

``` console
$ sudo apt install --reinstall <package>;
```

### Upgrading IRedMail

The upgrade is basically manual and should follow a guide to each upgrade. Look at the following
[link](https://docs.iredmail.org/iredmail.releases.html#how-upgrading-works) every time that
a new version is released. As stated in the documentation:

- Usually, upgrading iRedMail is just updating some config files to achieve new features or fix bugs,
you do **NOT** need to download and run the latest iRedMail installer (console iRedMail.sh).
- Do **NOT** skip releases. Upgrades are only supported from one release to the release immediately following it.

### Add new sub-domains to the certificate

Here we just need to run certbot again and add the new domains. To test we will use the `—dry-run` option.
To truly generate the certificate remove the `—dry-run` option.

``` console
$ sudo certbot certonly --webroot --dry-run -w /var/www/html -d <Mail-Hostname> -w /var/www/nextcloud -d <Hostname> -w /var/www/<folder_subdomain1> -d subdomain1.<Hostname> -w /var/www/<folder_subdomain2>; -d subdomain2.<Hostname> ...
```

And don't forget to modify the dovecot config to point to the new private key following what was done in
Generating certificates.

### Site to test the SSL configuration

[https://www.ssllabs.com/ssltest/](https://www.ssllabs.com/ssltest/)

### Upgrading Nextcloud High performance server

When a update occurs you should restart the notify_push service.

``` console
$ sudo systemctl restart notify_push.service
```

### Add IPs to the fail2ban allow-list(white-list)

``` diff
--- /etc/fail2ban/jail.conf	2021-07-26 17:34:17.229803531 -0300
+++ /etc/fail2ban/jail.conf	2021-07-26 17:43:45.301587996 -0300

@@ -92,27 +92,27 @@
 # "ignoreip" can be a list of IP addresses, CIDR masks or DNS hosts. Fail2ban
 # will not ban a host which matches an address in this list. Several addresses
 # can be defined using space (and/or comma) separator.
-ignoreip = 127.0.0.1/8 ::1
+ignoreip = 127.0.0.1/8 ::1 <IPV4-1> <IPV6-1> <IPV4-2> <IPV6-2> ...
```

### Disk failure

When a disk fails it will appears as `DEGRADED` when call `zpool status server_data` so what we need to do this.

``` console
$ sudo zpool offline server_data <name_of_disk>
```

Change physically the disk and then.

``` console
$ sudo zpool online server_data <name_of_disk>
```

### Backup recover procedure

First we need to send all the data to the from the backup server to the main server using the syncoid.

``` console
$ sudo syncoid --recursive --skip-parent --no-privilege-elevation --sendoptions=w --source-bwlimit 1M backup_pool backup-user@<Server-IP>:server_data
```

And now we will copy the decryption keys to the `/root` and setup the datasets properties one by one,
as the example below.

``` console
$ sudo zfs set keylocation=file:///root/zfs_dataset_nextcloud_pass backup_pool/nextcloud
```

Load all the keys.

``` console
$ sudo zfs load-key -a
```

### Useful commands

``` console
$ sudo netstat -tlp --numeric-hosts --numeric-ports --extend # show listening ports
$ sudo zpool status <pool>
$ sudo nginx -t # test the settings,
$ sudo zfs list -t snapshot # List all snapshots
```

## Troubleshoot

### Roundcube plugin html5_notifier log error

The log error looks like that:

```
Jun 22 19:51:38 caedrium roundcube: <vneuh37c> PHP Error: Failed to load config from /var/lib/roundcube/plugins/html5_notifier/config/config.inc.php in /usr/share/roundcube/program/lib/Roundcube/rcube_plugin.php on line 165 (POST /?_task=mail&_action=refresh)
```

To solve this issue edit the `config.inc.php` file.

{% highlight php wl linenos %}
/usr/share/roundcube/plugins/html5_notifier/config/config.inc.php

 <?php
// Display duration of Desktop Notifications (0 = disabled)
$config['html5_notifier_duration'] = '3';

// Length of displayed mailboxname
// 0 - do not show,
// 1 - short (default),
// 2 - max
$config['html5_notifier_smbox'] = '1';

// Directories excluded for notifications. Use ; as separator for multiple directories
$config['html5_notifier_excluded_directories'] = 'Trash';
{% endhighlight %}

### iRedMail intalation error on Debian 11

Hopefully when you download and install the iRedMail everything will run fine, but if
you are using Debian 11 and receive this message:

```
********* ERROR *********
Release version of the operating system on this server is unsupported by
iRedMail, please access below link to get the latest iRedMail and a list
of supported Linux/BSD distributions and release versions.

http://www.iredmail.org/download.html

*************************
```

There's a ~~very ugly~~ hack that I developed.

First we will transform the folder into a git repo.

``` console
$ cd <iRedMail-version>
$ git init
$ git add .
$ git commit -m "initial commit"
$ cd ..
```

After that we need to clone the repo with the hack and create a patch.

``` console
$ git clone git@github.com:Igortorrente/iRedMail.git iRedMail_hack
$ cd iRedMail_hack
$ git checkout workarround
$ git format-patch HEAD~1
```

And apply the patch.

``` console
$ cp 0001-A-ugly-hack-to-install-on-the-debian-11-bullseye.patch ../<iRedMail-version>
$ cd <iRedMail-version>
$ git apply 0001-A-ugly-hack-to-install-on-the-debian-11-bullseye.patch
```

Now you should be able to run the iRedMail installer and everything should work as expected
in the case of the installation of this documentation.

## Gallery

Roundcube Nextcloud app.

![Roundcube contacts integration](/assets/images/cloud-server/Roundcube-nextcloud-app.png)

Element chat app.

![Roundcube contacts integration](/assets/images/cloud-server/Element-chat.png)

Roundcube carddav plugin.

![Roundcube contacts integration](/assets/images/cloud-server/Cardav-Contacts.png)

Nextcloud contacts as reference.

![Contacts at Nextcloud](/assets/images/cloud-server/Nextcloud-contacts.png)

## See also

- [Other good tutorial](https://123qwe.com/tutorial-debian-10/)
- [Nextcloud security hardening](https://docs.nextcloud.com/server/21/admin_manual/installation/harden_server.html)
- [Linux workstation security](https://github.com/lfit/itpol/blob/master/linux-workstation-security.md)
- [Mozilla SSL configs](https://ssl-config.mozilla.org/)
- [Open-ZFS Bootcamp](https://youtu.be/mLbtJQmfumI)
- [Move over rsync zfs replication with syncoid - Jim Salter (NLUUG 2016-11-17)](https://www.youtube.com/watch?v=VolTJ_t4o0M)
- [DNS configuration]({% post_url 2021-06-29-DNS-Records %})
- [Backup server configuration]({% post_url 2021-06-30-Backup-Server %})
