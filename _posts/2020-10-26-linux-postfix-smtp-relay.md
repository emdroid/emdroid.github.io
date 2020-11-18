---
title: "Linux postfix SMTP relay e-mail delivery"
header:
  teaser: /assets/images/posts/email-delivery.png
toc: true
categories:
  - Operating Systems
  - Linux
tags:
  - os
  - linux
  - email
  - postfix
  - smtp
---

![Postman](/assets/images/posts/email-delivery.png){: .align-center .img-large}

Linux send-only postfix mailer SMTP relay configuration for error reporting and notifications.

<!-- more -->

## 1. Introduction

When building dedicated server machines like [NAS boxes]({% post_url 2020-07-13-project-ares-part-1-choosing-the-hardware %}), the **error reporting** becomes one of the essential maintenance tools.
However by default the Linux only delivers the mail reports locally.

The "postfix" mailer allows to setup forwarding of such notifications to a standard e-mail address.

This can be useful for various reports and notifications, for example:
- disk S.M.A.R.T. reports, warnings
  (including temperatures / overheating)
- RAID disk failures and rebuilds
- CPU / GPU overheating
- etc.

In particular, the basic mail forwarding is relatively simple.
However the tricky part is to not have those mails **rejected as spam** by the e-mail providers.
Therefore we can configure the "postfix" mailer to use the **SMTP relay** to a regular e-mail provider (Gmail, Yahoo, Hotmail etc.).

## 2. Postfix installation

```bash
# enter the admin console
sudo -i

# install the postfix mailer
apt install -y mailutils
```

**2.1. Basic initial configuration**

The system should ask for the basic options at the end of the installation.
If it doesn't (or if you need to reconfigure later) you can run the configuration manually:

```bash
# should ask at the end, if not, run:
dpkg-reconfigure postfix
```

The basic options:
- Mail configuration: **Internet site**
- System mail name:
  - if having a domain, use it
  - or use any chosen name ("my-host-name.com")

**2.2. Configuring the interfaces and destinations**

```bash
vim /etc/postfix/main.cf

# set the interface:
inet_interfaces = loopback-only

# change the destination line:
mydestination = localhost.$mydomain, localhost, $myhostname
```

The options:
- **inet_interfaces**: send-only postfix config  
  (only need to send the e-mail, no need to receive)
- **mydestination**: domains the machine will deliver locally, instead of forwarding to another machine (the default is to receive mail for the machine itself)

**2.3. Testing the setup**

The initial configuration can now be tested - you can use the "mail" command to send a test mail:

```bash
# restart the postfix service to pick up the new settings
systemctl restart postfix

# send the test e-mail
  echo "Testing sending mail." | mail \
    -r <sender_email_address> \
    -s "Test notification" \
    <target_email_address>
```

{% capture notice_contents %}
**<a name="mail-sender">Sender address</a>**

