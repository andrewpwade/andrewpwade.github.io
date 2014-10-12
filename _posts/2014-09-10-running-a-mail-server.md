---
title: Notes on running your own mail server
layout: post
---

{{ page.title }}
================

Since June 2013, when Edward Snowden published those leaked documents,
I have been running my own mail server. Why? Well, I wanted to have
complete control of my email, hopefully more privacy and to learn more
about the challenges of running a mail server.

I started with a very basic set up of Debian, shorewall, Postfix and
mutt, and later configured TLS, IMAP and SpamAssassin.

I run my mail service on a 512MB DigitalOcean virtual server in their
UK region with Debian Wheezy. I chose Debian because I wanted my mail
server to be rock solid and I trust the quality of Debian's packaging
and defaults.

Postfix is a good choice for an MTA: it is the default MTA on most
distributions these days and quite simple to set up and configure.

Below are some notes on running your personal mail server.

# Deliverability #

It is worth nothing that running your own mail server means that at
some point you will encounter problems delivering email to your
recipients. While you can [take steps to improve your chances](http://documentation.mailgun.com/best_practices.html]), at some point, a mail server will reject your mail, so it is useful to
configure your mail clients with an alternative SMTP server from a
major mail provider such as Gmail, Yahoo or Microsoft.

The most basic of steps to take include:

 - Checking if your IP is on any DNS blacklists.
 - Ensuring your reverse DNS record is set up correctly.
 - Setting up SPF records.

# Security

I decided I only wanted to communicate with my mail server over
encrypted channels so I configured IMAP with SSL on port 993 and
ESMTP (submission, on port 587).

# Spam

I actually receive surprisingly little spam but I decided to integrate
SpamAssassin anyway. With little free RAM and low volumes of mail, you
can set spamd's max-children to 1.

# Backups

With providers such as Gmail, Hotmail and Yahoo, you expect them to
take care of the availability of your email. When you run your own
mail, that is your job.

I maintain three local copies of my Maildir with a basic shell script that runs hourly via cron:

```
#!/bin/bash
set -e
set -o pipefail
cd /backups/mail/
[ -e Maildir.tar.gz.2 ] && rm -v Maildir.tar.gz.2
[ -e Maildir.tar.gz.1 ] && mv -v Maildir.tar.gz.1 Maildir.tar.gz.2
[ -e Maildir.tar.gz ] && mv -v Maildir.tar.gz Maildir.tar.gz.1
( GZIP=-1 tar hczf Maildir.tar.gz -C / /home/andrew/Maildir/ 3>&1 1>&2 2>&3 | \
 egrep  -v 'file changed as we read it|Removing leading' ) 3>&1 1>&2 2>&3
```

For remote backups, I use [tarsnap](http://www.tarsnap.com), mainly
due to the low cost (achieved with compression and deduplication) and
the owner's security credentials (he the Security Officer for
FreeBSD). It is backed by Amazon S3.

This is the shell script to backup my Maildir which runs every other hour via cron:

```
#!/bin/bash
DATESUFFIX=$(date +'%Y-%m-%dT%H:%M:%S')
ARCHIVE=andrewwade.-maildir-$DATESUFFIX-$(date +%s)
echo $ARCHIVE
/usr/local/bin/tarsnap -v --keyfile /root/tarsnap.r_w.key \
 --noisy-warnings -c -H -f "$ARCHIVE" /home/andrew/Maildir/
```

(N.B.: to restore, you need to know the exact filename.)

# Summary

I am under no illusion that my data is more hidden from governments
nor that my data is any more private due to it being hosted on a
server in the UK (as opposed to Gmail's server across the
globe). Further, my hosting provider has the ability to read the
contents of the server's disk whenever they choose and I, nor anyone,
knows what access the UK government have to their network.

That said, my main goal however was to learn more about email hosting
and to be able to boast to PFYs that of course I run my own mail
server!

I am sure I will receive more spam and face more deliverability issues
in the future, so there will be lots more tweaking to do.