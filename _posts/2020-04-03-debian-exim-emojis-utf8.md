---
layout: post
title: "Supporting Emoji (UTF8) Emails on Debian Exim"
date: 2020-04-03 12:20
---

It's 2020 so lets get some emoji in our email 😎


Exim4 has had (experimental) support for UTF8 email since version [`4.86 RC1`](https://github.com/Exim/exim/commit/6023a6ad2ac0294879b14127f62795095da573b5) with it being mainlined in version [`4.87 RC1`](https://github.com/Exim/exim/commit/8c5d388a6e12d1a8bd4aa565920238f8a921414a).


However the Debian maintainers have only enabled the compile time options for the support very recently in [`4.93-6`](https://metadata.ftp-master.debian.org/changelogs//main/e/exim4/exim4_4.93-6_changelog). So Buster doesn't have support for it out the box.

Thankfully buster-backports has it, so [enable backports](https://backports.debian.org/Instructions/) and `apt install exim4-config exim4-daemon-heavy -t buster-backports`, I'm unable to tell you about whether or not you need to be accepting new versions of config files, but once you have that installed it's a simple case of enabling it.

If you have split config (which you should!) then just create a file called `/etc/exim4/conf.d/main/00_smtputf8_config` with the contents:

```
MAIN_SMTPUTF8_ADVERTISE_HOSTS = *
```

The explanation behind this is that Exim only advertises `SMTPUTF8` support to set list of hosts (which is blank by default), so we override that by telling it to advertise it to all hosts.

Once that is done it was a simple case of updating my aliases file with some emoji:

```
📧: contact
```

However nothing just works, Dovecot which is my LDA doesn't support UTF8

```
2020-04-03 12:37:16 1jKLZU-0000X4-Fp <**REDACTED**@imranh.co.uk>: dovecot_vmail transport output: lda(**REDACTED**): Fatal: Invalid -a parameter: Invalid character in localpart
2020-04-03 12:37:16 1jKLZU-0000X4-Fp == **REDACTED**@imranh.co.uk <📧@imranh.co.uk> R=vmail_user T=dovecot_vmail defer (0): Child process of dovecot_vmail transport returned 64 (could mean usage or syntax error) from command: /usr/lib/dovecot/deliver
```

Googling around there [doesn't seem to be much momentum](https://dovecot.org/list/dovecot/2018-September/112887.html) to support [RFC 6531](https://tools.ietf.org/html/rfc6531) in Dovecot.

As I'm using Exim as my MTA I'm able to tweak my Exim router to resolve and capture the non-emoji ASCII localpart of my email alias (contact) and feed that to Dovecot.

```
cat /etc/exim4/conf.d/router/170_vmail_aliases
  ...
  data = ${lookup{$local_part}lsearch{VMAIL_ALIASES}}
  ...
```

```
cat /etc/exim4/conf.d/transport/30_dovecot_vmail
  ...
  command = /usr/bin/spamc -e /usr/lib/dovecot/deliver -d $local_part@$domain -f $sender_address
  ...
```


OP success :) (or should that be 😎)