Note that the `"-r"` parameter is used to specify the sender address.
This is necessary for the e-mail to not be rejected at the destination (the postfix default "username@hostname" isn't accepted by most providers).

This is however a problem for the system tools notifications, as these tools usually don't allow to set the sender address explicitly (and use the "postfix" default sender).

Therefore we'll setup the SMTP relay later to circumvent this issue.
{% endcapture %}

{% include notice level="warning" %}

## 3. Setting up SMTP forwarding

**3.1. The motivation**

Here we will configure the e-mail sending via a **public e-mail provider** (can use Gmail, Yahoo, Hotmail, Yandex etc.).
This will help to avoid the e-mail notifications falling into the spam folder or being outright rejected.

This is also necessary to provide a proper **sender e-mail address** (if your machine is not configured as a part of Internet domain) - the standard postfix sender address follows the "username@hostname[.domain]" format, which is outright rejected by the most public e-mail providers (not even allowing it to reach the spam folder where you could whitelist it otherwise).  
The only provider I've seen to accept these (but still moving them into the spam folder) was Yandex.

This could normally be worked-around by the client application providing the sender address (which is the expected behavior in the fact).
But this doesn't work with the various **system tools** that might be sending the errors and notifications, as these **usually don't allow to set the sender address**.

**3.2. Preparing the sender e-mail address**

- this will be used as the **sender** address  
  (the address the e-mails will be coming _from_)
- it is recommended to create a **new separate e-mail account** for these purposes to not expose your main account  
  (for example "_my-server@gmail.com_")
- find out and note down the **e-mail provider SMTP server** address  
  (for example "_smtp.gmail.com_")

**3.3. Setting up the username/password**

- the sender username and password needs to be known by the "postfix" to authenticate when sending the e-mail via SMTP

- example for Gmail:

```bash
# create the SASL password file
vim /etc/postfix/sasl/sasl_passwd

# insert the SMTP record:
[smtp.gmail.com]:587 <username>@gmail.com:<password>

# generate the password DB
postmap /etc/postfix/sasl/sasl_passwd

# check the DB was created
ls -al /etc/postfix/sasl/sasl_passwd.db

# set the permissions to protect the files
chown root:root /etc/postfix/sasl/sasl_passwd /etc/postfix/sasl/sasl_passwd.db
chmod 0600 /etc/postfix/sasl/sasl_passwd /etc/postfix/sasl/sasl_passwd.db
```

{% capture notice_contents %}
**<a name="two-factor">Two-factor authentication</a>**

When the Two-factor authentication (2FA) is enabled (for the "sender" account), the e-mail provider might eventually refuse to send the message as the second authentication step is not performed by the "postfix".

Therefore additional setup might be needed, for example [generating an App Password](https://www.linode.com/docs/guides/configure-postfix-to-send-mail-using-gmail-and-google-apps-on-debian-or-ubuntu/) for Google.
{% endcapture %}

{% include notice level="info" %}

**3.4. Configuring the SMTP relay**

- example for Gmail:

```bash
# edit the postfix config
vim /etc/postfix/main.cf

# set the relay:
relayhost = [smtp.gmail.com]:587

# set the relay options:
# - enable SASL authentication
smtp_sasl_auth_enable = yes
# - disallow anonymous authentication
smtp_sasl_security_options = noanonymous
# - location of sasl_passwd
smtp_sasl_password_maps = hash:/etc/postfix/sasl/sasl_passwd
# - enable STARTTLS encryption
smtp_tls_security_level = encrypt
# - location of CA certificates
smtp_tls_CAfile = /etc/ssl/certs/ca-certificates.crt

# restart the postfix service to pick up the settings:
systemctl restart postfix
```

**3.5. Testing the setup**

```bash
# restart the postfix service to pick up the new settings
systemctl restart postfix

# send the test e-mail
  echo "Testing sending mail via SMTP relay." | mail \
    -r <sender_email_address> \
    -s "Test notification" \
    <target_email_address>
```

{% capture notice_contents %}
**<a name="smtp-sender">SMTP sender address</a>**

Note that we still need to specify the sender e-mail address, as the postfix would still use the default one otherwise.

And in particular, you actually need to provide the **exact sender address used by the SMTP account**, as the e-mail provider will reject any non-matching sender address.

The setup to not have to specify the sender address explicitly will be done in the next step.
{% endcapture %}

{% include notice level="info" %}

## 4. SMTP address rewriting

- the mail is sent using the "_user@host_" as sender by default, which is normally nor accepted by the SMTP relays (must match the provider mail username)

- therefore we'll translate it to a valid mail address

- see here for further details: [Postfix address rewriting](http://www.postfix.org/ADDRESS_REWRITING_README.html#generic)

**4.1. Add the address translation**

```bash
# create/open the file "generic"
vim /etc/postfix/generic

# insert your SMTP sender address translations
<user>@<host>   <username>@gmail.com

# generate the generic db
postmap /etc/postfix/generic
```

- the "&lt;user&gt;" is the username you want to translate, for example "root"
  (usually used as the system notifications sender)

**4.2. Set the address translation options**

```bash
# edit the postfix config
vim /etc/postfix/main.cf

# set the path
smtp_generic_maps = hash:/etc/postfix/generic
```

**4.3. Testing the setup**

```bash
# restart the postfix service to pick up the new settings
systemctl restart postfix

# send the test e-mail
  echo "Testing sending mail via SMTP relay (sender address translation)." | mail \
    -s "Test notification" \
    <target_email_address>
```

- note that we no longer have to set the sender address via the `"-r"` parameter

## 5. Local user aliases (optional)

This can be used to forward the mail sent to local users (e.g. "root") to an external address (e.g. "_destination.user@gmail.com_").  
This is optional, but **highly recommended**.

Note than until here, we were setting up the **sender address** (the mail address the notifications are sent _*from*_), this step is about setting up the **target address** (destination - the mail address the notifications are sent _*to*_).

In particular, most of the Linux system notifications are sent to the "root" user by default.
The tools usually allow to change the destination e-mail address (so that the notifications will be delivered to the chosen external address instead of just locally to "root" where no-one will read them).

However if you setup the aliases, you **don't have to change the system defaults** for the tools, so they **can still keep sending the notifications to "root"**, but they will be **forwarded** to your external address of choice automagically.

```bash
# edit the e-mail aliases
vim /etc/aliases

# add the aliases, for example (for root):
root:          <destination.user>@gmail.com

# generate the db
postalias /etc/aliases
```

**5.3. Testing the setup**

```bash
# restart the postfix service to pick up the new settings
systemctl restart postfix

# send the test e-mail
  echo "Testing sending mail via SMTP relay (target address translation)." | mail \
    -s "Test notification" \
    root
```

- note here that we are trying to send the e-mail to the "root" user - the alias should be taken and the mail forwarded to the other (external) address

## 6. Troubleshooting

**6.1. Checking the logs**

- check the "syslog" to see the messages being sent, or any sending errors  
  (recommended to open in a separate shell window):

```bash
tail -f /var/log/syslog
```

## Resources and references

- [How To Install and Configure Postfix as a Send-Only SMTP Server on Ubuntu](https://www.digitalocean.com/community/tutorials/how-to-install-and-configure-postfix-as-a-send-only-smtp-server-on-ubuntu-20-04)
- [Configure Postfix to Send Mail Using Gmail and Google Apps on Debian or Ubuntu](https://www.linode.com/docs/guides/configure-postfix-to-send-mail-using-gmail-and-google-apps-on-debian-or-ubuntu/)
- [Postfix address rewriting](http://www.postfix.org/ADDRESS_REWRITING_README.html#generic)

{% include abbrev domain="computers" %}
