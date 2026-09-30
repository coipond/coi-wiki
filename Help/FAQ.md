# Frequently Asked Questions

Browse by category or use the troubleshooting quick links below.

## Categories

| Category | Questions | Description |
|----------|-----------|-------------|
| [Platform Comparisons](FAQ-Platform-Comparisons) | 5 | How Coi differs from Docker, DevContainers, Distrobox, plain VMs, and Docker Sandboxes |
| [Security and Trust](FAQ-Security-and-Trust) | 5 | What Coi protects against, credential handling, prompt injection, and trustworthiness |
| [Setup and Operation](FAQ-Setup-and-Operation) | 9 | Platform support, configuration, Docker-in-Coi, local AI models, and daily use |

## Troubleshooting Quick Links

**Agent freezes or hangs mid-task** - Almost always a full `/tmp`. See [Agent Freezes Completely Mid-Task](Troubleshooting#agent-freezes-completely-mid-task) for diagnosis and fixes.

**Container paused unexpectedly** - The security monitor detected a HIGH-severity event. Check the audit log, then `coi unfreeze <name>`. See [Container Paused by Security Monitoring](Troubleshooting#container-paused-by-security-monitoring).

**Container killed unexpectedly** - The security monitor detected a CRITICAL event (reverse shell, metadata endpoint access). See [Container Killed by Security Monitoring](Troubleshooting#container-killed-by-security-monitoring).

**coi shell fails with "security.privileged=true"** - Remove privileged mode from the Incus default profile. See [Coi Refuses to Start](Troubleshooting#coi-refuses-to-start-securityprivilegedtrue).

**Docker Compose fails inside the container** - Update Coi to get the three-step launch fix. See [Docker Compose Fails Inside Session Containers](Troubleshooting#docker-compose-fails-inside-session-containers).

**DNS fails during `coi build`** - systemd-resolved issue on Ubuntu. Coi auto-fixes this; see [DNS Issues During Build](Troubleshooting#dns-issues-during-build) for permanent fix.

**Many orphaned firewall (nftables) rules** - Usually caused by Docker running on the host. Run `coi clean --orphans`. See [Firewall Rules Accumulating](Troubleshooting#firewall-rules-accumulating-thousands-of-rules).

## More Questions?

[Join the Coi community on Slack](https://slack.karafka.io) for live discussion and support.

## See Also

- [Troubleshooting](Troubleshooting) - Diagnosis steps for common operational issues
- [Security Monitoring](Security-Monitoring) - How Coi detects and responds to threats
- [Architecture and Security Model](Architecture-and-Security-Model) - How Coi's defense layers work
- [Getting Started](Getting-Started) - New to Coi? Start here
