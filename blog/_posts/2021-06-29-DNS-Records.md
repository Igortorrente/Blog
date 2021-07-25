---
layout: post
title: Cloud DNS Records
subtitle: How to setup all the DNS recordings to have a functional cloud and mail server
categories: Cloud
tags: [Nextcloud, DNS, Matrix, Mail]
---
Any correction, improvement, or suggestion send [here](https://github.com/Igortorrente/Blog).

This post is a continuation of the [Cloud post series]({% post_url 2021-06-27-Cloud-Posts %})
and [How to host a alternative to the Google/Microsoft services]({% post_url 2021-06-28-Cloud-server %}).

If you don't know anything about DNS records, check this great [overview](https://www.pbrumby.com/2018/05/09/dns-records-explained/).
Because we will setup all the DNS records necessary for our services, so a basic understanding of DNS records will help.

Here we will setup all the DNS records to our cloud, from the basic records (A, AAAA, and MX), to some more specifics to
our email (SRV, DKIM, DMARC, etc). The last ones will help the email clients to autodiscover our email server
configuration and decrease the chance of our emails being marked as spam.

### NameCheap

I choose the NameCheap as my domain name register, so this tutorial is based on their system. But the same register and
concepts can be found in other companies.

### Host Records

The only things that we need to configure are the Host records and mail settings. We need to add some
A[<sup>[1]</sup>](https://www.namecheap.com/support/knowledgebase/article.aspx/9776/2237/how-to-create-a-subdomain-for-my-domain/),
 TXT, and SRV records.

We should add the recordings accordingly with the IP(s) and Protocol version(s) of our server.

### A Record (IPV4)

| **Type** |   **Host**   |       **IP**       |  **TTL**  |
| -------- | ------------ | ------------------ | --------- |
| A        | @            | \<Server IPV4\&gt; | Automatic |
| A        | www          | \<Server IPV4\&gt; | Automatic |
| A        | mail         | \<Server IPV4\&gt; | Automatic |
| A        | www.mail     | \<Server IPV4\&gt; | Automatic |
| A        | matrix       | \<Server IPV4\&gt; | Automatic |
| A        | autoconfig   | \<Server IPV4\&gt; | Automatic |
| A        | autodiscover | \<Server IPV4\&gt; | Automatic |

### AAAA Record (IPV6)

| **Type** |   **Host**   |       **IP**       |  **TTL**  |
| -------- | ------------ | ------------------ | --------- |
| AAAA     | @            | \<Server IPV6\&gt; | Automatic |
| AAAA     | www          | \<Server IPV6\&gt; | Automatic |
| AAAA     | mail         | \<Server IPV6\&gt; | Automatic |
| AAAA     | www.mail     | \<Server IPV6\&gt; | Automatic |
| AAAA     | matrix       | \<Server IPV6\&gt; | Automatic |
| AAAA     | autoconfig   | \<Server IPV6\&gt; | Automatic |
| AAAA     | autodiscover | \<Server IPV6\&gt; | Automatic |

The `@` count as our domain, and the other as our subdomains.

If we need a subdomain called `mastodon`, we just a add new record(s) with the `Host` field as `mastodon`
and the value the IPV4 and/or IPV6 of our server.

### CNAME Record

Commonly, if the client fails to autodicover the server configuration, it tries to connect to
`<protocol>.<Mail-Hostname>` like `imap.<Mail-Hostname>`. But our SMTP/IMAP/POP3 server is listening
to the `mail.<Mail-Hostname>`. So we will use these CNAMEs to redirect the clients to the actual
address that we are using.

| **Type** | **Host** |        **IP**        |  **TTL**  |
| -------- | -------- | -------------------- | --------- |
| CNAME    | imap     | \<Mail-Hostname\&gt;. | Automatic |
| CNAME    | smtp     | \<Mail-Hostname\&gt;. | Automatic |
| CNAME    | pop3     | \<Mail-Hostname\&gt;. | Automatic |

### Mail

To configure our email we need to add the MX record. We will add two, one to each email domain.

### MX Record

| **Type** | **Host** |         **IP**        | **Priority** |  **TTL**  |
| -------- | -------- | --------------------- | ------------ | --------- |
| MX       | @        | \<Hostname\&gt;.      |      0       | Automatic |
| MX       | mail     | \<Mail-Hostname\&gt;. |      0       | Automatic |

To test the records wait the TTL time (30 min for 'Automatic') and use the command below to check them.

``` console
$ sudo dig <Mail-Hostname> <Hostname> MX +noall +answer
```

### TXT Record

These informations are used by other mail servers to avoid [email spoofing](https://en.wikipedia.org/wiki/Email_spoofing)
and to check the legitimacy of our mail server by comparing email sender information with the TXT record in the DNS.
The SPF  TXT Records are better explained [here](https://wordtothewise.com/2014/06/authenticating-spf/) and
[here](https://mxtoolbox.com/dmarc/spf/spf-record-tags).

| **Type** |        **Host**       |                                  **Value**                                  |  **TTL**  |
| -------- | --------------------- | --------------------------------------------------------------------------- | --------- |
| TXT      | @                     | v=spf1 a mx:caedrium.com ip4:\<Server IPV4\&gt; ip6:\<Server IPV6\&gt; -all | Automatic |
| TXT      | \_dmarc               | v=DMARC1; p=reject; sp=none; adkim=s; aspf=s;                               | Automatic |
| TXT      | \_dmarc.mail          | v=DMARC1; p=reject; sp=none; adkim=s; aspf=s;                               | Automatic |
| TXT      | dkim.\_domainkey      | v=DKIM1; p=\<DKIM Public Key\&gt;                                           | Automatic |
| TXT      | dkim.\_domainkey.mail | v=DKIM1; p=\<DKIM Public Key\&gt;                                           | Automatic |
| TXT      | mail                  | v=spf1 a mx ip4:\<Server IPV4\&gt; ip6:\<Server IPV6\&gt; -all              | Automatic |
| TXT      | @                     | mailconf=https://autoconfig.caedrium.com/mail/config-v1.1.xml               | Automatic |

#### DKIM TXT record

DKIM key pairs are used to validate the domain against the DNS TXT record. Like in asymmetric cryptography, we have a
pair of keys, a public key (which may be known to others), and a private key (which may never be known by any except the
owner).
We should add the public key in the DKIM record, so other mail providers can use the public key to check that we
indeed are the owners of the private key.

Here we will follow what was done in the section [Shared DKIM Key]({% post_url 2021-06-28-Cloud-server %}#h-shared-dkim-key),
and therefore we will have two domains with the same DKIM key. If you chose separate keys, you will have to deal with two
different keys. No big deal.

To the DKIM TXT record[<sup>[2]</sup>](https://docs.iredmail.org/setup.dns.html)
[<sup>[3]</sup>](https://www.linuxbabe.com/mail-server/set-up-iredmail-multiple-domains-nginx),
we need an additional procedure to get the `<DKIM public key>` values.
In your server use the command below to show the public keys.

````console
$ sudo amavisd-new showkeys
; key#1 2048 bits, i=dkim, d=<Mail-Hostname>, /var/lib/dkim/<Mail-Hostname>.pem
dkim._domainkey.<Mail-Hostname>.      3600 TXT (
  "v=DKIM1; p="
  "MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAvaHhTHp9V/lHWpy2Ebav"
  "BNKVUOL45gt5x4kmOgnA1CRI95Kd8a9WTbjjLINq2n5fwsgGsk1auvHXCqNx4XIa"
  "rFgqvZlrO5JbaR1BLCOiQydHxtqHJfMMxyrCbiomr6DJPYZyxHjmS9r3qgVIUEnh"
  "TCE+Ho2XHe1NmW887uqLRJlrlWyMB/GONys5KEA4Q7cJDX9mDZ28MJ4A6BCVxT6m"
  "TP6XZ08zDoWLlm448ZisPyIajFTQ3u/R2lUeCOw6ZfKEziAPGFm/82IKfyZQUjSK"
  "p0XRtVnX+7LYqBFuqDz75VIfii6WHL+rYPIyifX9/3G7oOcSYHkDkPz/AJyAJmzs"
  "3wIDAQAB")

; key#2 2048 bits, i=dkim, d=<Hostname>, /var/lib/dkim/<Mail-Hostname>.pem
dkim._domainkey.<Hostname>.   3600 TXT (
  "v=DKIM1; p="
  "MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAvaHhTHp9V/lHWpy2Ebav"
  "BNKVUOL45gt5x4kmOgnA1CRI95Kd8a9WTbjjLINq2n5fwsgGsk1auvHXCqNx4XIa"
  "rFgqvZlrO5JbaR1BLCOiQydHxtqHJfMMxyrCbiomr6DJPYZyxHjmS9r3qgVIUEnh"
  "TCE+Ho2XHe1NmW887uqLRJlrlWyMB/GONys5KEA4Q7cJDX9mDZ28MJ4A6BCVxT6m"
  "TP6XZ08zDoWLlm448ZisPyIajFTQ3u/R2lUeCOw6ZfKEziAPGFm/82IKfyZQUjSK"
  "p0XRtVnX+7LYqBFuqDz75VIfii6WHL+rYPIyifX9/3G7oOcSYHkDkPz/AJyAJmzs"
  "3wIDAQAB")
```

Construct the DKIM TXT record by removing the quotes and copy the output of the command above into one line. Like this:

~~~
v=DKIM1; p=MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEA09JI4QyL9wgVhWMZru/LUYsP6AK6o5CrpLyO+8jsY0PzO3K6Z6K1i/yH388fgTUy7rHtk2KK3Ff6QeOORwmon73JzM5zc0Y4xkFn/lRMS4ZQVpbu5+eBpH7Uy0+4ZWRWbOZs34DZbAG3hpeh1GCWckIjIqOjdWOGBoSi30i3FzsFk+3UPMAzgM/6x9DpIU40EfA1YBsL8aGjIsolO7DhDUyhQhzsSdFtCJs2Rklb1k3WLrYvfiOgd7uRLAWKr1g7o4z2qM4OQTyTyNKpbA6+VKVKTljxcKbozQvPQYlrHpUK/dxuP608xFT413Jj+4VCQeRUK21ytz6xZ02riKEDGwIDAQAB
~~~

Add the value of each key to its respective TXT record. Wait for the propagation, and test with the following command.

``` console
$ sudo amavisd-new testkeys
TESTING#1 <Mail-Hostname>: dkim._domainkey.<Mail-Hostname> => pass
TESTING#2 <Hostname>: dkim._domainkey.<Hostname> => pass
```

#### DMARC

[Here](https://dmarc.org/) is a little explanation about what is the DMARC TXT record.

After add the DMARC record as shown in the table at [TXT Record](#h-txt-record), test it.

``` console
$ dig txt +short \_dmarc.caedrium.com
```

### SRV Records

We will use the SRV records to help the mail clients find the configuration of our email server. If you wanna learn a
little bit more about SRV records there's a
[short Cloudflare blog post](https://www.cloudflare.com/learning/dns/dns-records/dns-srv-record/) about it.

| **Type** |   **Service**  | **Protocol** | **Priority** | **Weight** | **Port** |          **Target**          | **TTL** |
| -------- | -------------- | ------------ | ------------ | ---------- | -------- | ---------------------------- | ------- |
| SRV      | \_imap         | \_tcp        | 0            | 1          | 143      | \<Mail-Hostname\&gt;         | 30 min  |
| SRV      | \_imaps        | \_tcp        | 0            | 1          | 993      | \<Mail-Hostname\&gt;         | 30 min  |
| SRV      | \_pop3         | \_tcp        | 0            | 1          | 110      | \<Mail-Hostname\&gt;         | 30 min  |
| SRV      | \_pop3s        | \_tcp        | 0            | 1          | 995      | \<Mail-Hostname\&gt;         | 30 min  |
| SRV      | \_smtps        | \_tcp        | 0            | 1          | 465      | \<Mail-Hostname\&gt;         | 30 min  |
| SRV      | \_smtp         | \_tcp        | 0            | 1          | 25       | \<Mail-Hostname\&gt;         | 30 min  |
| SRV      | \_submission   | \_tcp        | 0            | 1          | 587      | \<Mail-Hostname\&gt;         | 30 min  |
| SRV      | \_autodiscover | \_tcp        | 0            | 1          | 443      | autodiscover.\<Hostname\&gt; | 30 min  |
