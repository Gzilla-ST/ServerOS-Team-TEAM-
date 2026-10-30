# AI Usage and Critique

Group: [Team_TEAM]  Members: [Kaung_Thant — @github-kaungthant79, Phyo_Thura_Chit — @github-Kpthura, Hein_Lwin_Oo - @github-Bruce-Oo, Min_Thein_Kyaw - @github-matty17-mt, Nay_Shin_Thant - @github-Gzilla-ST]

## Part 3a: Declaration (required)

| AI tool | What we used it for |
|---|---|
| Claude (Anthropic) | To find the file name of the company database of name configuration in order to fix named.conf.local file |
| Claude (Anthropic) | To find the errors in /etc/bind/named.conf.options and fix the file by filling the missing IP Addresses |
| Claude (Anthropic) | Diagnosis of the iptable INPUT errors and troubleshooting of it |
| Claude (Anthropic) | Step by step finding of API files and fixing the IP Addresses of each API |

## Part 3b: Confirm you diagnosed first

We completed our own diagnosis (INCIDENT_LOG.md) before the comparison below. Briefly, the faults WE found on our own:

1. named.conf.local pointed the novapeak.lab zone at a non-existent file (/etc/bind/db.MISSING), so named-checkconf -z failed (C4).
2. dhcpd.conf advertised 8.8.8.8 as the DNS server instead of the company DNS 192.168.19.10 (C9, C11).
3. The api A record returned 192.168.19.1 instead of 192.168.19.40 (C2). We found this from the checker output; we used Claude for guidance on fixing it.

## Part 3c: Putting the AIs to the test

We gave the following to one AI tool: Claude.

| Fault (in our words) | We found it? | AI #1 (Claude) | Was each AI actually right? |
|---|---|---|---|
| Zone file path pointed to db.MISSING | Yes | To replace it with right file | [right] |
| iptables rule dropping UDP 53 | Yes | To test with dig command and fix the configuration file with correct ip addresses | [right] |
| listen-on / allow-query not serving the LAN | No | To check the /etc/bind/named.conf.options and fix the file by filling the missing IP Addresses | [right] |
| DHCP advertised 8.8.8.8 instead of 192.168.19.10 | Yes | To use nano command to edit the configuration file and replace the right IP Address| [right] |
| api A record answered 192.168.19.1 | Yes (from checker) | Said the record was wrong and to fix the record, bump the serial, check and reload | Partly: the real cause was that named had not been restarted after editing |

## Part 3d: Verdict

- Where our human diagnosis beat the AIs: [We find and troubleshoot the congnitive errors with our system administration knowledge such as ls -ltr, cd, cat and finding out the errors accurately without time consuming]
- Where an AI genuinely helped: [pointing out the faulty files and step by step fixing the configuration files and reassuring to check the errors by restarting the system]
- What we learned about trusting AI to diagnose a live system it cannot see: [An AI cannot see the server, so it fills the gaps with the most common answer, and it sounds equally confident whether it is right or guessing. Claude assumed our zone file was called db.company.local (ours was `db.novapeak.lab`), and it assumed the api record in the file was wrong, when the file was already correct and named was simply still serving the old zone from memory. Our conclusion is that AI is useful for suggesting where to look and what order to do things in, but it is not proof.]
