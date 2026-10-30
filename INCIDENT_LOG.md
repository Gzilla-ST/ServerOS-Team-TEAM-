# Incident Log – NovaPeak (novapeak.lab), group lab01

All commands were run in our shared KillerCoda session and recorded in session.log (`script -a /root/session.log`). The first checker run (11/25) happened just before recording started; everything after it is in the log.

Starting score: **11.0 / 25** → final score: **25.0 / 25**

## Summary of faults (in the order we fixed them)

| Check | Fault | Root cause | Fix | Found/proved by |
|---|---|---|---|---|
| C4 | BIND config failed to load | Forward zone in named.conf.local pointed at a missing file (`db.MISSING`) | Changed it to file "/etc/bind/db.novapeak.lab"; and restarted BIND | Jack |
| C9 | DHCP advertised the wrong DNS server | dhcpd.conf had option domain-name-servers 8.8.8.8; | Changed it to `192.168.19.10`, validated with `dhcpd -t`, restarted isc-dhcp-server | Ko Phyo |
| C1 | Client could not reach named | iptables rule dropped all UDP port 53 traffic | Deleted the rule with `iptables -D INPUT -p udp --dport 53 -j DROP`; set `listen-on`, allow-query and recursion no in named.conf.options | Nay Shin Thant |
| C3 | Reverse lookup returned nothing | Same firewall rule (the reverse zone itself was valid) | Fixed by removing the firewall rule (see C1) | Jack |
| C11 | Client could not resolve www via the leased resolver | Wrong DNS in DHCP (C9), then the firewall rule (C1) | Fixed by the C9 and C1 fixes together | MT |
| C2 | api returned 192.168.19.1 instead of 192.168.19.40 | named was serving an old copy of the zone after the record was corrected | Set api IN A 192.168.19.40 in db.novapeak.lab and restarted named | Bruce |

## Score progress

| Step | Score |
|---|---|
| Start (before recording) | 11.0 / 25 |
| After C4 fix | 12.0 / 25 |
| After C9 fix | 15.0 / 25 |
| After C1 fixes (also fixed C3 and C11) | 24.25 / 25 |
| After C2 fix | 25.0 / 25 |

---
Fault 1
## C4 — Forward Zone Pointed at a Missing File

### Symptom (what we observed):
The checker reported that the BIND configuration failed to load, and no DNS records could be resolved:

```text
[   1/2 ] C4   named-checkconf -z FAILED; named-checkzone clean
[ 0.0/3 ] C2   ns1 expected 192.168.19.10 got nothing; www expected 192.168.19.20 got nothing; ...
```

### Hypothesis (what we suspected and why):
`named-checkconf -z` failed but `named-checkzone` was clean. This meant the zone data itself was fine, but the way `named.conf` referred to a zone was wrong. We suspected a zone declaration was pointing to a wrong or missing file.

### Proof (the command that confirmed the cause):

```bash
cat /etc/bind/named.conf.local
```

```text
zone "novapeak.lab" {
        type master;
        file "/etc/bind/db.MISSING";
};
```

We then listed the real zone files:

```bash
ls -l /etc/bind/db.*
```

```text
-rw-r--r-- 1 bind bind 212 Oct 19 08:04 /etc/bind/db.novapeak.lab
-rw-r--r-- 1 bind bind 233 Oct 19 08:04 /etc/bind/db.novapeak.lab.rev
```

This confirmed that the forward zone pointed to `db.MISSING`, which does not exist. The real file is `db.novapeak.lab`.

### Fix (what we changed):
We opened the file with `nano /etc/bind/named.conf.local` and changed the forward zone line to:

```text
file "/etc/bind/db.novapeak.lab";
```

Then we restarted BIND with `systemctl restart bind9`.

### Confirmation (output showing it now works):

```bash
named-checkconf -z
```

```text
zone novapeak.lab/IN: loaded serial 2024010101
zone 19.168.192.in-addr.arpa/IN: loaded serial 2024010101
```

The checker then showed:

```text
[   2/2 ] C4   named-checkconf -z clean; named-checkzone clean
```

## C3 — Reverse Lookup Returned Nothing

### Symptom (what we observed):

```text
[   0/2 ] C3   reverse lookup of 192.168.19.10 did not return ns1 (got nothing). Check the reverse zone and its PTR.
```

### Hypothesis (what we suspected and why):
The checker hinted at the reverse zone and its PTR record. But "got nothing" means no answer came back at all, not a wrong answer. C1 and C2 also got nothing at the same time. We suspected the reverse zone was fine and that something was stopping all DNS queries from reaching named.

### Proof (the command that confirmed the cause):
First, we checked that the reverse zone itself was valid:

```bash
named-checkzone 19.168.192.in-addr.arpa /etc/bind/db.novapeak.lab.rev
```

```text
zone 19.168.192.in-addr.arpa/IN: loaded serial 2024010101
OK
```

The reverse zone loaded with no errors, so the PTR data was not the problem. Then we checked the firewall:

```bash
iptables -L INPUT -n -v --line-numbers
```

```text
num   pkts bytes target     prot opt in     out     source               destination
1       25  2153 DROP       17   --  *      *       0.0.0.0/0            0.0.0.0/0            udp dpt:53
```

A rule was dropping all UDP port 53 (DNS) traffic, so reverse lookups never reached named.

### Fix (what we changed):
The firewall rule was removed (see the C1 section):

```bash
iptables -D INPUT -p udp --dport 53 -j DROP
```

No change to the reverse zone was needed.

### Confirmation (output showing it now works):

```text
[   2/2 ] C3   reverse lookup of 192.168.19.10 returns ns1.
```


### Found/proved by:
Jack
