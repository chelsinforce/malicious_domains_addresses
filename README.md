# malicious_domains_addresses

Aggregated blocklist of malicious and phishing domains, for DNS filtering, firewalls and web proxies.

This repository mirrors public threat intelligence feeds into a single, normalized, deduplicated list. It is regenerated automatically every 6 hours. No entry is ever added by hand.

> ⚠️ **Non commercial use only.** See the [License](#license-and-attribution) section. This constraint differs from the companion IPv4 repository.

---

## Files

| File | Content |
|---|---|
| `malicious_domains.txt` | Full list, all entries |
| `malicious_domains_aa.txt` | Entries 1 to 30 000 |
| `malicious_domains_ab.txt` | Entries 30 001 to 60 000 |
| `malicious_domains_ac.txt` | Entries 60 001 to 90 000 |
| `malicious_domains_ad.txt` | Entries 90 001 to 120 000 |
| `malicious_domains_ae.txt` | Entries 120 001 to 150 000 |
| `malicious_domains_af.txt` | Entries 150 001 to 180 000 |

The segmented files exist for appliances that cap the number of entries per external feed. **The number of segments is fixed**: a segment containing only its header is normal, it means the full list is currently shorter than that range. Segment files never disappear, so a device pointed at `_af.txt` will not start returning 404 when the list shrinks.

Raw URLs follow this pattern:

```
https://raw.githubusercontent.com/chelsinforce/malicious_domains_addresses/main/malicious_domains.txt
https://raw.githubusercontent.com/chelsinforce/malicious_domains_addresses/main/malicious_domains_aa.txt
```

## Format

Plain text, one domain per line, alphabetically sorted, lowercase, no scheme, no trailing dot, no wildcard prefix. Lines starting with `#` are comments and carry the generation metadata and the attributions.

```
# Aggregated malicious domains blocklist
# Generated: 2026-08-20T09:41:12.004Z
# Entries: 147302
...
000011.pro
0000f.top
00012tt.com
```

Upstream `hosts` file syntax (`0.0.0.0 example.com`) is parsed and reduced to the domain alone.

## What is filtered out

- Malformed domains, single labels, invalid TLDs, entries longer than 253 characters
- Bare IP addresses (they belong in the [IPv4 repository](https://github.com/chelsinforce/malicious_ip_addresses))
- An allowlist of major providers, to avoid catastrophic false positives:
  `google.com`, `gstatic.com`, `googleapis.com`, `microsoft.com`, `microsoftonline.com`,
  `office.com`, `office365.com`, `windows.net`, `live.com`, `apple.com`, `icloud.com`,
  `cloudflare.com`, `amazonaws.com`, `akamai.net`, `akamaiedge.net`, `github.com`,
  `githubusercontent.com`, `fortinet.com`, `fortiguard.com`
  (including all their subdomains)

Duplicates across sources are collapsed.

## Integration

### FortiGate (External Threat Feed, domain type)

Reference syntax for FortiOS 7.4, check it against your own version.

```
config system external-resource
    edit "CodeIQ-Malicious-Domains"
        set type domain
        set resource "https://raw.githubusercontent.com/chelsinforce/malicious_domains_addresses/main/malicious_domains.txt"
        set refresh-rate 360
        set status enable
    next
end
```

Then reference the resource in a DNS Filter profile as a remote category, or in a Web Filter profile. If your model caps entries per feed, declare one resource per segment.

### Pi-hole

```bash
# Admin interface: Adlists > add the raw URL
# Or via CLI:
sqlite3 /etc/pihole/gravity.db \
  "INSERT INTO adlist (address, enabled, comment) VALUES \
  ('https://raw.githubusercontent.com/chelsinforce/malicious_domains_addresses/main/malicious_domains.txt', 1, 'CodeIQ malicious domains');"
pihole -g
```

### AdGuard Home

Settings, then DNS blocklists, then Add blocklist, then Add a custom list with the raw URL. The plain domain format is parsed natively.

### Unbound (RPZ style local zone)

```bash
#!/usr/bin/env bash
set -euo pipefail

URL="https://raw.githubusercontent.com/chelsinforce/malicious_domains_addresses/main/malicious_domains.txt"
OUT="/etc/unbound/unbound.conf.d/blocklist.conf"

{
  echo "server:"
  curl -fsSL "$URL" | grep -Ev '^\s*(#|$)' | while read -r d; do
      echo "  local-zone: \"${d}\" always_nxdomain"
  done
} > "${OUT}.tmp"

mv "${OUT}.tmp" "$OUT"
unbound-checkconf && systemctl reload unbound
```

### Squid

```
acl malicious_domains dstdomain "/etc/squid/malicious_domains.txt"
http_access deny malicious_domains
```

Squid does not accept `#` comments inline in that file format, strip them at download time:

```bash
curl -fsSL "$URL" | grep -Ev '^\s*(#|$)' > /etc/squid/malicious_domains.txt
squid -k reconfigure
```

## Update schedule

Every 6 hours, at minute 5. Each run fully regenerates every file: this is a mirror, not an accumulator. A domain that disappears from all upstream sources disappears here too.

## License and attribution

This repository is distributed under the **Creative Commons Attribution NonCommercial 4.0 International license ([CC BY-NC 4.0](https://creativecommons.org/licenses/by-nc/4.0/))**, inherited from the Phishing Army Blocklist it aggregates.

You may copy, redistribute and adapt this list, provided that you:

- **credit** the original author, **Andrea Draghetti** ([Phishing Army](https://phishing.army/)), and link to the license
- **do not use it for commercial purposes**

Commercial use includes any integration into a paid product, a managed service billed to a client, or any offering primarily directed toward commercial advantage. If you need this data in a commercial context, contact the upstream source directly to obtain suitable terms.

> **Note on licensing**: this repository is deliberately kept separate from [malicious_ip_addresses](https://github.com/chelsinforce/malicious_ip_addresses), which is published under GPL-3.0. CC BY-NC and GPL are incompatible, the two datasets must never be merged into a single file.

## Disclaimer

This list is provided as is, with no warranty of any kind. It aggregates third party feeds and inherits their false positives. **Test before enforcing in production.** DNS blocking is brutal: a false positive takes down an entire service for all your users.

A domain appearing here means it was reported by at least one upstream source. Phishing domains are frequently hosted on compromised legitimate infrastructure, and a shared hosting domain can be reported because of a single malicious page.

## Removal requests

If a domain you own or operate appears here in error, note that this repository only mirrors upstream sources and adds nothing of its own. The durable fix is to request removal from the source feed, identified in the header of the file where you found the domain.

You can also open an issue on this repository and the entry will be added to the local exclusion list within 24 hours, pending upstream removal.

## Contributing

Source suggestions are welcome via issues. A feed will only be added if its license explicitly permits public redistribution and the required attribution can be honored. Feeds whose terms forbid derivative works or redistribution, such as OpenPhish or the abuse.ch URLhaus and ThreatFox datasets, are deliberately excluded.
