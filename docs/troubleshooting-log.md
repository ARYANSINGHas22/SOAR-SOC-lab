# Troubleshooting Log

A day-by-day record of the infrastructure issues encountered while building this lab, and how
each was diagnosed and resolved. Kept here deliberately — the debugging process is as much a
demonstration of SOC/infra skill as the final working pipeline.

---

## Day 1 — Wazuh deployment and network setup

**Goal:** Get Wazuh 4.7.0 running single-node via Docker Compose on a headless Ubuntu VM
(`soc-brain`), with a Windows victim machine and Kali Linux attacker connected.

**Issue: subnet mismatch between VM and lab network**
The Windows and Kali VMs couldn't reach the Wazuh manager — they were landing on a different
subnet than `soc-brain`.

- **Diagnosis:** compared `ip addr` output across all three machines; found the VM's netplan
  config was using a static/incorrect subnet definition rather than picking up DHCP.
- **Fix:** corrected the netplan configuration to use DHCP, bringing all three machines onto the
  same subnet.
- **Verification:** confirmed bidirectional ping between all three hosts.

## Day 2 — Attack-detect pipeline validation

**Goal:** Confirm that attacker traffic from Kali actually triggers Wazuh detections.

**Issue: noisy/irrelevant alerts from BlueStacks**
An Android emulator (BlueStacks) running on the Windows host was generating background traffic
that cluttered the Wazuh alert stream, making it hard to isolate real detections.

- **Fix:** removed BlueStacks from the monitored environment entirely rather than trying to
  filter it out at the rule level — cleaner signal for a lab meant to demonstrate specific
  detections.

**Result:** confirmed both Windows failed-login and SMB brute-force traffic from Kali correctly
triggered Wazuh rules.

## Day 3 — Introducing n8n as the SOAR layer

**Goal:** Add automated alert routing so Wazuh detections reach Slack, not just the Wazuh
dashboard.

**Issue: VM ran out of disk space mid-build**
Installing n8n and its dependencies exhausted the VM's allocated disk space.

- **Diagnosis:** `df -h` showed the root partition at 100% utilization.
- **Fix:** extended the logical volume with `lvextend`, then grew the filesystem to match with
  `resize2fs`.
- **Why this matters:** a from-scratch VM disk sizing mistake that's easy to hit in any
  self-hosted lab — and a real operational skill (LVM management) beyond just SIEM usage.

**Build outcome:**
- Deployed n8n as a standalone Docker container on port `5678`, backed by a named volume
  (`n8n_data`) for persistence.
- Built the first working workflow: **Webhook → HTTP Request → Slack Incoming Webhook**.
- Created `custom-n8n` (shell wrapper) and `custom-n8n.py` inside `/var/ossec/integrations/` to
  let Wazuh call out to the n8n webhook on rule match.

**Key persistence architecture established:**
- Integration scripts are volume-mounted via `./config/wazuh_integrations`
- n8n must be attached to the `single-node_default` Docker network to reach the Wazuh manager
- n8n must run with `N8N_SECURE_COOKIE=false` (no HTTPS front-end in this lab)
- `ossec.conf` lives on the host at `~/wazuh-docker/single-node/config/wazuh_manager/ossec.conf`

## Day 4 — The layered failure day

**Goal:** Get the full pipeline (Wazuh → n8n → Slack) working end-to-end and persistent across
container restarts.

This was the densest debugging day — five distinct, layered issues stacked on top of each other.

**Issue 1: silent config overwrite**
Manual edits to `ossec.conf` kept disappearing after container restarts.

- **Diagnosis:** found a *second*, redundant `wazuh_manager.conf` volume mount in the Docker
  Compose file that was silently overwriting `ossec.conf` on every container start — the edits
  were being applied, then immediately clobbered.
- **Fix:** removed the redundant mount, consolidating to a single source of truth for the config.

**Issue 2: UID permission mismatch cascading through daemons**
After the config fix, Wazuh still failed to start cleanly — `wazuh-db` and a cascade of
dependent daemons (`analysisd`, `remoted`, and others) were failing.

- **Diagnosis:** the host filesystem's mounted config directories were owned by host UID 1000,
  but the Wazuh daemons inside the container run as UID 101. The daemons couldn't read/write
  their own data directories.
- **Fix:** corrected ownership on the host-mounted paths to match UID 101, resolving the
  cascading daemon failures.

**Issue 3: network orphaning on compose cycles**
n8n would connect to the Wazuh manager fine on first boot, but lost connectivity after any
`docker compose down` / `up` cycle.

- **Diagnosis:** n8n's container definition didn't explicitly declare the `single-node_default`
  network as *external*, so compose was recreating a new, disconnected network on each cycle
  instead of reattaching to the existing one.
- **Fix:** explicitly declared the external network in n8n's compose service definition.

**Issue 4: integration wrapper rejected by Wazuh's security check**
The `custom-n8n` shell wrapper was being silently ignored by Wazuh.

- **Diagnosis:** Wazuh's `wpopenv` security check enforces strict permission requirements on
  integration scripts — the wrapper's permissions didn't meet them.
- **Fix:** `chmod 550` on the wrapper script satisfied the check.

**Issue 5: Python dependency isolation**
`custom-n8n.py` needed the `requests` module, but installing it with system `pip` had no effect
inside the Wazuh container.

- **Diagnosis:** Wazuh ships its own bundled Python interpreter
  (`/var/ossec/framework/python/bin/pip3`), completely separate from the system interpreter.
  Installing to system Python was invisible to the integration script.
- **Fix:** installed `requests` using the bundled pip, targeted directly at the integration
  directory: `pip3 install requests -t /var/ossec/integrations/`.

**End-to-end result:** both attack scenarios (Windows failed login, SMB brute force) confirmed
firing through the full pipeline into Slack, verified persistent across multiple
`docker compose down/up` cycles.

---

