# agent-skills

Agent skills for Linux hosting operations: cPanel/WHM, CloudLinux, Imunify360, MySQL, Apache
(mod_lsapi), mail, and a few deployment/scraping/WordPress workflows. Written for
[Claude Code](https://claude.com/claude-code), Codex, OpenCode, OpenClaw and any tool that
reads `SKILL.md` folders.

These are sanitized versions of skills used daily on real shared-hosting servers. Host names,
customer names and internal aliases were replaced with placeholders (`<host-prod>`, `<alias>`,
`<usuario>`); adapt them to your environment before running anything.

## Install

```bash
# Claude Code / Codex / OpenCode (any skills-compatible agent)
npx skills add leoarayas/agent-skills --all

# or manually
git clone https://github.com/leoarayas/agent-skills ~/.agents/skills
```

With [Skills Manager](https://github.com/xingkongliang/skills-manager):

```bash
skills-manager-cli skills install https://github.com/leoarayas/agent-skills/tree/main/<skill>
```

## What's inside

| Skill | When it's useful |
|---|---|
| `server-triage` | a cPanel/WHM server looks down or slow: read-only first pass |
| `mysql-health` | MySQL/MariaDB slow, locks, too many connections, buffer pool tuning |
| `cloudlinux-lve` | a user hits CPU/IO/entry-process limits; LVE, MySQL Governor, CageFS |
| `imunify-response` | Imunify360 blocking a legitimate client, false positives, malware |
| `security-hardening` | hardening and auditing a cPanel host: SSH, CSF, cPHulk, 2FA |
| `apache-lsapi-tuning` | PHP slow under Apache + mod_lsapi (503, 508, TTFB) |
| `cpanel-account-ops` | create/suspend accounts, change plans, quotas, PHP version |
| `cpanel-csf-alert-noise` | CSF/LFD alert floods without weakening blocking |
| `cpanel-mail-triage` | mail delivery/sending/login failures: ports, DNS, WHMCS, Exim |
| `imap-mailbox-triage` | triage a mailbox: classify unread mail, reversible folders |
| `shared-server-docker-deploy` | deploy a Node app next to other Docker stacks, no downtime |
| `playwright-scraping-pitfalls` | Playwright scrapers: `__name` errors, memory growth, timeouts |
| `wp-plugin-i18n-release` | WordPress plugin i18n + release to the .org SVN |

## Conventions

- Each skill is `<name>/SKILL.md` with YAML frontmatter (`name`, `description`).
- `description` is what the agent reads to decide whether to load the skill: keep it specific
  and trigger-oriented.
- Diagnostic skills are read-only first. Anything that changes state (restart, kill, edit config)
  is proposed and confirmed before running.

## License

MIT — see [LICENSE](LICENSE).
