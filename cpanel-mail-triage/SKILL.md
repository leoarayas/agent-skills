---
name: cpanel-mail-triage
description: Diagnosticar problemas de correo en cPanel/WHM: envio, recepcion, login o timeout de puertos. Cubre dos caminos: (A) puertos y enrutamiento (Exim/Dovecot, CSF, nft, tcpdump) y (B) servicio del dominio (WHMCS, MX/DNS, whmapi). Usar ante "no llega el correo", "no puedo enviar", "timeout SMTP/IMAP" o "el cliente no puede configurar Outlook".
---

# Triage de correo en cPanel

> Consolida dos skills que se solapaban: `cpanel-mail-port-triage`, `cpanel-mail-service-diagnosis`.
> Fuente de verdad de este archivo: `dev-workspace/automations/skills-ops/skills/cpanel-mail-triage`.

## Que camino seguir

Arranca por A cuando el sintoma es de red ("timeout", "connection refused", "solo algunos remitentes").
Arranca por B cuando el sintoma es del dominio o del cliente ("no recibe", "no puede entrar al webmail", "MX mal apuntado").
Si no esta claro, hace B primero: identifica el servidor autoritativo por WHMCS antes de tocar nada.

## Camino A — puertos y enrutamiento

1. Confirm the target host before every investigation with `hostname -f`; do not diagnose or change another server by mistake.

2. Establish whether the mail stack is locally healthy using read-only commands:
   `systemctl is-active exim`, `systemctl status exim --no-pager -n 20`, `exim -bP daemon_smtp_ports`, and `ss -ltnp | awk 'NR==1 || /:(25|465|587|993)[[:space:]]/'`.
   Record Exim/Dovecot listeners and service state; do not restart services while they are listening and active.

3. Check CSF and packet policy before proposing a firewall change:
   `grep -E '^(TCP_IN|TCP6_IN|TCP_OUT|TCP6_OUT)' /etc/csf/csf.conf`, `csf -l`, and `nft list ruleset`.
   Treat configured ACCEPT rules as policy evidence, not proof that a source IP reaches them.

4. Verify actual mail flow from the server logs with `tail -80 /var/log/exim_mainlog` and `exim -bpc`.
   Recent external SMTP connections and successful `Completed` deliveries prove the mail service is not globally down.

5. Reproduce the reported path from the same external probe source with bounded TCP tests for 25, 465, 587, and 993, plus a known control port such as 2083.
   Compare results: a reachable control port with mail timeouts narrows the issue to port-specific filtering.

6. If the server is reachable through a management path, capture a bounded packet trace while reproducing the external TCP tests:
   `timeout 25 tcpdump -nn -i <public_nic> 'host <probe_ip> and tcp and (port 25 or port 465 or port 587 or port 993 or port 2083)'`.
   SYN arriving without SYN-ACK on mail ports while the same source completes a control-port handshake proves a local pre-accept filter; do not blame Exim or an upstream route.

7. For a local pre-accept filter, inspect Imunify before CSF exceptions:
   `ipset test i360.ipv4.blacklist <probe_ip>` and `imunify360-agent ip-list local list --by-ip <probe_ip>`.
   Capture the reason, scope, and expiry; an Imunify blacklist can drop traffic before the CSF ACCEPT rules.

8. For a mailbox that fails after a password change, map the domain and enumerate mailboxes before proposing a network fix: `whmapi1 listaccts search=DOMAIN searchtype=domain` and `uapi --user=CPUSER Email list_pops_with_disk`; confirm `suspended_login: 0` for the affected mailbox because an enabled mailbox can still reject a stale client credential.

9. Correlate the IP and mailbox in the Dovecot journal with `journalctl -u dovecot --since "YYYY-MM-DD HH:MM:SS" --no-pager | grep -F "IP"`; record each `Logged in` and `auth_failed` line because a successful TLS login proves the server and firewall are reachable while failed attempts identify the exact credential or username mismatch.
   Treat a local-part username followed by a full mailbox username as a client-configuration issue; instruct the user to use the full address and replace saved credentials on every client before considering a firewall exception.

