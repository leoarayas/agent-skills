---
name: cpanel-csf-alert-noise
description: "Cut CSF/LFD email alert floods on cPanel hosts without weakening blocking, and choose the brute-force layer (Imunify360 vs cPHulk)."
---

# cPanel CSF/LFD alert noise

Use when a cPanel host floods a mailbox with ConfigServer Firewall / LFD alert email, or when deciding whether to enable cPHulk or Greylisting next to Imunify360.

1. Identify the sending host before touching configuration, because the alert mail and the server you inspect can be different machines.
   - Take the From domain of the alert mail and resolve it (`getent hosts <domain>`). A daily Logwatch-style digest comes from a separate notification host, so CSF knobs on the inspected server cannot silence it.
   - Compare the alert knobs across sibling hosts of the same fleet: `grep -E "^(LF_[A-Z0-9_]*ALERT|[A-Z]*_ALERT|LF_DIRWATCH|LF_INTEGRITY) *=" /etc/csf/csf.conf`. A host whose noisy knobs are already "0" is the working reference, and the divergence identifies the cause.

2. Profile the alert mail by subject class before choosing knobs, and never classify infrastructure alerts by sender domain: they share one address (`root@<host>`) so domain-first grouping hides the composition.
   - Fetch only `BODY.PEEK[HEADER.FIELDS (SUBJECT)]` over the alert folder and count classes with a regex histogram (port scanning, distributed attack, SSH login alert, blocked IP, WHM/cPanel access, file alert).
   - Expect the largest class to be your own successful admin logins ("SSH login alert for user root from ..."), which belong to the SSH-login toggle rather than to the block alerts.

3. List the knobs that exist in the installed CSF version before proposing any.
   - `grep -nE "ALERT_LEVEL|RATE_PERCENT|RATE_PERIOD|MAXNOTIFY|NOTIFY_LOCALHOST" /etc/csf/csf.conf`; an empty result means this version has no rate limiting, so the only levers are the per-category `*_EMAIL_ALERT` toggles.
   - Confirm there is no separate `/etc/lfd/lfd.conf`; CSF 16.x keeps LFD settings inside `csf.conf`.
   - Do not apply `/usr/local/csf/profiles/disable_alerts.conf`: it also sets functional options such as LF_DIRWATCH, LF_INTEGRITY and LOGSCANNER to "0".

4. Change only the email toggles and keep every functional knob at its current value.
   - Set to "0": LF_EMAIL_ALERT, LF_TEMP_EMAIL_ALERT, LF_SU_EMAIL_ALERT, LF_WEBMIN_EMAIL_ALERT, LF_CONSOLE_EMAIL_ALERT, LT_EMAIL_ALERT, CT_EMAIL_ALERT, PS_EMAIL_ALERT, LF_PERMBLOCK_ALERT, LF_NETBLOCK_ALERT, LF_DISTFTP_ALERT, LF_DISTSMTP_ALERT.
   - Leave LF_SSH_EMAIL_ALERT at "1": silencing it also removes the alert that root logged in from an unknown address, which is a security signal rather than noise. Suppress your own addresses per IP with csf.ignore (step 9) instead of turning the toggle off.
   - Leave unchanged: LF_PERMBLOCK, LF_NETBLOCK, LF_DISTATTACK, LF_DIRWATCH, LF_INTEGRITY, LF_CPANEL_ALERT, LF_QUEUE_ALERT, LF_MODSECIPDB_ALERT, AT_ALERT.
   - Keep LF_CPANEL_ALERT enabled: WHM/cPanel root access is rare and high-value.

5. Back up and edit with an exact-match replace so only the intended keys move.
   - `cp -a /etc/csf/csf.conf /etc/csf/csf.conf.bak-alertnoise-$(date +%Y%m%d-%H%M%S)`
   - Rewrite `^KEY[[:space:]]*=[[:space:]]*"[^"]*"` for the target keys only, then `systemctl restart lfd` and confirm the service is active.

