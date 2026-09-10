# Jenkins Agent Farm Setup & Operations Guide

**Version:** 1.0  
**Last Updated:** 2026-09-10  
**Author:** DevOps Team  
**Environment:** Production Jenkins Master `https://jenkins.tech-bridge.biz`

---

## Table of Contents

1. [Overview](#overview)
2. [Architecture](#architecture)
3. [Prerequisites](#prerequisites)
4. [Step-by-Step Agent Setup](#step-by-step-agent-setup)
5. [Certificate Management](#certificate-management)
6. [Connection Details](#connection-details)
7. [Maintenance](#maintenance)
8. [Troubleshooting](#troubleshooting)
9. [Diagnostic Commands](#diagnostic-commands)
10. [Agent Inventory](#agent-inventory)
11. [Summary](#summary)

---

## 1. Overview

This document describes the design, implementation, and operation of a **Jenkins agent farm** that connects securely to a Jenkins master over HTTPS. The system replaces a manual, daily `screen`-based process with an automated, production-ready solution using `systemd`.

### 1.1 Before vs After

| Aspect               | Before (Manual)                                      | After (Automated)                                      |
|----------------------|------------------------------------------------------|--------------------------------------------------------|
| **Startup**          | Manual `screen` session every day                    | Automatic on system boot via `systemd`                 |
| **Restart on crash** | Manual intervention                                  | Automatic restart (`Restart=always`)                   |
| **Security**         | HTTP (unencrypted) and self-signed certificates      | HTTPS with trusted Let’s Encrypt certificate           |
| **Management**       | Ad-hoc `screen` commands                             | Standard `systemctl` and `journalctl`                  |
| **Scalability**      | Difficult to manage multiple agents                  | Easy to add new agents with a repeatable process       |
| **Logging**          | Screen buffer (ephemeral)                            | Centralized `journald` logs                            |

### 1.2 Key Benefits

- **Zero daily manual work** – agents start automatically on boot and restart on failure.
- **Secure communication** – all traffic encrypted with a trusted CA certificate.
- **Production ready** – managed by `systemd`, the standard Linux service manager.
- **Easy troubleshooting** – consistent logs and status commands across all agents.

---

## 2. Architecture

### 2.1 High-Level Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                   Jenkins Master                               │
│           https://jenkins.tech-bridge.biz                     │
│                    Java 21                                     │
│                 Let's Encrypt SSL                              │
└─────────────────────────────────────────────────────────────────┘
                              │
       ┌──────────────────────┼──────────────────────┐
       │                      │                      │
       ▼                      ▼                      ▼
┌─────────────────────────────────────────────────────────────────┐
│                        11 Agents                               │
├─────────────────────────────────────────────────────────────────┤
│ agent-51 │ agent-126 │ agent-127 │ agent-129 │ agent-130      │
│ agent-132 │ agent-137 │ agent-168 │ agent-194 │ agent-217     │
│ agent-28 (test)                                              │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 Components

| Component          | Description                                                                 |
|--------------------|-----------------------------------------------------------------------------|
| **Jenkins Master** | Central server at `https://jenkins.tech-bridge.biz`, running Java 21.       |
| **Agents**         | Worker nodes that execute jobs. Each runs a Java process (`agent.jar`).     |
| **systemd**        | Linux service manager that controls the agent process.                      |
| **Wrapper Script** | `start-agent.sh` – downloads `agent.jar` if missing and starts the agent.  |
| **WebSocket**      | Connection protocol between agent and master; persistent and efficient.    |
| **Let’s Encrypt**  | Trusted CA; Java and curl trust it automatically. No special flags needed. |
| **Java 21**        | Both master and agents use Java 21 to avoid version mismatch warnings.     |

### 2.3 Connection Flow

1. VM boots.
2. `systemd` starts `jenkins-agent.service`.
3. Service runs `/home/jenkins/start-agent.sh`.
4. Script downloads `agent.jar` (if not present).
5. Script executes Java agent with `-webSocket`.
6. Agent connects to `https://jenkins.tech-bridge.biz/`.
7. Agent registers as a node and becomes online.
8. If the process crashes, `systemd` restarts it after 10 seconds.

---

## 3. Prerequisites

Before setting up an agent, ensure the following:

- **Jenkins Master** is reachable via HTTPS at `https://jenkins.tech-bridge.biz/`.
- **DNS** resolves `jenkins.tech-bridge.biz` to the master’s IP on every agent VM.
- **Java 21** is installed on the agent (`java -version` should show `21.x.x`).
- A **`jenkins` user** and **`/home/jenkins`** directory exist (or will be created).
- A **unique secret** and **agent name** are generated from Jenkins:
  - Go to **Manage Jenkins → Nodes → New Node**.
  - Configure the node, then copy the secret and name.

---

## 4. Step-by-Step Agent Setup

The following steps are performed on **each agent VM**. Replace `<AGENT_NAME>` and `<SECRET>` with the actual values.

### 4.1 Create `jenkins` User and Directory

```bash
sudo useradd -m -s /bin/bash jenkins 2>/dev/null || true
sudo mkdir -p /home/jenkins
sudo chown -R jenkins:jenkins /home/jenkins
```

### 4.2 Create the Wrapper Script

The wrapper script is the heart of the agent startup. It ensures `agent.jar` is present and then runs the Java agent.

```bash
sudo tee /home/jenkins/start-agent.sh > /dev/null <<'EOF'
#!/bin/bash
cd /home/jenkins

# Download agent.jar if it doesn't exist
if [ ! -f agent.jar ]; then
    curl -sO https://jenkins.tech-bridge.biz/jnlpJars/agent.jar
fi

# Start the agent with HTTPS and WebSocket
exec java -jar agent.jar -url https://jenkins.tech-bridge.biz/ -secret <SECRET> -name "<AGENT_NAME>" -webSocket -workDir "/home/jenkins"
EOF

sudo chmod +x /home/jenkins/start-agent.sh
sudo chown jenkins:jenkins /home/jenkins/start-agent.sh
```

> **Important:** If the `jenkins` user’s `PATH` does not include `java`, use the full path in the `exec` line:
> ```bash
> exec /usr/lib/jvm/java-21-openjdk-amd64/bin/java -jar agent.jar ...
> ```

### 4.3 Create the systemd Service

```bash
sudo tee /etc/systemd/system/jenkins-agent.service > /dev/null <<'EOF'
[Unit]
Description=Jenkins Agent - <AGENT_NAME>
After=network.target

[Service]
Type=simple
User=jenkins
WorkingDirectory=/home/jenkins
ExecStart=/home/jenkins/start-agent.sh
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
EOF
```

### 4.4 Enable and Start the Service

```bash
sudo systemctl daemon-reload
sudo systemctl enable jenkins-agent
sudo systemctl start jenkins-agent
sudo systemctl status jenkins-agent
```

### 4.5 Verify Connection

```bash
sudo journalctl -u jenkins-agent -f
```

Look for:

```
INFO: WebSocket connection open
INFO: Connected
```

Then check the Jenkins dashboard under **Manage Jenkins → Nodes** – the agent should appear online.

### 4.6 Repeat for Each Agent

Repeat sections 4.1–4.5 on every agent VM, using the unique `<AGENT_NAME>` and `<SECRET>` for each.

---

## 5. Certificate Management

### 5.1 Self-Signed vs Let’s Encrypt

Initially we used a self-signed certificate on IP `192.168.0.17`. This required SSL bypass flags (`curl -k`, Java `-noCertificateCheck`) and manual certificate imports into Java’s truststore.

Later we switched to a **Let’s Encrypt** certificate for `jenkins.tech-bridge.biz`. Because Let’s Encrypt is a trusted CA, Java and curl trust it automatically. No special flags or imports are needed.

### 5.2 Importing a Certificate (If Needed)

If you ever use a self-signed or internal CA certificate, import it into Java’s truststore on each agent:

```bash
# Download the certificate
openssl s_client -connect jenkins.tech-bridge.biz:443 -showcerts </dev/null 2>/dev/null | openssl x509 -outform PEM > /tmp/jenkins-ca.crt

# Import into Java 21 truststore
sudo keytool -import -alias jenkins -keystore /usr/lib/jvm/java-21-openjdk-amd64/lib/security/cacerts -file /tmp/jenkins-ca.crt -storepass changeit -noprompt -trustcacerts
```

### 5.3 Updating Agents When the Certificate or Hostname Changes

If the master’s certificate or hostname changes, update each agent’s wrapper script and restart:

```bash
# Update hostname and remove SSL bypass flags
sudo sed -i 's|https://old-host|https://new-host|g' /home/jenkins/start-agent.sh
sudo sed -i 's/curl -skO/curl -sO/g' /home/jenkins/start-agent.sh
sudo sed -i 's/ -noCertificateCheck//g' /home/jenkins/start-agent.sh

# Clean and restart
sudo rm -rf /home/jenkins/remoting
sudo systemctl restart jenkins-agent
```

---

## 6. Connection Details

| Setting          | Value                                                                 |
|------------------|-----------------------------------------------------------------------|
| **Jenkins URL**  | `https://jenkins.tech-bridge.biz/`                                    |
| **Protocol**     | WebSocket (`-webSocket`)                                              |
| **Secret**       | Unique per agent, generated in Jenkins                                |
| **Name**         | Unique per agent, must match the node name in Jenkins                 |
| **Work Dir**     | `/home/jenkins` – where `agent.jar` and remoting data live            |
| **Java**         | Java 21 (`/usr/lib/jvm/java-21-openjdk-amd64/bin/java`)               |

---

## 7. Maintenance

### 7.1 Checking Status

```bash
sudo systemctl status jenkins-agent
```

### 7.2 Viewing Logs

```bash
sudo journalctl -u jenkins-agent -f          # live
sudo journalctl -u jenkins-agent -n 100      # last 100 lines
```

### 7.3 Restarting an Agent

```bash
sudo systemctl restart jenkins-agent
```

### 7.4 Adding a New Agent

Follow the same steps in Section 4, using the new secret and name. Update the service description to reflect the new agent name.

### 7.5 Removing an Agent

1. Stop and disable the service:
   ```bash
   sudo systemctl stop jenkins-agent
   sudo systemctl disable jenkins-agent
   ```
2. Remove the service file:
   ```bash
   sudo rm /etc/systemd/system/jenkins-agent.service
   sudo systemctl daemon-reload
   ```
3. Delete the agent from Jenkins: **Manage Jenkins → Nodes → (select agent) → Delete Agent**.

---

## 8. Troubleshooting

### 8.1 `java: not found`

**Symptom:**  
`/home/jenkins/start-agent.sh: line 10: exec: java: not found` in `journalctl`.

**Cause:**  
The `jenkins` user’s `PATH` does not include the Java binary.

**Fix:**  
Use the full path to Java in the wrapper script:
```bash
exec /usr/lib/jvm/java-21-openjdk-amd64/bin/java -jar agent.jar ...
```
Or ensure Java is in the system-wide `PATH` and the `jenkins` user inherits it.

---

### 8.2 `Handshake error` / `Did not receive X-Remoting-Capability header`

**Symptom:**  
Logs show repeated:
```
WARNING: Did not receive X-Remoting-Capability header
INFO: Failed to connect: Handshake error.
```

**Cause:**  
Usually a mismatch in the connection URL (e.g., still using HTTP, wrong port, or WebSocket not enabled on master).

**Fix:**
- Confirm the wrapper script uses `https://jenkins.tech-bridge.biz/` (no port, no IP).
- Verify WebSocket is enabled in Jenkins: **Manage Jenkins → Configure Global Security → WebSocket**.
- If WebSocket is not available, remove `-webSocket` from the wrapper script and use TCP mode (ensure the master’s TCP agent port is open).

---

### 8.3 `PKIX path building failed` / `unable to find valid certification path`

**Symptom:**  
Java throws `javax.net.ssl.SSLHandshakeException: PKIX path building failed`.

**Cause:**  
The certificate is not trusted by Java’s truststore.

**Fix:**
- If using Let’s Encrypt, ensure the master presents the full chain.
- For self-signed/internal CA, import the certificate into Java’s truststore (see Section 5.2).
- As a temporary workaround, add `-noCertificateCheck` to the Java command, but this is insecure.

---

### 8.4 `No subject alternative names present`

**Symptom:**  
`java.security.cert.CertificateException: No subject alternative names present`.

**Cause:**  
The certificate’s Common Name (CN) is an IP address but it lacks a Subject Alternative Name (SAN) for that IP. Modern Java requires SAN.

**Fix:**  
Regenerate the certificate with a SAN that includes the IP or DNS name. For Let’s Encrypt, this is handled automatically.

---

### 8.5 `Connection refused`

**Symptom:**  
`java.net.ConnectException: Connection refused` when trying to reach the master.

**Cause:**
- Wrong URL/port.
- Firewall blocking.
- Jenkins master not listening on that address/port.

**Fix:**
- Verify `curl -I https://jenkins.tech-bridge.biz` works from the agent.
- Check firewall rules (`sudo ufw status`).
- Ensure the master is running and accessible.

---

### 8.6 Service Fails with `status=7/NOTRUNNING`

**Symptom:**  
`systemctl status jenkins-agent` shows `Control process exited, code=exited, status=7/NOTRUNNING`.

**Cause:**  
The `ExecStartPre` command (often `curl`) failed, so the main process never started.

**Fix:**
- Remove `ExecStartPre` and move the download into the wrapper script.
- Test the download manually: `curl -sO https://jenkins.tech-bridge.biz/jnlpJars/agent.jar`.

---

### 8.7 Agent Shows Offline on Dashboard

**Symptom:**  
Agent process is running but Jenkins dashboard shows it offline.

**Cause:**
- Agent cannot reach master (network/DNS/firewall).
- Certificate issue.
- Secret or name mismatch.

**Fix:**
- Check agent logs: `sudo journalctl -u jenkins-agent -n 50 --no-pager`.
- Verify DNS: `ping jenkins.tech-bridge.biz`.
- Confirm the secret and name in the wrapper script match the Jenkins node configuration.
- Restart the agent: `sudo systemctl restart jenkins-agent`.

---

### 8.8 `UnsupportedClassVersionError` Warning

**Symptom:**  
Logs show `java.lang.UnsupportedClassVersionError` after connection.

**Cause:**  
Java version mismatch between master and agent (e.g., master uses Java 21, agent uses Java 17).

**Fix:**  
Install the same Java version on the agent as the master:
```bash
sudo apt install openjdk-21-jdk -y
sudo update-alternatives --config java
```
Then restart the agent.

---

### 8.9 Agent Connects but Immediately Disconnects

**Symptom:**  
Logs show `Connected` followed by disconnection or repeated handshake errors.

**Cause:**  
Often a stale `remoting` directory or corrupted `agent.jar`.

**Fix:**  
Clean the remoting directory and restart:
```bash
sudo rm -rf /home/jenkins/remoting
sudo systemctl restart jenkins-agent
```

---

## 9. Diagnostic Commands

```bash
# Check running agent processes
ps aux | grep agent.jar | grep -v grep

# Check service status
sudo systemctl status jenkins-agent

# View recent logs
sudo journalctl -u jenkins-agent -n 50 --no-pager

# Test HTTPS connectivity
curl -I https://jenkins.tech-bridge.biz

# Test certificate chain
openssl s_client -connect jenkins.tech-bridge.biz:443 -showcerts </dev/null 2>/dev/null | openssl x509 -noout -subject -issuer

# Check Java version
java -version

# Check if agent.jar exists
ls -la /home/jenkins/agent.jar

# Check DNS resolution
nslookup jenkins.tech-bridge.biz
ping jenkins.tech-bridge.biz

# Check firewall status
sudo ufw status
```

---

## 10. Agent Inventory

| Agent Name      | Status  | Java Version |
|-----------------|---------|--------------|
| agent-51        | Online  | 21           |
| agent-126       | Online  | 21           |
| agent-127       | Online  | 21           |
| agent-129       | Online  | 21           |
| agent-130       | Online  | 21           |
| agent-132       | Online  | 21           |
| agent-137       | Online  | 21           |
| agent-168       | Online  | 21           |
| agent-194       | Online  | 21           |
| agent-217       | Online  | 21           |
| agent-28 (test) | Online  | 21           |

> **Note:** Secrets are not listed here for security reasons. They are stored in the respective wrapper scripts on each agent.

---

## 11. Summary

We built a robust, automated Jenkins agent farm using:

- **systemd** for service management (auto-start, auto-restart).
- **HTTPS with Let’s Encrypt** for secure, trusted communication.
- **WebSocket** for efficient, persistent connections.
- **Wrapper scripts** for flexibility and easy updates.
- **Java 21** on both master and agents for compatibility.

This setup eliminates daily manual work, improves reliability, and provides a clear path for troubleshooting and scaling.

For any issues, start with the logs (`journalctl -u jenkins-agent -f`) and the diagnostic commands in Section 9.

---

**End of Document**
