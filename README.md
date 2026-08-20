# malicious_ip_addresses

Aggregated IPv4 blocklist for firewalls, WAFs and intrusion prevention systems.

This repository mirrors several public threat intelligence feeds into a single, normalized, deduplicated list. It is regenerated automatically every 6 hours. No entry is ever added by hand.

**Maintained by [CodeIQ](https://codeiq.tech)** (Titouan Renard, Ajaccio, France).

---

## Files

| File | Content |
|---|---|
| `malicious_ip_addresses.txt` | Full list, all entries |
| `malicious_ip_addresses_aa.txt` | Entries 1 to 30 000 |
| `malicious_ip_addresses_ab.txt` | Entries 30 001 to 60 000 |
| `malicious_ip_addresses_ac.txt` | Entries 60 001 to 90 000 |
| `malicious_ip_addresses_ad.txt` | Entries 90 001 to 120 000 |
| `malicious_ip_addresses_ae.txt` | Entries 120 001 to 150 000 |
| `malicious_ip_addresses_af.txt` | Entries 150 001 to 180 000 |

The segmented files exist for appliances that cap the number of entries per external feed. **The number of segments is fixed**: a segment containing only its header is normal, it means the full list is currently shorter than that range. Segment files never disappear, so a device pointed at `_af.txt` will not start returning 404 when the list shrinks.

Raw URLs follow this pattern:

```
https://raw.githubusercontent.com/chelsinforce/malicious_ip_addresses/main/malicious_ip_addresses.txt
https://raw.githubusercontent.com/chelsinforce/malicious_ip_addresses/main/malicious_ip_addresses_aa.txt
```

## Format

Plain text, one IPv4 address per line, sorted numerically. Lines starting with `#` are comments and carry the generation metadata, the per source entry counts and the attributions.

```
# Aggregated malicious IPv4 blocklist
# Generated: 2026-08-20T09:41:12.004Z
# Entries: 98739
...
1.71.91.53
1.83.221.145
1.95.125.50
```

Every consumer listed below ignores comment lines natively.

## What is filtered out

- Malformed addresses and anything that is not a valid IPv4
- Private ranges (RFC 1918), loopback, link local, multicast and reserved space
- CGNAT space (`100.64.0.0/10`)
- Documentation and benchmark ranges (`192.0.2.0/24`, `198.18.0.0/15`, `198.51.100.0/24`, `203.0.113.0/24`)

Duplicates across sources are collapsed. An address present in three feeds appears once.

## Integration

### FortiGate (External Threat Feed)

Reference syntax for FortiOS 7.4, check it against your own version.

```
config system external-resource
    edit "CodeIQ-Malicious-IP"
        set type address
        set resource "https://raw.githubusercontent.com/chelsinforce/malicious_ip_addresses/main/malicious_ip_addresses.txt"
        set refresh-rate 360
        set status enable
    next
end
```

If your model caps the number of entries per feed, declare one resource per segment (`_aa`, `_ab`, ...) and group them in a firewall address group. Use the feed as a source in a DENY policy placed above your accept rules.

### fail2ban and iptables

Do not load 98 000 individual iptables rules. Use an ipset.

```bash
#!/usr/bin/env bash
set -euo pipefail

URL="https://raw.githubusercontent.com/chelsinforce/malicious_ip_addresses/main/malicious_ip_addresses.txt"

ipset create blocklist_new hash:ip -exist maxelem 200000
curl -fsSL "$URL" | grep -Ev '^\s*(#|$)' | while read -r ip; do
    ipset add blocklist_new "$ip" -exist
done

# Atomic swap, no window without protection
ipset create blocklist hash:ip -exist maxelem 200000
ipset swap blocklist_new blocklist
ipset destroy blocklist_new

iptables -C INPUT -m set --match-set blocklist src -j DROP 2>/dev/null \
    || iptables -I INPUT -m set --match-set blocklist src -j DROP
```

Schedule it every 6 hours with cron or a systemd timer. Persist the set with `ipset save` if you need it across reboots.

### Suricata

```
# suricata.yaml
default-rule-path: /etc/suricata/rules
...
```

```
# blocklist.rules
drop ip [!$HOME_NET] any -> $HOME_NET any (msg:"CodeIQ blocklist"; \
  iprep:src,blocklist,>,0; sid:1000001; rev:1;)
```

### pfSense / OPNsense

Add the raw URL as a **pfBlockerNG IPv4 list** (pfSense) or an **alias of type URL Table (IPs)** (OPNsense), with a 6 hour refresh.

## Update schedule

Every 6 hours, at minute 5. Each run fully regenerates every file: this is a mirror, not an accumulator. An address that disappears from all upstream sources disappears here too, which keeps the list in sync with the current state of the feeds rather than growing forever.

## License and attribution

This repository is distributed under the **GNU General Public License v3.0**, inherited from the Data-Shield IPv4 Blocklist it aggregates.

Required attributions:

- **Data-Shield IPv4 Blocklist** by Laurent Minne (duggytuxy), licensed under GPL-3.0
- **abuse.ch Feodo Tracker**, released under CC0 1.0
- **Binary Defense Systems Banlist**, public use only

**Resale of this data, or its inclusion in a paid product or service, is prohibited** by the Binary Defense terms. This repository and its content must remain freely available.

Domain and hostname indicators are published separately in [malicious_domains_addresses](https://github.com/chelsinforce/malicious_domains_addresses), under a different license.

## Disclaimer

This list is provided as is, with no warranty of any kind. It aggregates third party feeds and inherits their false positives. **Test before enforcing in production.** Blocking legitimate traffic because of an entry in this list is your responsibility, not the maintainer's.

An IP address appearing here means it was reported by at least one upstream source. It does not constitute an accusation against the owner of the address, which is very often a compromised host rather than a deliberate actor.

## Removal requests

If an address you own or operate appears here in error, note that this repository only mirrors upstream sources and adds nothing of its own. The durable fix is to request removal from the source feed, identified in the header of the file where you found the address.

You can also open an issue on this repository and the entry will be added to the local exclusion list within 24 hours, pending upstream removal.

## Contributing

Source suggestions are welcome via issues. A feed will only be added if its license explicitly permits public redistribution and the required attribution can be honored.
