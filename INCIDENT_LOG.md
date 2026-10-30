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
## KaungThant- 6705140043
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

Fault 2
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


Fault 3
## Nay Shin Thant - 6705140032
## C1 — Client Could Not Reach the DNS Server

### Symptom (what we observed):

```text
[   0/2 ] C1   client cannot reach named at 192.168.19.10 (timeout/no status). Check listen-on, allow-query, firewall, and that it serves on lan0.
```

Testing from the client also timed out:

```bash
ip netns exec client dig
```

```text
;; communications error to 8.8.8.8#53: timed out
;; no servers could be reached
```

### Hypothesis (what we suspected and why):
A timeout means no answer came back at all. named was running (`systemctl status` showed active), so we suspected something between the client and named was dropping the packets, most likely a firewall rule. The checker also listed `listen-on` and `allow-query` as possible causes.

### Proof (the command that confirmed the cause):

```bash
iptables -L INPUT -n -v --line-numbers
```

```text
Chain INPUT (policy ACCEPT 2948 packets, 207K bytes)
num   pkts bytes target     prot opt in     out     source               destination
1       25  2153 DROP       17   --  *      *       0.0.0.0/0            0.0.0.0/0            udp dpt:53
```

Protocol 17 is UDP and port 53 is DNS. The counter (25 packets) proved this rule was dropping real DNS queries.

### Fix (what we changed):
We deleted the firewall rule:

```bash
iptables -D INPUT -p udp --dport 53 -j DROP
```

We also set `/etc/bind/named.conf.options` so named serves the LAN as an authoritative-only server:

```text
listen-on { 192.168.19.10; };
allow-query { 192.168.19.0/24; };
recursion no;
```

Then we checked with `named-checkconf -z` and restarted with `systemctl restart named`.

### Confirmation (output showing it now works):
The firewall chain was empty:

```text
Chain INPUT (policy ACCEPT 3324 packets, 231K bytes)
num   pkts bytes target     prot opt in     out     source               destination
```

A query to our server now got a reply in 1 ms instead of a timeout:

```text
;; ->>HEADER<<- opcode: QUERY, status: REFUSED, id: 62218
;; Query time: 1 msec
;; SERVER: 192.168.19.10#53(192.168.19.10) (UDP)
```

(REFUSED was caused by our typo, `www.novapeak. lab` with a space. What matters is that named replied, so the network path was open.)

The checker showed:

```text
[   2/2 ] C1   named is authoritative for novapeak.lab and reachable from the client.
```

### Found/proved by:
Nay Shin Thant

Fault 4 
## Hein Lwin Oo - 6705140006
## C2 — api Record Returned the Wrong IP

### Symptom (what we observed):
After the other DNS faults were fixed, three records were correct, but api was wrong:

text
[2.25/3 ] C2   api expected 192.168.19.40 got ['192.168.19.1']

### Hypothesis (what we suspected and why):
api returned a wrong IP, not nothing. So named was reachable and the zone was loaded, but it was serving wrong data for api. 192.168.19.1 is the gateway address (`option routers` in dhcpd.conf), not the API server.

### Proof (the command that confirmed the cause):
We edited the forward zone and set the api record to 192.168.19.40, then checked the file:

bash
cat /etc/bind/db.novapeak.lab

text
ns1 IN A 192.168.19.10
www IN A 192.168.19.20
mail IN A 192.168.19.30
api IN A 192.168.19.40

But the checker still failed:

text
[2.25/3 ] C2   api expected 192.168.19.40 got ['192.168.19.1']

This proved the file was now correct, but named was still serving the old zone it had loaded into memory.

### Fix (what we changed):
We restarted named so it loaded the corrected zone:

bash
systemctl restart named

### Confirmation (output showing it now works):

bash
ip netns exec client dig @192.168.19.10 api.novapeak.lab +short

text
192.168.19.40

text
[ 3.0/3 ] C2   all four A records correct
TOTAL: 25.0 / 25

### Found/proved by:
Hein Lwin Oo

Fault 5
## Phyo Thura Chit - 6705140015
## C9 — DNS Advertised by DHCP

### Symptom (what we observed):
The checker reported that DHCP was giving clients the wrong DNS server:

text
[   0/3 ] C9   DHCP advertises 8.8.8.8, expected the company DNS 192.168.19.10 -- this is the integration the whole project is about

### Hypothesis (what we suspected and why):
We thought the DNS server address in the DHCP configuration was wrong. Because of this, clients were using Google's public DNS (8.8.8.8) instead of the company DNS server, so they could not find the company's internal hostnames such as www.novapeak.lab.

### Proof (the command that confirmed the cause):

bash
cat /etc/dhcp/dhcpd.conf

text
subnet 192.168.19.0 netmask 255.255.255.0 {
        range 192.168.19.104 192.168.19.144;
        option routers 192.168.19.1;
        option domain-name-servers 8.8.8.8;
}

This confirmed that the DHCP configuration was advertising the wrong DNS server.

### Fix (what we changed):
We opened the file with nano /etc/dhcp/dhcpd.conf and changed the line to:

text
option domain-name-servers 192.168.19.10;

We checked the syntax before restarting:

bash
dhcpd -t -cf /etc/dhcp/dhcpd.conf

text
Config file: /etc/dhcp/dhcpd.conf
Database file: /var/lib/dhcp/dhcpd.leases
PID file: /var/run/dhcpd.pid

No errors, so we restarted the service with `systemctl restart isc-dhcp-server`.

### Confirmation (output showing it now works):

bash
systemctl status isc-dhcp-server --no-pager

text
Active: active (running) since Mon 2026-10-19 08:24:43 UTC; 9s ago
dhcpd[3659]: Listening on LPF/lan0/22:c5:ba:ee:85:23/192.168.19.0/24

The checker then showed:

text
[   3/3 ] C9   DHCP advertises the company DNS (192.168.19.10)

### Found/proved by:
Phyo Thura Chit

Fault 6
## Min Thein Kyaw - 6705140026
## C11 — Client Could Not Resolve www Through the Leased Resolver

### Symptom (what we observed):
Before the C9 fix:

text
[   0/3 ] C11  using the leased resolver 8.8.8.8, www did not resolve to 192.168.19.20 (got nothing).

After the C9 fix, the resolver changed, but www still did not resolve:

text
[   0/3 ] C11  using the leased resolver 192.168.19.10, www did not resolve to 192.168.19.20 (got nothing).

### Hypothesis (what we suspected and why):
C11 is the integration check: DHCP gives the client a DNS server, and the client uses it to look up www. The message changing from 8.8.8.8 to 192.168.19.10 proved the DHCP part (C9) was now correct. Since the client was now asking the right server but still got nothing, we suspected the company DNS server itself could not be reached from the client.

### Proof (the command that confirmed the cause):

bash
iptables -L INPUT -n -v --line-numbers

text
num   pkts bytes target     prot opt in     out     source               destination
1       25  2153 DROP       17   --  *      *       0.0.0.0/0            0.0.0.0/0            udp dpt:53

A firewall rule was dropping all DNS traffic to the server, so the leased resolver could never answer.

### Fix (what we changed):
No DHCP change was needed beyond C9. The firewall rule was deleted, and named's access settings were set (see the C1 section):

bash
iptables -D INPUT -p udp --dport 53 -j DROP

### Confirmation (output showing it now works):

text
[   3/3 ] C11  client resolves www via the leased resolver

### Found/proved by:
MT
