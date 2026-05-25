# mail-bsd

Ansible playbooks to install and configure a full mail stack on OpenBSD and FreeBSD:

- **OpenSMTPD** — MTA
- **Dovecot 2.3** — IMAP server
- **Rspamd** — spam filtering and DKIM signing

> **Note:** These playbooks target Dovecot 2.3, which is the version currently available on both FreeBSD and OpenBSD ports. They will be updated once Dovecot 2.4 lands in the ports trees.

## Usage

```sh
ansible-playbook -i inventory_example.ini -l mail playbooks/mail_freebsd.yml
ansible-playbook -i inventory_example.ini -l mail playbooks/mail_openbsd.yml
```

## Inventory

Host variables `certdir` and `domain` must be set per host in the inventory:

```ini
[mail]
192.168.1.42 certdir=/etc/ssl/mail domain=domain.com
```

- `certdir` — path on the remote host where the TLS certificates are stored
- `domain` — mail domain to configure

## Requirements

### TLS certificates

The playbook expects certificates already present on the server at:

```
<certdir>/fullchain.pem
<certdir>/privkey.pem
```

## DNS records

| Type | Name                       | Value                              |
|------|----------------------------|------------------------------------|
| A    | mail.domain.tld            | `<your server IP>`                 |
| MX   | domain.tld                 | `10 mail.domain.tld`               |
| TXT  | domain.tld                 | `v=spf1 mx a:mail.domain.tld -all` |
| TXT  | dkim._domainkey.domain.tld | `v=DKIM1; k=rsa; p=<public key>`   |

### DKIM DNS record

The DKIM private key is generated automatically by the playbook and placed at:

- OpenBSD: `/etc/mail/dkim/private/domain.key`
- FreeBSD: `/usr/local/etc/mail/dkim/private/domain.key`

To get the public key value to put in the DNS TXT record:

```sh
# OpenBSD
cat /etc/mail/dkim/domain.pub | grep -v '^--' | tr -d '\n'

# FreeBSD
cat /usr/local/etc/mail/dkim/domain.pub | grep -v '^--' | tr -d '\n'
```

Paste the output as the value of `p=` in the DKIM TXT record.