10. Query `imunify360-agent ip-list local list --by-ip IP` after the blacklist check; distinguish `purpose=captcha` from a blacklist because a CAPTCHA/graylist entry applies to HTTP/cPanel and does not block IMAP. Record its expiration and do not whitelist a temporary CAPTCHA entry merely to restore mail access.

11. Report the evidence in this order: account/mailbox state, Dovecot authentication evidence, daemon/listener state, firewall policy, Imunify/CSF list result, and whether a narrow unblock is actually required. State separately whether the issue is global or only affects the tested probe IP.

12. Do not whitelist, remove blacklist entries, restart Exim/Dovecot, or alter CSF without explicit authorization.
   If an exception is approved, use the narrowest IP-specific whitelist, document the reason and expiry, then repeat the external port tests and capture the post-change result.
## Camino B — servicio, DNS y cuenta

Use this procedure when a client reports mail delivery, sending, receiving, or login problems for a domain hosted in cPanel.

1. Query the hosting billing record before probing infrastructure, because the service record identifies the authoritative cPanel server and account without relying on web DNS.
   - Run MCP.whmcs.whmcsGetServices({domain:'<domain>'}).
   - Record only the service status, server hostname/IP, account username, plan, and suspension state; never return or log password fields.
   - Confirm the service is active and note the server/IP for the next steps.

2. Query mail DNS independently of the website, because a site can point to Shopify or another platform while mail remains on cPanel.
   - Run dig +short MX <domain>, dig +short A mail.<domain>, dig +short TXT <domain>, and dig +short TXT _dmarc.<domain>.
   - Confirm that the MX target resolves to the cPanel server IP identified in step 1.

3. Test the advertised mail endpoints from an external host, because a cPanel login page alone does not prove SMTP or IMAP is available.
   - Run for p in 25 465 587 993; do timeout 5 bash -c "</dev/tcp/<server-ip>/$p" >/dev/null 2>&1 && echo "$p open" || echo "$p unreachable"; done.
   - Test the same ports against one known-good external SMTP host or another reachable company mail server, so an egress block is not misdiagnosed as a server outage.
   - Run curl -ksSI --connect-timeout 8 --max-time 15 https://<server-ip>:2083 | sed -n '1,16p' to distinguish an otherwise reachable cPanel host from a total host outage.

4. Inspect the mail server only after identifying its authoritative host, because SSH aliases and presumed server names can be stale.
   - Connect using the approved alias and run whmapi1 --output=json listaccts search=<domain> searchtype=domain.
   - If found, run uapi --user=<cpanel-user> Email list_pops_with_disk, exim -bpc, and grep -i '<domain>|<cpanel-user>' /var/log/exim_mainlog | tail -30.
   - Report mailbox quota, account suspension, queue state, and concrete Exim/Dovecot log errors without exposing addresses beyond what is necessary or any credentials.

5. When diagnosing a reported IP block, query the authoritative host before attributing the incident: inspect CSF/LFD deny, tempdeny, and allow lists; Imunify local IP events; Dovecot authentication logs; and Exim rejects/rate limits for the exact IP and the requested time window. Correlate the mailbox, event reason, timestamp, block expiry, and current state, because a later SMTP error does not prove an earlier access failure.

6. Use read-only cPHulk queries and never call a `flush`, `clear`, or purge endpoint, because those actions can destroy global authentication-history evidence needed for the diagnosis.

7. Escalate an infrastructure fault to the server owner with the server IP, affected domain, DNS mapping, failed ports, control comparison, and requested checks for Exim, Dovecot, firewall, and provider networking; do not change services or firewall rules without authorization.

8. Report concisely in Spanish: state whether the IP is currently blocked, distinguish historical versus current blocks, identify the mailbox/event only when corroborated by timestamped logs, and name the next action. Do not conflate website availability with mail availability.
