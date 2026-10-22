# AI Usage and Critique

Group: [group name / number]  Members: [name — @github-username, name — @github-username]

## Part 3a: Declaration (required)

| AI tool | What we used it for |
|---|---|
| Claude (Anthropic) | Step-by-step guidance for the last fault (api A record returning 192.168.19.1), and help with git commits and the submission structure |
| [AI #2 name] | [what you used it for, or "only used for the Part 3c comparison"] |

## Part 3b: Confirm you diagnosed first

We completed our own diagnosis (INCIDENT_LOG.md) before the comparison below. Briefly, the faults WE found on our own:

1. named.conf.local pointed the novapeak.lab zone at a non-existent file (/etc/bind/db.MISSING), so named-checkconf -z failed (C4).
2. An iptables INPUT rule dropped all UDP port 53 traffic, so the client could not reach named (C1, C2, C3).
3. named.conf.options did not allow named to serve the LAN; we set listen-on 192.168.19.10 and allow-query 192.168.19.0/24 (C1).
4. dhcpd.conf advertised 8.8.8.8 as the DNS server instead of the company DNS 192.168.19.10 (C9, C11).
5. The api A record returned 192.168.19.1 instead of 192.168.19.40 (C2). We found this from the checker output; we used Claude for guidance on fixing it.

## Part 3c: Putting the AIs to the test

We gave the following to at least two AI tools: Claude and [AI #2 name].

| Fault (in our words) | We found it? | AI #1 (Claude) | AI #2 ([name]) | Was each AI actually right? |
|---|---|---|---|---|
| Zone file path pointed to db.MISSING | Yes | [what Claude said] | [what AI #2 said] | [right / wrong / partly] |
| iptables rule dropping UDP 53 | Yes | [what Claude said] | [what AI #2 said] | [right / wrong / partly] |
| listen-on / allow-query not serving the LAN | Yes | [what Claude said] | [what AI #2 said] | [right / wrong / partly] |
| DHCP advertised 8.8.8.8 instead of 192.168.19.10 | Yes | [what Claude said] | [what AI #2 said] | [right / wrong / partly] |
| api A record answered 192.168.19.1 | Yes (from checker) | Said the record was wrong and to fix the record, bump the serial, check and reload | [what AI #2 said] | Partly: the real cause was that named had not been restarted after editing |

For each AI suggestion that was WRONG or misleading, describe it and how you verified it was wrong:

- **Claude guessed the wrong zone file name.** Its first instructions used /etc/bind/db.company.local as an example path. That file does not exist on our system; `ls /etc/bind` showed the real file was db.novapeak.lab, and `nano /etc/bind/db.company.local` opened an empty new file.
- **Claude first assumed the zone record itself was wrong.** When we checked with `cat /etc/bind/db.novapeak.lab`, the file already said `api IN A 192.168.19.40`, but the checker still reported 192.168.19.1. The real problem was that named was still serving the old copy loaded in memory. After `systemctl restart named`, `dig @192.168.19.10 api.novapeak.lab +short` returned 192.168.19.40 and the checker gave 25/25.
- **Claude said bumping the SOA serial was needed.** In our case, restarting named alone fixed it without changing the serial. The serial matters for secondary servers and for `rndc reload`, but not for a full restart of a single primary server.
- [any wrong suggestions from AI #2]

## Part 3d: Verdict

- Where our human diagnosis beat the AIs: [for example: we read the real files and logs on the live system; the AI could only guess file names and the cause until we pasted our actual output]
- Where an AI genuinely helped: [for example: explaining the order fix → check → restart → verify with dig, and the git and submission steps]
- What we learned about trusting AI to diagnose a live system it cannot see: [for example: AI advice is only as good as the output you give it; always verify with cat, dig and the checker before trusting a suggestion]
