---
layout: post
title: Cloud post series
subtitle: How to setup all the DNS recordings to have a functional cloud and mail server
categories: Cloud
tags: [Nextcloud, Matrix, Mail]
---
Any correction, improvement, or suggestion send [here](https://github.com/Igortorrente/Blog).

This is the first post of a small series of posts about how to setup the a cloud server.

The objective of this serie of posts is to provide a guide to create a replacement to Google and Microsoft's services,
that is as well integrated as it can be, and enjoyable to use. And a good integration of the services is very important
to this enjoyable experience. So, we will expend a considerable amount of the following tutorials integrating the
services (mainly the webmail in the Nextcloud).

I want to offer services like:
- Email server (Postfix, Dovecot, iRedAdmin, etc)
- Webmail (Roundcube)
- Cloud, Callendar, Contacts, To-do list, etc (Nextcloud)
- Text chat (Synapse matrix server)

And another extremely important feature of these clouds that isn't advertised, is **data integrity**. You probably never
consider this a feature, mainly because this is implicit in these services, but this is a thing that isn't available
out-of-the-box in ours operational systems (Unless you are using FreeBSD or a distro Linux with btrfs as default).

To solve this issue, in the following tutorials we will use the ZFS and ECC memory. I will explain in more dept in the
future.

I also want to have the backup of all data in the worst case. And to solve it, I will use the sanoid and syncoid to help
me to make this process automatic. And this will require a separate blog post.

After this introduction let's define some things.

## Notation

In this series, I will use the following **\<Label>** to indicate when the readers need to change something in the
command, diff, or configuration. **Pay attention if what you are copying containis the \<label>**{: style="color: red" }

## Users, Password, names, and etc

We will have a bunch of the information to define, I recommend you to use some kind of password manager - like
[KeePassXC](https://keepassxc.org/) - to generate and safely save the passwords, users, and others pieces of information.

**Nextcloud admin user:** **\<Nextcloud-Admin>**{: style="color: red" } (Ex: Admin)<br>
**Nextcloud password:** **\<Nextcloud-Admin-Pass>**{: style="color: blue" }\*<br>
**Nextcloud database name:** **\<Nextcloud-database-Name>**{: style="color: green" } (Ex: nextcloud_db)<br>
**Nextcloud postgresql user:** **\<Nextcloud-Postgres-user>**{: style="color: #ffc000" } (Ex: nextcloud)<br>
**Nextcloud database password:** **\<Nextcloud-database-pass>**{: style="color: purple" }\*\*<br>
**Postgresql superuser password:** **\<Postgres-SU-Pass>**{: style="color black" }\*\*<br>
**Hostname:** **\<Hostname>**{: style="color: #d2929d" } (Ex: caedrium.com)<br>
**Hostname to email:** **\<Mail-Hostname>**{: style="color: #f87e19" } (Ex: mail.caedrium.com)<br>
**Iredadmin postmaster password:** **\<Mail-Admin-Pass>**{: style="color: #00b0f0" }<br>
**Redis password:** **\<Redis-Password>**{: style="color: #ff4e00" }<br>
**Matrix server database name:** **\<Matrix-database-Name>**{: style="color: #ff63b5" } (Ex: synapse_db)<br>
**Matrix server postgresql user:** **\<Matrix-Postgres-user>**{: style="color: #779d3c" } (Ex: synapse)<br>
**Matrix server database password:** **\<Matrix-database-pass>**{: style="color: red" }\*\*<br>
**Matrix registration_shared_secret:** **\<Matrix-registration_shared_secret>**{: style="color: blue" }\*\*<br>
**Matrix admin user:** **\<Nextcloud-admin-user>**{: style="color: green" } (Ex: admin)<br>
**Matrix admin password:** **\<Matrix-Admin-Password>**{: style="color: #ffc000" }\*\*<br>
**Roundcube database password:** **\<Roundcube-database-Pass>**{: style="color: purple" }<br>

\* Nextcloud has a limit of 72 characters[<sup>[1]</sup>](https://docs.nextcloud.com/server/latest/admin_manual/installation/harden_server.html#limit-on-password-length)<br>
\*\* Only use letters and numbers. Special characters can cause problems with some programs like Nextcloud Files High-Performance Backend and IRedMail.