6. Prove the protection is intact, because silencing mail must not weaken blocking.
   - `pgrep -af "lfd -"` shows the daemon alive.
   - `grep -E "^(LF_SSHD|LF_FTPD|LF_IMAPD|LF_POP3D|LF_SMTPAUTH|LF_HTACCESS|LF_TRIGGER|LF_DISTATTACK|LF_DISTATTACK_UNIQ|LF_SELECT) *=" /etc/csf/csf.conf` shows the triggers still armed.
   - `iptables -S | grep -c "^-A INPUT"` shows rules present.
   - `grep -c "Blocked in csf" /var/log/lfd.log` is above zero and keeps rising.

7. Prove the mail stopped, reading the MTA log rather than the inbox.
   - `grep -i "lfd on <host>" /var/log/exim_mainlog | grep -oE "^YYYY-MM-DD HH:MM:SS" | sort | tail -3`; the newest timestamp must precede the change.
   - Count per hour with the full date: `grep -i "lfd on <host>" /var/log/exim_mainlog | grep -cE "^YYYY-MM-DD 20:"`. Compare only date-qualified buckets; a filter that compares the time field alone also matches other days and inflates the count.
   - `postqueue -p` must be empty.

8. State the audit-trail consequence in the report: LF_SSH_EMAIL_ALERT="0" also stops the `*SSH login*` line in lfd.log. The authoritative record of who logged in stays in `/var/log/secure` ("Accepted publickey for root", "Failed password for root", weekly rotations); check `grep -c "Accepted publickey for root" /var/log/secure` before claiming no traceability was lost.

9. Suppress only your own addresses per IP, and keep the alert toggle on. Put your static infrastructure addresses in `/etc/csf/csf.ignore` so LFD records them as "- ignored" and sends no mail, while LF_SSH_EMAIL_ALERT stays "1" and still fires for any root login from an address you do not control. Verify this empirically instead of assuming it: connect from the ignored address and confirm `grep "SSH login" /var/log/lfd.log | tail -3` shows the "- ignored" suffix, that the mailbox received nothing, and that `/var/log/secure` still holds the "Accepted publickey for root" entry. List only static infrastructure addresses, because a root login from a dynamic or residential address is exactly the event the owner needs to see.

10. Decide the brute-force layering from evidence before enabling anything.
    - Determine the active layer: `systemctl is-active imunify360.service imunify360-pam.service imunify360-wafd.service imunify360-webshield.service imunify-realtime-av.service imunify360-php-daemon.service`, `ls /etc/pam.d/*imunify*`, `ls -l /etc/cpanel/hulkd/hulkd.conf`, `grep -E "^CPHULK" /etc/csf/csf.conf`.
    - With Imunify360 active and its PAM hooks present, keep cPHulk off: both hook the same PAM authentication paths, so enabling both creates a double hook.
    - cPHulk does not cover IMAP or POP3; LFD does, so enabling cPHulk would not have stopped those credential-stuffing bursts.
    - Recommend rotating the passwords of the targeted mailboxes instead of stacking another layer.

11. Verify any push sink before routing security alerts into it: an unauthenticated sink lets a third party read or forge alerts. Publish a test message and read the account limits (`curl -s http://127.0.0.1:<port>/v1/account`) to confirm whether anonymous publishing or subscribing is allowed, and test reachability from the server's public address; a container-published port is reachable even when the host firewall is closed. Configure sink authentication and a non-public bind before use.
12. Reconcile any peer agent's recommendation against the state after your own change before acting, because a report produced earlier can recommend a fix your change already made obsolete. Reject the parts that trade visibility for silence with the reason stated (moving trusted addresses to csf.ignore, or routing alerts into an unauthenticated sink) rather than applying them or dropping them silently, and send the peer the correction so its records match the server.
13. Report in Spanish: what was silenced, what was deliberately kept, the blocking and MTA evidence, and the decisions left for the owner (password rotation, any order confirmation to verify). Never report the change as done without the verification output from steps 6 and 7.
