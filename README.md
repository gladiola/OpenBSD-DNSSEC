# OpenBSD DNSSEC — Authoritative DNS Server Setup

This guide walks through configuring a DNSSEC-signed authoritative DNS server on OpenBSD using **NSD** (Name Server Daemon) and **ldns** signing tools.

---

## Table of Contents

1. [Overview](#overview)
2. [Prerequisites](#prerequisites)
3. [Install Packages](#install-packages)
4. [Configure NSD](#configure-nsd)
5. [Create a Zone File](#create-a-zone-file)
6. [Generate DNSSEC Keys](#generate-dnssec-keys)
7. [Sign the Zone](#sign-the-zone)
8. [Configure NSD to Serve the Signed Zone](#configure-nsd-to-serve-the-signed-zone)
9. [Enable and Start NSD](#enable-and-start-nsd)
10. [Publish the DS Record at Your Registrar](#publish-the-ds-record-at-your-registrar)
11. [Automate Zone Re-signing](#automate-zone-re-signing)
12. [Verify DNSSEC](#verify-dnssec)
13. [Mail DNS for Cloud-Hybrid Relays (mx1/mx2)](#mail-dns-for-cloud-hybrid-relays-mx1mx2)
14. [Troubleshooting](#troubleshooting)

---

## Overview

DNSSEC (Domain Name System Security Extensions) adds cryptographic signatures to DNS records, allowing resolvers to verify that responses are authentic and have not been tampered with. Setting it up requires:

- An **authoritative DNS server** (NSD) that serves the signed zone.
- **DNSSEC key pairs** — a Zone Signing Key (ZSK) and a Key Signing Key (KSK).
- A **signed zone file** produced by `ldns-signzone`.
- A **DS (Delegation Signer) record** published at your domain registrar to anchor the chain of trust.

---

## Prerequisites

- OpenBSD 7.4 or later (commands work on earlier releases with minor adjustments).
- Root or `doas` access.
- A registered domain name (e.g., `example.com`).
- Your server's public IP addresses to use as authoritative nameservers.
- Port 53 (UDP and TCP) open in `pf`.

---

## Install Packages

OpenBSD ships with `unbound` in base for recursive resolving, but NSD must be installed as a package. The `ldns` package provides the zone-signing tools.

```sh
doas pkg_add nsd ldns
```

> **Note:** `pkg_add` selects the version matching your OpenBSD release. Confirm with `pkg_info nsd ldns` after installation.

---

## Configure NSD

NSD's main configuration file lives at `/var/nsd/etc/nsd.conf`. A minimal authoritative-only configuration looks like this:

```conf
# /var/nsd/etc/nsd.conf

server:
    hide-version: yes
    verbosity: 1
    do-ip4: yes
    do-ip6: yes

    # Listen on all interfaces; restrict to specific IPs in production
    ip-address: 0.0.0.0
    ip-address: ::

    # NSD runs chrooted under /var/nsd by default on OpenBSD
    zonesdir: "/var/nsd/zones"
    logfile: "/var/log/nsd.log"
    pidfile: "/var/nsd/run/nsd.pid"

zone:
    name: "example.com"
    zonefile: "master/example.com.signed"
```

Paths inside the `zonesdir` are relative to `/var/nsd/zones/`. Create the directory for master zones:

```sh
doas mkdir -p /var/nsd/zones/master
```

---

## Create a Zone File

Create the unsigned zone file. The serial number format `YYYYMMDDnn` is conventional and makes it easy to track updates.

```sh
doas vi /var/nsd/zones/master/example.com
```

```dns
; /var/nsd/zones/master/example.com
$ORIGIN example.com.
$TTL 3600

@   IN  SOA ns1.example.com. hostmaster.example.com. (
            2024010101  ; serial
            3600        ; refresh
            900         ; retry
            604800      ; expire
            300 )       ; negative TTL

; Name servers
@       IN  NS  ns1.example.com.
@       IN  NS  ns2.example.com.

; A records for the name servers themselves
ns1     IN  A   203.0.113.1
ns2     IN  A   203.0.113.2

; Other records
@       IN  A       203.0.113.1
@       IN  AAAA    2001:db8::1
www     IN  A       203.0.113.1

; Cloud-hybrid mail relays (public ingress/egress)
mx1     IN  A       198.51.100.10
mx1     IN  AAAA    2001:db8:100::10
mx2     IN  A       198.51.100.20
mx2     IN  AAAA    2001:db8:100::20
@       IN  MX  10  mx1.example.com.
@       IN  MX  10  mx2.example.com.

; Mail authentication records used with relay egress
@               IN  TXT "v=spf1 ip4:198.51.100.10 ip4:198.51.100.20 ip6:2001:db8:100::10 ip6:2001:db8:100::20 -all"
mail._domainkey IN  TXT "v=DKIM1; k=rsa; p=<base64-public-key>"
_dmarc          IN  TXT "v=DMARC1; p=none; rua=mailto:postmaster@example.com; adkim=s; aspf=s"
```

Verify the zone is well-formed:

```sh
doas nsd-checkzone example.com /var/nsd/zones/master/example.com
```

Expected output: `zone example.com is ok`

---

## Generate DNSSEC Keys

DNSSEC uses two key pairs:

| Key | Purpose | Recommended algorithm |
|-----|---------|----------------------|
| **KSK** (Key Signing Key) | Signs the DNSKEY RRset; its hash becomes the DS record at your registrar | ECDSAP256SHA256 |
| **ZSK** (Zone Signing Key) | Signs all other records in the zone | ECDSAP256SHA256 |

Create a directory to hold your keys and generate them with `ldns-keygen`:

```sh
doas mkdir -p /var/nsd/zones/keys
cd /var/nsd/zones/keys

# Generate the KSK (-k flag marks it as a Key Signing Key)
doas ldns-keygen -a ECDSAP256SHA256 -b 256 -k example.com

# Generate the ZSK
doas ldns-keygen -a ECDSAP256SHA256 -b 256 example.com
```

Each `ldns-keygen` call produces two files: a `.key` (public key in DNS zone format) and a `.private` (private key). List them to confirm:

```sh
ls /var/nsd/zones/keys/
# Example output:
# Kexample.com.+013+12345.key     Kexample.com.+013+12345.private   <- KSK
# Kexample.com.+013+67890.key     Kexample.com.+013+67890.private   <- ZSK
```

**Protect the private keys:**

```sh
doas chmod 600 /var/nsd/zones/keys/*.private
doas chown _nsd:_nsd /var/nsd/zones/keys/*.private
```

---

## Sign the Zone

Use `ldns-signzone` to produce a signed zone file. The `-e` flag sets the signature expiry date; 30 days (`$(date -v +30d +%Y%m%d%H%M%S)` on OpenBSD) is a common interval.

```sh
cd /var/nsd/zones/master

# Calculate expiry 30 days from now (OpenBSD date syntax)
EXPIRY=$(date -v +30d +%Y%m%d%H%M%S)

doas ldns-signzone \
    -n \
    -e "${EXPIRY}" \
    -f example.com.signed \
    example.com \
    /var/nsd/zones/keys/Kexample.com.+013+12345.key \
    /var/nsd/zones/keys/Kexample.com.+013+67890.key
```

**Flag reference:**

| Flag | Meaning |
|------|---------|
| `-n` | Use NSEC (not NSEC3) for authenticated denial of existence |
| `-e` | Signature expiry timestamp (`YYYYMMDDHHmmSS`) |
| `-f` | Output file name |

The final two arguments are the KSK and ZSK public key files (order does not matter; `ldns-signzone` reads the corresponding `.private` files automatically).

Verify the signed zone:

```sh
doas nsd-checkzone example.com /var/nsd/zones/master/example.com.signed
```

---

## Configure NSD to Serve the Signed Zone

The `nsd.conf` snippet shown earlier already points at `example.com.signed`. Reload NSD to pick up the new file:

```sh
doas nsd-control reload
```

If NSD is not yet running, start it (see the next section).

---

## Enable and Start NSD

```sh
# Enable NSD to start at boot
doas rcctl enable nsd

# Start NSD now
doas rcctl start nsd

# Check status
doas rcctl check nsd
```

Open port 53 in `/etc/pf.conf` if it is not already permitted:

```pf
# /etc/pf.conf — add these lines (adjust interface name as needed)
pass in on egress proto { tcp udp } from any to any port 53
```

Reload pf:

```sh
doas pfctl -f /etc/pf.conf
```

---

## Publish the DS Record at Your Registrar

The **DS (Delegation Signer) record** links your zone's DNSSEC to its parent zone (e.g., `.com`). You submit it to your domain registrar's control panel.

Extract the DS record from the KSK:

```sh
doas ldns-key2ds -n -2 /var/nsd/zones/keys/Kexample.com.+013+12345.key
```

**Flag reference:**

| Flag | Meaning |
|------|---------|
| `-n` | Print to stdout instead of writing a file |
| `-2` | Use SHA-256 digest (recommended; algorithm ID 2) |

Example output:

```
example.com.	3600	IN	DS	12345 13 2 a1b2c3d4e5f6...
```

Log in to your registrar and enter the DS record details:
- **Key Tag:** `12345`
- **Algorithm:** `13` (ECDSAP256SHA256)
- **Digest Type:** `2` (SHA-256)
- **Digest:** `a1b2c3d4e5f6...`

Propagation typically takes minutes to a few hours depending on the registrar and TLD registry.

---

## Automate Zone Re-signing

DNSSEC signatures expire. A cron job that re-signs the zone before signatures expire prevents validation failures. Add the following to root's crontab (`doas crontab -e`):

```cron
# Re-sign example.com every 20 days (signatures valid for 30 days)
0 3 */20 * * cd /var/nsd/zones/master && \
    EXPIRY=$(date -v +30d +%Y%m%d%H%M%S) && \
    ldns-signzone -n -e "${EXPIRY}" -f example.com.signed \
        example.com \
        /var/nsd/zones/keys/Kexample.com.+013+12345.key \
        /var/nsd/zones/keys/Kexample.com.+013+67890.key && \
    nsd-control reload
```

> **Tip:** Increment the SOA serial before each re-sign to ensure secondary servers transfer the new zone. A helper script that patches the serial, re-signs, and reloads NSD is recommended for production use.

---

## Verify DNSSEC

### Query your server directly

```sh
# Check that DNSKEY records are present
dig @203.0.113.1 example.com DNSKEY +dnssec

# Check that RRSIG records are returned for A records
dig @203.0.113.1 example.com A +dnssec
```

Look for the `ad` (Authenticated Data) flag in the response header, and `RRSIG` records alongside each RRset.

### Use an online validator

- **Verisign DNSSEC Debugger:** https://dnssec-debugger.verisignlabs.com/
- **DNSViz:** https://dnsviz.net/

Both tools display the full chain of trust from the root zone down to your domain and highlight any configuration problems.

### Check DS propagation

```sh
# Query the parent zone's nameservers for the DS record
dig example.com DS +trace
```

---

## Mail DNS for Cloud-Hybrid Relays (mx1/mx2)

To align with the cloud-hybrid architecture used in [OpenBSD-Email](https://github.com/gladiola/OpenBSD-Email), publish only the two public relays as MX targets.

- `mx1.example.com` and `mx2.example.com` are Internet-facing SMTP relays.
- The on-prem mail host stays private and is not published as an MX record.
- SPF should authorize only relay egress IPs.
- DKIM selector TXT should match where signing occurs (relay or on-prem).
- Start DMARC with `p=none`, monitor reports, then tighten to `quarantine`/`reject`.

Required reverse DNS (PTR) for each relay must be configured at the IP provider:

```text
10.100.51.198.in-addr.arpa.  IN PTR mx1.example.com.
20.100.51.198.in-addr.arpa.  IN PTR mx2.example.com.
```

When DNSSEC signs the zone, these mail-policy TXT records (SPF, DKIM, DMARC) are signed like any other RRset, protecting them against in-transit tampering.

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---------|-------------|-----|
| `nsd-checkzone` reports errors | Syntax error in zone file | Review the zone file; check `$ORIGIN` and trailing dots |
| RRSIG records missing | Zone not signed or NSD serving unsigned file | Re-run `ldns-signzone`; verify `zonefile` in `nsd.conf` points to `.signed` |
| Signatures expired | Re-signing cron job missed | Run the sign command manually; check cron logs in `/var/cron/log` |
| `SERVFAIL` after DS published | DS does not match KSK | Re-run `ldns-key2ds` and resubmit correct values to registrar |
| NSD fails to start | Configuration or permission error | Run `nsd-checkconf /var/nsd/etc/nsd.conf`; check `/var/log/nsd.log` |
| Port 53 unreachable | pf blocking DNS | Verify pf rules with `pfctl -sr`; reload after changes |

### Useful log locations

```sh
tail -f /var/log/nsd.log       # NSD operational log
doas nsd-control stats          # Live NSD statistics
doas nsd-control zonestatus     # Zone load status
```

---

## References

- [NSD documentation](https://www.nlnetlabs.nl/projects/nsd/)
- [ldns man pages](https://www.nlnetlabs.nl/projects/ldns/) — `man ldns-signzone`, `man ldns-keygen`, `man ldns-key2ds`
- [RFC 4033](https://www.rfc-editor.org/rfc/rfc4033) — DNS Security Introduction and Requirements
- [RFC 4034](https://www.rfc-editor.org/rfc/rfc4034) — Resource Records for DNS Security Extensions
- [RFC 4035](https://www.rfc-editor.org/rfc/rfc4035) — Protocol Modifications for DNS Security Extensions
- [OpenBSD FAQ](https://www.openbsd.org/faq/)
