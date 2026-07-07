# Attack Scenarios

Detection scenarios implemented and confirmed end-to-end in this lab: attacker action → Wazuh
rule match → n8n webhook → Slack notification.

---

## Scenario 1: Windows failed login (brute force / repeated auth failure)

**MITRE ATT&CK mapping:** T1110 — Brute Force

**Attacker action:** Repeated failed authentication attempts against the Windows victim machine.

**Wazuh rule:** `60122`

**Detection logic:** Wazuh's Windows agent forwards Security event log entries (event ID 4625 —
failed logon) to the manager. Rule 60122 correlates repeated failed-logon events within a short
time window to flag a probable brute-force attempt, rather than alerting on a single failed
login.

**Pipeline flow:**
1. Failed logon attempts generated against the Windows victim
2. Wazuh agent ships the Security event log to the manager
3. Manager correlates the events and fires rule 60122
4. `custom-n8n.py` integration script POSTs the alert to the n8n webhook
5. n8n formats and forwards the alert to Slack

**Status:** ✅ Confirmed end-to-end, including persistence across `docker compose down/up`
cycles.

---

## Scenario 2: SMB brute force via nmap (from Kali)

**MITRE ATT&CK mapping:** T1046 — Network Service Discovery / T1110 — Brute Force

**Attacker action:** `nmap` SMB enumeration and brute-force scripts run from the Kali Linux
attacker machine against the Windows victim's SMB service.

**Wazuh rules:** `92032` / `92052`

**Detection logic:** These rules match on SMB protocol events indicating enumeration and
credential brute-force attempts against the SMB service — distinguishing reconnaissance/brute
force traffic from normal SMB file-sharing activity.

**Pipeline flow:**
1. `nmap` SMB scripts executed from Kali against the Windows victim
2. Wazuh agent/manager parses the resulting SMB protocol activity
3. Manager fires rules 92032/92052 on match
4. Same integration path as Scenario 1: `custom-n8n.py` → n8n webhook → Slack

**Status:** ✅ Confirmed end-to-end, including persistence across `docker compose down/up`
cycles.

---

---

## Roadmap: additional scenarios

Planned additions to broaden MITRE ATT&CK coverage beyond initial access/brute force:

- [ ] Privilege escalation (e.g., Windows local privilege escalation techniques)
- [ ] Lateral movement (e.g., PsExec / remote service creation detection)
- [ ] Persistence mechanisms (e.g., scheduled task or registry run-key creation)

Each new scenario should follow the same documentation pattern above: MITRE mapping, attacker
action, Wazuh rule ID(s), detection logic, and confirmed pipeline flow.
