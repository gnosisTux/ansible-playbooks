# mail-bsd

Ansible playbooks to install and configure a full mail stack on **OpenBSD** and **FreeBSD**.

## Stack

| Component  | Role                            |
|------------|---------------------------------|
| OpenSMTPD  | MTA (send/receive, ports 25/465/587) |
| Dovecot 2.3 | IMAP server + Sieve filtering  |
| Rspamd     | Spam filtering and DKIM signing |

> **Note:** These playbooks target Dovecot 2.3, the version currently available in both FreeBSD and OpenBSD ports trees. They will be updated once Dovecot 2.4 is available.

---

## Requirements

### Control machine

- Ansible 2.9+
- Collection `community.general` (for `pkgng`)
- Collection `community.crypto` (for key generation)

```sh
ansible-galaxy collection install community.general community.crypto
```

### Target server

- OpenBSD 7.x or FreeBSD 14.x
- SSH access as `root`
- TLS certificates already present at the path defined by `certdir` (see Variables):

```
<certdir>/fullchain.pem
<certdir>/privkey.pem
```

### Open ports

The playbook does **not** manage firewall rules (`pf.conf`). Open these ports manually before running:

| Port | Protocol | Purpose                        |
|------|----------|--------------------------------|
| 25   | TCP      | SMTP (incoming mail)           |
| 465  | TCP      | SMTPS (authenticated clients)  |
| 587  | TCP      | SMTP submission (clients)      |
| 993  | TCP      | IMAPS (Dovecot)                |

---

## Inventory

Variables are defined per host in the inventory file. See `inventory_example.ini`:

```ini
[mail]
192.168.1.42 certdir=/etc/ssl/mail domain=domain.tld
```

| Variable  | Description                                      | Example          |
|-----------|--------------------------------------------------|------------------|
| `certdir` | Directory containing the TLS certificate files   | `/etc/ssl/mail`  |
| `domain`  | Mail domain                                      | `domain.tld`     |

---

## Usage

```sh
# FreeBSD
ansible-playbook -i inventory_example.ini -l mail playbooks/mail_freebsd.yml

# OpenBSD
ansible-playbook -i inventory_example.ini -l mail playbooks/mail_openbsd.yml
```

---

## What the playbook does

1. Installs packages: `opensmtpd`, `dovecot`, `dovecot-pigeonhole`, `rspamd`, `opensmtpd-filter-rspamd`, and the Python `cryptography` library.
2. Deploys configuration from Jinja2 templates for OpenSMTPD, Dovecot, and Rspamd.
3. Creates the DKIM key directory structure and generates a 2048-bit RSA key pair.
4. Generates Diffie-Hellman parameters for Dovecot (4096-bit).
5. Enables and restarts `dovecot`, `smtpd`, and `rspamd`.

### Notable behaviour

- **Mail location:** `~/Mail` with `~/Mail/Inbox` as the inbox (maildir, `LAYOUT=fs`). Standard folders (Drafts, Junk, Sent, Archive, Trash) are auto-created.
- **Junk folder:** auto-expunged after 30 days.
- **Sub-address delimiter:** `_` — so `user_tag@domain.tld` delivers to `user@domain.tld`.
- **Auth backend:** `bsdauth` on OpenBSD, `pam` on FreeBSD.
- **DKIM selector:** hardcoded to `dkim`.
- **Sieve:** enabled for both LDA and LMTP delivery.

---

## DNS records

| Type | Name                         | Value                               |
|------|------------------------------|-------------------------------------|
| A    | `mail.domain.tld`            | `<your server IP>`                  |
| MX   | `domain.tld`                 | `10 mail.domain.tld`                |
| TXT  | `domain.tld`                 | `v=spf1 mx a:mail.domain.tld -all`  |
| TXT  | `dkim._domainkey.domain.tld` | `v=DKIM1; k=rsa; p=<public key>`    |

### Getting the DKIM public key

The DKIM private key is generated automatically. The public key is placed at:

- **OpenBSD:** `/etc/mail/dkim/domain.pub`
- **FreeBSD:** `/usr/local/etc/mail/dkim/domain.pub`

Extract the value for the DNS TXT record:

```sh
# OpenBSD
cat /etc/mail/dkim/domain.pub | grep -v '^--' | tr -d '\n'

# FreeBSD
cat /usr/local/etc/mail/dkim/domain.pub | grep -v '^--' | tr -d '\n'
```

Paste the output as the value of `p=` in the `dkim._domainkey` TXT record.

---

## What the playbook does NOT manage

- Firewall rules (`pf.conf`)
- TLS certificate issuance and renewal (`acme-client` / `certbot`)
- DNS record creation
- Mailbox user creation (users must exist in the system — auth is via `bsdauth`/`pam`)
- Backups

---

## Known issues / TODO

- [ ] Update to Dovecot 2.4 once available in ports.
