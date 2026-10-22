## <Kaung Thant - 6705140043> – C4 and C3

**Background.** The company DNS server (BIND, service `named`, IP 192.168.19.10) must answer questions about `novapeak.lab` (for example, "what is the IP of www?"). The checker tests this from the client's side (`ip netns exec client ...`), not from the server itself. My checks were C4 (is the BIND configuration valid?) and C3 (does the reverse lookup of 192.168.19.10 return ns1?). My scores: C4 1/2 → 2/2, C3 0/2 → 2/2.

### 1. Firewall blocked DNS (C3)

**How I found it**
- The first checker run said: *"C3 reverse lookup of 192.168.19.10 did not return ns1 (got nothing). Check the reverse zone and its PTR."* C1 also failed with "client cannot reach named (timeout/no status)".
- I did not trust the hint and tested from the client:
  `ip netns exec client dig @192.168.19.10 www.novapeak.lab`
  The result was `communications error to 192.168.19.10#53: timed out`.
- A timeout is different from an error answer. With a timeout, the question never arrived or the answer never came back. If named were running but broken, it would reply with an error such as SERVFAIL or NXDOMAIN. So I suspected the network path, not the DNS data.
- I checked the firewall: `iptables -L INPUT -n -v --line-numbers`. Rule 1 was `DROP  udp  ...  udp dpt:53`. I ran the command twice, and the counter changed from 0 pkts / 0 bytes to 3 pkts / 255 bytes. This proves the rule was catching the client's DNS queries.

**Analysis**
- DNS normally uses UDP port 53. The INPUT chain filters packets coming into the server. Rule 1 dropped every UDP packet to port 53, so named never received the queries.
- The reverse zone (`19.168.192.in-addr.arpa`) was not the problem. `named-checkconf -z` later reported it as `loaded serial 2024010101`. The checker's C3 message pointed at the PTR record, but that was only the symptom: the PTR could not be tested because the query was blocked.
- This kind of fault is not in a BIND config file, which matches the lab hint "faults are not always in a config file".

**How I fixed it**
- Deleted the rule: `iptables -D INPUT -p udp --dport 53 -j DROP`
- Restarted the service: `systemctl restart named`
- Checked the rule was gone with `iptables -L INPUT -n -v --line-numbers`.

**Proof**
The checker then showed C3 "reverse lookup of 192.168.19.10 returns ns1" (2/2).

### 2. listen-on / allow-query (C1 hint)

**How I found it**
- The C1 message said to check `listen-on`, `allow-query`, the firewall, and that named serves on lan0. I looked at the options file:
  `grep -n -E 'listen-on|allow-query' /etc/bind/named.conf.options`
  Line 3: `listen-on { any; };`  Line 4: `allow-query { any; };`

**Analysis**
- `listen-on` says which addresses named listens on. `allow-query` says who may ask it questions.
- Both were `any`, which is open, so they were not blocking the client. The cause of the timeout was the firewall (fault 1).

**How I fixed it**
- I tightened them to the server address and the lab network:
  `listen-on { 192.168.19.10; };`
  `allow-query { 192.168.19.0/24; };`
- Then `systemctl restart named`. This is safer than `any`, and the client (192.168.19.104, inside 192.168.19.0/24) can still query.

### 3. Missing zone file (C4)

**How I found it**
- The checker said: *"C4 named-checkconf -z FAILED; named-checkzone clean"* (1/2).
- After the firewall fix, the client could reach named, but dig returned `status: SERVFAIL` with `flags: qr rd` and no `aa`. SERVFAIL means named could not produce an answer. The missing `aa` flag means named was not authoritative for the zone, so it had no data for `novapeak.lab`.
- I ran `named-checkconf -z`, which checks the configuration and loads the zones. It printed:
  `zone novapeak.lab/IN: loading from master file /etc/bind/db.MISSING failed: file not found`
  `zone novapeak.lab/IN: not loaded due to errors.`
- I opened the zone definition: `cat -n /etc/bind/named.conf.local`. Line 3 was `file "/etc/bind/db.MISSING";`. The reverse zone on line 8 used a real file (`db.novapeak.lab.rev`).

**Analysis**
- The zone `novapeak.lab` was declared `type master`, but the file with its records did not exist. named could not load the zone and answered every query for it with SERVFAIL.
- The reverse zone loaded fine, so the fault was only in the forward zone declaration.

**How I fixed it**
- I edited `/etc/bind/named.conf.local` and changed line 3 to `file "<REAL_FILE>";`.
- Checked: `named-checkconf -z` now showed `zone novapeak.lab/IN: loaded serial 2024010101`.
- Restarted: `systemctl restart named`.

**Proof**
- `ip netns exec client dig @192.168.19.10 www.novapeak.lab` returned `status: NOERROR`, `flags: qr aa rd` and `www.novapeak.lab. 3600 IN A 192.168.19.20`.
- The checker showed C4 "named-checkconf -z clean; named-checkzone clean" (2/2) and C1 "named is authoritative for novapeak.lab and reachable from the client".

### What I learned
- Test from the client's point of view, not with `dig @127.0.0.1`.
- The kind of failure tells you where to look: timeout = network or firewall; SERVFAIL without `aa` = zone not loaded; NXDOMAIN or a wrong IP = wrong data in the zone.
- A checker message describes a symptom. The real cause can be somewhere else (C3 looked like a PTR problem but was a firewall rule).
