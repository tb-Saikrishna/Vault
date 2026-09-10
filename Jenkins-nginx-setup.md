# Secure Jenkins HTTPS Architecture with Nginx

## Detailed Implementation, Configuration, Troubleshooting, Certificate Renewal, and Expansion Guide

> **Purpose:** This document records the complete implementation used to secure the Jenkins controller with HTTPS, place Nginx in front of Jenkins, restrict direct Jenkins access, introduce internal DNS, support Java 21 agents, renew certificates, and provide a reusable pattern for future internal tools such as SonarQube.

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Original State](#2-original-state)
   - [Problems with the Original Design](#problems-with-the-original-design)
3. [Target Architecture](#3-target-architecture)
4. [Why Nginx Was Used](#4-why-nginx-was-used)
5. [Nginx Installation](#5-nginx-installation)
6. [Important Nginx Discovery](#6-important-nginx-discovery)
7. [Nginx Configuration Location](#7-nginx-configuration-location)
8. [HTTP to HTTPS Redirect](#8-http-to-https-redirect)
9. [Nginx HTTPS Reverse Proxy](#9-nginx-https-reverse-proxy)
10. [Explanation of the Nginx Configuration](#10-explanation-of-the-nginx-configuration)
11. [Why the Proxy Headers Were Added](#11-why-the-proxy-headers-were-added)
12. [Why WebSocket/Upgrade Headers Were Included](#12-why-websocketupgrade-headers-were-included)
13. [Initial Self-Signed Certificate](#13-initial-self-signed-certificate)
14. [Why the Self-Signed Certificate Was Not Enough](#14-why-the-self-signed-certificate-was-not-enough)
15. [Java 21 and SAN](#15-java-21-and-san)
16. [Internal DNS](#16-internal-dns)
17. [Company Wildcard Certificate](#17-company-wildcard-certificate)
18. [Certificate Contents](#18-certificate-contents)
19. [Certificate Chain](#19-certificate-chain)
20. [Runtime Certificate Location](#20-runtime-certificate-location)
21. [Private Key Handling](#21-private-key-handling)
22. [Certificate and Private Key Match](#22-certificate-and-private-key-match)
23. [Jenkins Localhost Hardening](#23-jenkins-localhost-hardening)
24. [Why a Systemd Override Was Used](#24-why-a-systemd-override-was-used)
25. [How Jenkins Reads the Listen Address](#25-how-jenkins-reads-the-listen-address)
26. [Applying the Jenkins systemd Override](#26-applying-the-jenkins-systemd-override)
27. [Why Jenkins Does Not Need HTTPS Internally](#27-why-jenkins-does-not-need-https-internally)
28. [TLS Termination](#28-tls-termination)
29. [Jenkins URL](#29-jenkins-url)
30. [Jenkins Agents](#30-jenkins-agents)
31. [Agent Systemd Configuration](#31-agent-systemd-configuration)
32. [Certificate Validation From an Agent](#32-certificate-validation-from-an-agent)
33. [Useful HTTPS Tests](#33-useful-https-tests)
34. [Why `curl -k` Was Used During Testing](#34-why-curl--k-was-used-during-testing)
35. [Browser Certificate Warning vs Mixed Content Warning](#35-browser-certificate-warning-vs-mixed-content-warning)
36. [Nginx Configuration Validation](#36-nginx-configuration-validation)
37. [Why Reload Instead of Restart](#37-why-reload-instead-of-restart)
38. [Certificate Renewal Process](#38-certificate-renewal-process)
39. [Step-by-Step Certificate Renewal](#39-step-by-step-certificate-renewal)
40. [Step 5: Build the Certificate Chain](#40-step-5-build-the-certificate-chain)
41. [Step 6: Back Up Existing Production Certificates](#41-step-6-back-up-existing-production-certificates)
42. [Step 7: Install the New Runtime Certificate](#42-step-7-install-the-new-runtime-certificate)
43. [Step 8: Validate Nginx](#43-step-8-validate-nginx)
44. [Step 9: Reload Nginx](#44-step-9-reload-nginx)
45. [Step 10: Verify the Live Certificate](#45-step-10-verify-the-live-certificate)
46. [Why the Jenkins Service Does Not Need Restart During Certificate Renewal](#46-why-the-jenkins-service-does-not-need-restart-during-certificate-renewal)
47. [Important Verification: Who Owns Each Port?](#47-important-verification-who-owns-each-port)
48. [Final Port Architecture](#48-final-port-architecture)
49. [Security Boundary](#49-security-boundary)
50. [Recommended File Inventory](#50-recommended-file-inventory)
51. [Important Principle: Source vs Runtime Configuration](#51-important-principle-source-vs-runtime-configuration)
52. [Useful Commands Cheat Sheet](#52-useful-commands-cheat-sheet)
53. [Troubleshooting Methodology](#53-troubleshooting-methodology)
54. [Common Failure Scenarios](#54-common-failure-scenarios)
55. [How to Add a New Internal Tool Using Nginx](#55-how-to-add-a-new-internal-tool-using-nginx)
56. [Step-by-Step Pattern for SonarQube](#56-step-by-step-pattern-for-sonarqube)
57. [Enable the New Tool](#57-enable-the-new-tool)
58. [Why Multiple Applications Can Share Port 443](#58-why-multiple-applications-can-share-port-443)
59. [One Nginx, Many Internal Tools](#59-one-nginx-many-internal-tools)
60. [Important Rule for New Tools](#60-important-rule-for-new-tools)
61. [Suggested Standard for Internal Hostnames](#61-suggested-standard-for-internal-hostnames)
62. [Certificate Reuse](#62-certificate-reuse)
63. [Future Certificate Strategy](#63-future-certificate-strategy)
64. [Operational Checklist](#64-operational-checklist)
65. [What Should Never Be Changed Casually](#65-what-should-never-be-changed-casually)
66. [Rollback Procedure](#66-rollback-procedure)
67. [Final Architecture Diagram](#67-final-architecture-diagram)
68. [Final Configuration Map](#68-final-configuration-map)
69. [Mental Model to Remember](#69-mental-model-to-remember)
70. [One-Line Interview Summary](#70-one-line-interview-summary)
71. [Short Operational Summary](#71-short-operational-summary)
72. [Production Philosophy](#72-production-philosophy)

---

## 1. Executive Summary

The Jenkins controller was originally exposed directly over HTTP:

```text
http://192.168.0.17:8080
```

The final architecture is:

```text
                         Internal Users / Jenkins Agents
                                      |
                                      | HTTPS 443
                                      v
                       jenkins.tech-bridge.biz
                                      |
                                      v
                             +----------------+
                             |      Nginx     |
                             | Reverse Proxy  |
                             | TLS Termination|
                             +----------------+
                                      |
                                      | HTTP localhost
                                      v
                             127.0.0.1:8080
                                      |
                                      v
                                +-----------+
                                |  Jenkins  |
                                +-----------+
```

**Key security improvements:**

- ✅ HTTPS is used for user and agent access.
- ✅ A trusted company/Let's Encrypt certificate is presented by Nginx.
- ✅ HTTP requests are redirected to HTTPS.
- ✅ Jenkins itself is bound to `127.0.0.1:8080`.
- ✅ Direct network access to Jenkins port `8080` is therefore removed.
- ✅ Internal DNS maps `jenkins.tech-bridge.biz` to `192.168.0.17`.
- ✅ Jenkins agents use the hostname rather than the IP address.
- ✅ Java 21 certificate/hostname validation is satisfied by the certificate SAN.
- ✅ Nginx handles TLS while Jenkins continues to run internally over HTTP.

---

## 2. Original State

The Jenkins controller was:

| Property | Value |
|---|---|
| Server IP | `192.168.0.17` |
| Jenkins Port | `8080` |
| Protocol | HTTP |

The original access URL was:

```text
http://192.168.0.17:8080/
```

The Jenkins process was listening on:

```text
*:8080
```

`*` means Jenkins was reachable on all network interfaces.

Conceptually:

```text
User
  |
  | HTTP
  v
192.168.0.17:8080
  |
  v
Jenkins
```

### Problems with the Original Design

#### 2.1 No transport encryption

HTTP does not provide TLS encryption. Sensitive information such as authentication/session traffic should not be sent over an unencrypted management interface.

#### 2.2 Direct application exposure

Jenkins itself was exposed directly to the network on port 8080.

#### 2.3 No centralized TLS layer

There was no dedicated reverse proxy managing HTTPS.

#### 2.4 Java 21 agent compatibility

The Jenkins agents use Java 21 and connect through HTTPS. Modern Java certificate/hostname validation requires correct certificate identity information, especially the SAN extension.

---

## 3. Target Architecture

The desired architecture was:

```text
                       Internal Network
                              |
                              v
                 https://jenkins.tech-bridge.biz
                              |
                              v
                     +----------------+
                     |      Nginx     |
                     |      :443      |
                     +----------------+
                              |
                              |
                              v
                     127.0.0.1:8080
                              |
                              v
                         Jenkins
```

**Important separation of responsibility:**

| Component | Responsibility |
|---|---|
| Internal DNS | Resolves Jenkins hostname to `192.168.0.17` |
| Nginx | Receives HTTP/HTTPS requests and reverse proxies |
| TLS certificate | Proves hostname identity and enables encrypted HTTPS |
| Jenkins | Runs the CI/CD application itself |
| systemd | Starts/stops Jenkins and provides runtime environment settings |
| Jenkins agents | Connect to Jenkins using the HTTPS hostname |

---

## 4. Why Nginx Was Used

Jenkins can technically provide HTTPS itself, but the chosen architecture places Nginx in front.

Nginx provides a centralized edge layer for:

- TLS termination
- HTTPS redirection
- Reverse proxying
- Hostname-based routing
- Future security controls
- Future application routing
- Centralized certificate management

The pattern is:

```text
Client
  |
  | HTTPS
  v
Nginx
  |
  | HTTP on localhost
  v
Application
```

This is reusable for other applications.

---

## 5. Nginx Installation

Nginx was installed on the Jenkins host.

**Ubuntu:**

```bash
apt update
apt install nginx -y
```

**Verify:**

```bash
nginx -v
```

**Check the service:**

```bash
systemctl status nginx --no-pager
```

Expected architecture:

```text
systemd
   |
   v
nginx.service
   |
   v
/usr/sbin/nginx
```

---

## 6. Important Nginx Discovery

Before configuration, it was discovered that another Nginx process was already listening on port `8001`. That process belonged to Docker.

**Evidence:**

```bash
cat /proc/<PID>/cgroup
```

showed a Docker scope.

This was important because it meant:

- The existing containerized Nginx was separate from the newly installed host Nginx.
- The host Nginx could safely be configured for ports 80/443 because those ports were free.

> ⚠️ **Always inspect existing listeners before binding a new reverse proxy.**

**Useful command:**

```bash
ss -tulpn
```

---

## 7. Nginx Configuration Location

The Jenkins Nginx site is configured in:

```text
/etc/nginx/sites-available/jenkins
```

The enabled configuration is exposed through:

```text
/etc/nginx/sites-enabled/jenkins
```

The enabled file is a symbolic link to the `sites-available` file.

**Verify:**

```bash
ls -l /etc/nginx/sites-enabled/
```

---

## 8. HTTP to HTTPS Redirect

The Jenkins Nginx configuration contains:

```nginx
server {
    listen 80;
    server_name jenkins.tech-bridge.biz;

    return 301 https://$host$request_uri;
}
```

**Meaning:**

```text
http://jenkins.tech-bridge.biz
            |
            v
       HTTP :80
            |
            v
    301 Permanent Redirect
            |
            v
https://jenkins.tech-bridge.biz
```

### Why 301?

`301 Moved Permanently` tells the client that the requested resource should be accessed through the new HTTPS URL.

**Test:**

```bash
curl -I http://jenkins.tech-bridge.biz
```

**Expected:**

```text
HTTP/1.1 301 Moved Permanently
Location: https://jenkins.tech-bridge.biz/...
```

---

## 9. Nginx HTTPS Reverse Proxy

The HTTPS server block is:

```nginx
server {
    listen 443 ssl http2;
    server_name jenkins.tech-bridge.biz;

    ssl_certificate     /etc/nginx/ssl/fullchain.pem;
    ssl_certificate_key /etc/nginx/ssl/server.key;

    client_max_body_size 100m;

    location / {
        proxy_pass http://127.0.0.1:8080;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;

        proxy_http_version 1.1;

        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}
```

---

## 10. Explanation of the Nginx Configuration

### `listen 443 ssl http2`

```nginx
listen 443 ssl http2;
```

Means:

- Nginx accepts HTTPS traffic on port 443.
- TLS is enabled.
- HTTP/2 can be negotiated.

### `server_name`

```nginx
server_name jenkins.tech-bridge.biz;
```

Nginx routes requests for this hostname to this server block. This is important because certificates and hostname-based routing work together.

### `ssl_certificate`

```nginx
ssl_certificate /etc/nginx/ssl/fullchain.pem;
```

This is the certificate chain presented to clients. It contains:

1. Server certificate
2. Intermediate certificate(s)

### `ssl_certificate_key`

```nginx
ssl_certificate_key /etc/nginx/ssl/server.key;
```

This is the private key corresponding to the server certificate.

> 🔒 **The private key must never be exposed or committed to Git.**

### `proxy_pass`

```nginx
proxy_pass http://127.0.0.1:8080;
```

This is the core reverse proxy directive.

It means:

```text
Client
  |
  | HTTPS 443
  v
Nginx
  |
  | HTTP
  v
127.0.0.1:8080
  |
  v
Jenkins
```

Nginx terminates TLS and forwards the request internally.

---

## 11. Why the Proxy Headers Were Added

### Host

```nginx
proxy_set_header Host $host;
```

Passes the original requested hostname to Jenkins. For example:

```text
jenkins.tech-bridge.biz
```

### X-Real-IP

```nginx
proxy_set_header X-Real-IP $remote_addr;
```

Passes the client's IP address to the backend.

### X-Forwarded-For

```nginx
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
```

Maintains the proxy chain of client IP addresses.

### X-Forwarded-Proto

```nginx
proxy_set_header X-Forwarded-Proto https;
```

Tells Jenkins:

> The original client connection was HTTPS.

This is important because Jenkins itself receives HTTP from localhost but needs to understand that the external URL is HTTPS.

---

## 12. Why WebSocket/Upgrade Headers Were Included

```nginx
proxy_http_version 1.1;
proxy_set_header Upgrade $http_upgrade;
proxy_set_header Connection "upgrade";
```

These headers allow protocols that require connection upgrades, including WebSocket-style communication. Keeping them in the reverse proxy configuration avoids problems with browser/live communication mechanisms.

---

## 13. Initial Self-Signed Certificate

During the first phase, HTTPS was tested using a self-signed certificate.

The initial certificate files were:

```text
/etc/nginx/ssl/jenkins.crt
/etc/nginx/ssl/jenkins.key
```

The certificate was generated using OpenSSL.

A self-signed certificate is useful for proving that:

- Nginx is listening on 443.
- TLS is functioning.
- Reverse proxying works.

However, a self-signed certificate is **not** automatically trusted by browsers or Java clients. That is why the browser displayed a security warning.

---

## 14. Why the Self-Signed Certificate Was Not Enough

There were two separate certificate concerns:

### 14.1 Trust

The certificate was not issued by a CA trusted by the client.

### 14.2 Identity / SAN

The certificate did not contain the required Subject Alternative Name.

The Jenkins agents use Java 21 and connect using HTTPS. The correct identity for the final environment is the DNS hostname:

```text
jenkins.tech-bridge.biz
```

---

## 15. Java 21 and SAN

The important certificate requirement is:

```text
Subject Alternative Name
```

The company certificate contains:

```text
DNS:*.tech-bridge.biz
DNS:tech-bridge.biz
```

Therefore:

```text
jenkins.tech-bridge.biz
```

is covered by the wildcard.

However, the certificate does **not** contain:

```text
IP:192.168.0.17
```

Therefore agents should use:

```text
https://jenkins.tech-bridge.biz
```

rather than:

```text
https://192.168.0.17
```

This is a key reason the hostname migration was necessary.

---

## 16. Internal DNS

An internal DNS record was created:

```text
jenkins.tech-bridge.biz -> 192.168.0.17
```

This is an internal DNS entry because Jenkins is intended for internal company users only.

**Verification:**

```bash
nslookup jenkins.tech-bridge.biz
```

**Expected:**

```text
Name:   jenkins.tech-bridge.biz
Address: 192.168.0.17
```

**DNS flow:**

```text
Client
  |
  | "What is jenkins.tech-bridge.biz?"
  v
Internal DNS
  |
  | 192.168.0.17
  v
Client connects to 192.168.0.17
```

The client uses the hostname for TLS validation while DNS resolves it to the private IP.

---

## 17. Company Wildcard Certificate

IT supplied three certificate components:

```text
CABUNDLE
CRT
Private Key
```

In the implementation these were stored as:

```text
/root/ssl-certificates/
```

Working files:

```text
/root/ssl-certificates/CABUNDLE.txt
/root/ssl-certificates/CRT.txt
/root/ssl-certificates/PRIVATEKEY.txt
```

A combined chain was created:

```text
/root/ssl-certificates/fullchain.pem
```

---

## 18. Certificate Contents

The certificate was verified using OpenSSL.

**SAN:**

```text
DNS:*.tech-bridge.biz
DNS:tech-bridge.biz
```

**Issuer:**

```text
Let's Encrypt R12
```

This means:

```text
jenkins.tech-bridge.biz
```

is covered by:

```text
*.tech-bridge.biz
```

---

## 19. Certificate Chain

The server certificate and intermediate CA bundle were combined into:

```text
fullchain.pem
```

**Command used:**

```bash
cat CRT.txt CABUNDLE.txt > fullchain.pem
```

If files do not contain a terminating newline, a safer construction is:

```bash
(
    cat CRT.txt
    echo
    cat CABUNDLE.txt
) > fullchain.pem
```

**Verify PEM boundaries:**

```bash
grep -n "BEGIN CERTIFICATE\|END CERTIFICATE" fullchain.pem
```

**Correct structure:**

```text
-----BEGIN CERTIFICATE-----
...
-----END CERTIFICATE-----
-----BEGIN CERTIFICATE-----
...
-----END CERTIFICATE-----
```

**Incorrect structure:**

```text
-----END CERTIFICATE----------BEGIN CERTIFICATE-----
```

The latter causes errors such as:

```text
PEM_read_bio_X509_AUX() failed
PEM routines::bad end line
```

---

## 20. Runtime Certificate Location

> ⚠️ **This is an important distinction.**

The certificate files received from IT are kept in the working/staging directory:

```text
/root/ssl-certificates/
```

But Nginx uses the production copies in:

```text
/etc/nginx/ssl/
```

Current production files:

```text
/etc/nginx/ssl/fullchain.pem
/etc/nginx/ssl/server.key
```

The Nginx configuration references exactly these paths:

```nginx
ssl_certificate     /etc/nginx/ssl/fullchain.pem;
ssl_certificate_key /etc/nginx/ssl/server.key;
```

Therefore:

```text
/root/ssl-certificates/
        |
        | copy
        v
/etc/nginx/ssl/
        |
        v
Nginx
```

> ⚠️ Do **not** assume that changing a file in `/root/ssl-certificates/` changes the live Nginx certificate. Nginx uses `/etc/nginx/ssl/`.

---

## 21. Private Key Handling

The production private key is:

```text
/etc/nginx/ssl/server.key
```

It was copied from:

```text
/root/ssl-certificates/PRIVATEKEY.txt
```

**Recommended permissions:**

```bash
chmod 600 /etc/nginx/ssl/server.key
chown root:root /etc/nginx/ssl/server.key
```

The private key should be readable only by the necessary account/root-controlled process.

> 🔒 **Never:**
> - Commit it to Git.
> - Send it in chat.
> - Put it in public storage.
> - Put it into Docker images.
> - Put it into source repositories.

---

## 22. Certificate and Private Key Match

Before installing a new certificate, verify that the certificate and private key belong together.

**For RSA:**

```bash
openssl x509 -noout -modulus -in CRT.txt | openssl md5
```

and:

```bash
openssl rsa -noout -modulus -in PRIVATEKEY.txt | openssl md5
```

The hashes **must match**.

If they do not match:

```text
STOP
```

Do not deploy the pair. Contact IT or the certificate issuer.

---

## 23. Jenkins Localhost Hardening

| Before | After |
|---|---|
| `*:8080` | `127.0.0.1:8080` |

The setting was implemented through a systemd drop-in:

```text
/etc/systemd/system/jenkins.service.d/override.conf
```

**Contents:**

```ini
[Service]
Environment="JENKINS_LISTEN_ADDRESS=127.0.0.1"
```

---

## 24. Why a Systemd Override Was Used

The main Jenkins unit says:

```text
/usr/lib/systemd/system/jenkins.service
```

and explicitly warns **not** to modify the package-managed unit directly.

Instead, systemd drop-ins should be used:

```text
/etc/systemd/system/jenkins.service.d/
```

This keeps custom configuration separate from vendor/package files.

**Advantages:**

- Package upgrades are safer.
- Local customizations are easy to identify.
- Configuration is reversible.
- systemd's intended override mechanism is used.

---

## 25. How Jenkins Reads the Listen Address

The Jenkins launcher at:

```text
/usr/bin/jenkins
```

was inspected and confirmed to translate:

```text
JENKINS_LISTEN_ADDRESS=127.0.0.1
```

into an argument equivalent to:

```text
--httpListenAddress=127.0.0.1
```

The active Jenkins command therefore effectively becomes:

```text
--httpPort=8080
--httpListenAddress=127.0.0.1
```

This is why the binding changed from all interfaces to localhost.

---

## 26. Applying the Jenkins systemd Override

After creating the override:

```bash
sudo systemctl daemon-reload
sudo systemctl restart jenkins
```

**Verify:**

```bash
ss -tulpn | grep 8080
```

**Expected:**

```text
127.0.0.1:8080
```

**not:**

```text
*:8080
```

---

## 27. Why Jenkins Does Not Need HTTPS Internally

The final path is:

```text
Client
  |
  | HTTPS
  v
Nginx :443
  |
  | HTTP over loopback
  v
127.0.0.1:8080
  |
  v
Jenkins
```

The HTTP segment is only between Nginx and Jenkins on the same server over the loopback interface.

There is no network hop between an external client and `127.0.0.1`.

This is the basis for TLS termination at Nginx.

---

## 28. TLS Termination

TLS is terminated at Nginx.

**Meaning:**

```text
Client
  |
  | Encrypted HTTPS
  v
Nginx
  |
  | Decrypted HTTP
  v
Jenkins
```

Nginx:

1. Receives the TLS connection.
2. Presents the certificate.
3. Validates the TLS handshake.
4. Decrypts the request.
5. Proxies the request to Jenkins.

Jenkins does not need to manage the public certificate in this design.

---

## 29. Jenkins URL

The externally visible Jenkins URL must be the hostname:

```text
https://jenkins.tech-bridge.biz/
```

**In Jenkins:**

```text
Manage Jenkins
  -> System
  -> Jenkins URL
```

**Set:**

```text
https://jenkins.tech-bridge.biz/
```

**Do not use:**

```text
http://192.168.0.17:8080/
```

**Do not use:**

```text
https://192.168.0.17/
```

The hostname is part of the certificate identity and the public application identity.

---

## 30. Jenkins Agents

The agents originally used:

```text
https://192.168.0.17
```

The final endpoint is:

```text
https://jenkins.tech-bridge.biz
```

**Why?**

The certificate contains:

```text
*.tech-bridge.biz
```

but not:

```text
192.168.0.17
```

The agent's Java runtime validates the hostname against the certificate SAN.

Therefore:

```text
jenkins.tech-bridge.biz
        |
        | matches
        v
*.tech-bridge.biz
```

while:

```text
192.168.0.17
        |
        | does not match
        v
*.tech-bridge.biz
```

---

## 31. Agent Systemd Configuration

The agents are managed using systemd services.

The exact service name may differ between environments. A typical structure is:

```text
/etc/systemd/system/jenkins-agent.service
```

or similar.

The important setting is that the agent URL should point to:

```text
https://jenkins.tech-bridge.biz
```

After modifying an agent unit:

```bash
sudo systemctl daemon-reload
sudo systemctl restart <agent-service>
```

**Check:**

```bash
systemctl status <agent-service> --no-pager
```

**Logs:**

```bash
journalctl -u <agent-service> -f
```

---

## 32. Certificate Validation From an Agent

Use:

```bash
openssl s_client \
  -connect jenkins.tech-bridge.biz:443 \
  -servername jenkins.tech-bridge.biz </dev/null 2>/dev/null \
  | openssl x509 -noout -subject -issuer -dates
```

**For SAN:**

```bash
openssl s_client \
  -connect jenkins.tech-bridge.biz:443 \
  -servername jenkins.tech-bridge.biz </dev/null 2>/dev/null \
  | openssl x509 -noout -ext subjectAltName
```

**Expected SAN:**

```text
DNS:*.tech-bridge.biz
DNS:tech-bridge.biz
```

---

## 33. Useful HTTPS Tests

### Test HTTPS response

```bash
curl -Ik https://jenkins.tech-bridge.biz
```

A Jenkins authentication page or a status such as `403` can still indicate that Nginx successfully reached Jenkins.

Look for Jenkins headers:

```text
X-Jenkins
X-Hudson
Set-Cookie
```

### Test HTTP redirect

```bash
curl -I http://jenkins.tech-bridge.biz
```

**Expected:**

```text
301
Location: https://jenkins.tech-bridge.biz/...
```

### Test TLS certificate

```bash
openssl s_client \
  -connect jenkins.tech-bridge.biz:443 \
  -servername jenkins.tech-bridge.biz
```

---

## 34. Why `curl -k` Was Used During Testing

**Example:**

```bash
curl -k https://192.168.0.17
```

The `-k` option means:

```text
Ignore certificate validation errors
```

It is useful when testing a self-signed certificate.

It should **not** be used as the normal production validation method.

For the final trusted certificate, use:

```bash
curl -I https://jenkins.tech-bridge.biz
```

without `-k`.

---

## 35. Browser Certificate Warning vs Mixed Content Warning

There are two different problems.

### Certificate warning

Usually means:

```text
Certificate is not trusted
```

or:

```text
Hostname does not match certificate
```

### Mixed content warning

Means:

```text
Page loaded via HTTPS,
but some resource was requested over HTTP.
```

The final deployment should use the HTTPS hostname consistently so generated URLs are HTTPS.

---

## 36. Nginx Configuration Validation

> ⚠️ **Never reload Nginx without testing the configuration first.**

**Use:**

```bash
sudo nginx -t
```

**Expected:**

```text
syntax is ok
test is successful
```

Only after the test succeeds:

```bash
sudo systemctl reload nginx
```

---

## 37. Why Reload Instead of Restart

Certificate renewal and Nginx configuration changes generally require Nginx to reload its configuration.

**Preferred:**

```bash
sudo systemctl reload nginx
```

This is preferable to restarting Nginx because it avoids unnecessary interruption.

**Typical certificate renewal flow:**

```text
Replace certificate
       |
       v
nginx -t
       |
       v
systemctl reload nginx
```

Jenkins does **not** need to restart just because the Nginx certificate changed.

---

## 38. Certificate Renewal Process

When IT provides a renewed certificate, you receive:

```text
CABUNDLE
CRT
PRIVATE KEY
```

Place them temporarily in:

```text
/root/ssl-certificates/
```

**For example:**

```text
/root/ssl-certificates/CABUNDLE.txt
/root/ssl-certificates/CRT.txt
/root/ssl-certificates/PRIVATEKEY.txt
```

---

## 39. Step-by-Step Certificate Renewal

### Step 1: Verify certificate dates

```bash
openssl x509 \
  -in /root/ssl-certificates/CRT.txt \
  -noout -dates
```

Confirm that:

```text
notBefore
notAfter
```

are correct.

### Step 2: Verify SAN

```bash
openssl x509 \
  -in /root/ssl-certificates/CRT.txt \
  -text -noout \
  | grep -A5 "Subject Alternative Name"
```

**Expected for the current wildcard certificate:**

```text
DNS:*.tech-bridge.biz
DNS:tech-bridge.biz
```

### Step 3: Verify issuer

```bash
openssl x509 \
  -in /root/ssl-certificates/CRT.txt \
  -noout -issuer -subject
```

### Step 4: Verify key/certificate match

```bash
openssl x509 -noout -modulus \
  -in /root/ssl-certificates/CRT.txt \
  | openssl md5
```

```bash
openssl rsa -noout -modulus \
  -in /root/ssl-certificates/PRIVATEKEY.txt \
  | openssl md5
```

**Hashes must match.**

---

## 40. Step 5: Build the Certificate Chain

**Use:**

```bash
cd /root/ssl-certificates

(
    cat CRT.txt
    echo
    cat CABUNDLE.txt
) > fullchain.pem
```

**Verify:**

```bash
grep -n "BEGIN CERTIFICATE\|END CERTIFICATE" fullchain.pem
```

---

## 41. Step 6: Back Up Existing Production Certificates

**Create a dated backup:**

```bash
sudo mkdir -p /etc/nginx/ssl/backup-$(date +%F)
```

**Copy existing certificate files:**

```bash
sudo cp /etc/nginx/ssl/fullchain.pem \
  /etc/nginx/ssl/backup-$(date +%F)/

sudo cp /etc/nginx/ssl/server.key \
  /etc/nginx/ssl/backup-$(date +%F)/
```

**Also back up the Nginx configuration before major changes:**

```bash
sudo cp /etc/nginx/sites-available/jenkins \
  /etc/nginx/sites-available/jenkins.bak
```

---

## 42. Step 7: Install the New Runtime Certificate

```bash
sudo cp /root/ssl-certificates/fullchain.pem \
  /etc/nginx/ssl/fullchain.pem
```

```bash
sudo cp /root/ssl-certificates/PRIVATEKEY.txt \
  /etc/nginx/ssl/server.key
```

**Secure the key:**

```bash
sudo chmod 600 /etc/nginx/ssl/server.key
sudo chown root:root /etc/nginx/ssl/server.key
```

---

## 43. Step 8: Validate Nginx

```bash
sudo nginx -t
```

> ⚠️ **Do not reload if this fails.**

**Common certificate errors include:**

```text
PEM_read_bio_X509_AUX() failed
```

or:

```text
bad end line
```

These often indicate malformed PEM formatting or an incorrect certificate file.

---

## 44. Step 9: Reload Nginx

```bash
sudo systemctl reload nginx
```

No Jenkins restart is required for a normal Nginx certificate renewal.

---

## 45. Step 10: Verify the Live Certificate

**From the Jenkins host or an agent:**

```bash
openssl s_client \
  -connect jenkins.tech-bridge.biz:443 \
  -servername jenkins.tech-bridge.biz </dev/null 2>/dev/null \
  | openssl x509 -noout -subject -issuer -dates
```

**Verify SAN:**

```bash
openssl s_client \
  -connect jenkins.tech-bridge.biz:443 \
  -servername jenkins.tech-bridge.biz </dev/null 2>/dev/null \
  | openssl x509 -noout -ext subjectAltName
```

---

## 46. Why the Jenkins Service Does Not Need Restart During Certificate Renewal

The certificate is owned by Nginx, not Jenkins.

**Certificate flow:**

```text
Client
  |
  | TLS
  v
Nginx
  |
  | HTTP
  v
Jenkins
```

**Therefore:**

```text
New certificate
      |
      v
Nginx reload
```

Jenkins continues running.

---

## 47. Important Verification: Who Owns Each Port?

**Use:**

```bash
ss -tulpn
```

**Current desired state:**

```text
443  -> nginx
80   -> nginx
8080 -> Jenkins on 127.0.0.1 only
```

**Example:**

```text
0.0.0.0:443
127.0.0.1:8080
```

The exact display can vary, especially with IPv4-mapped IPv6 sockets, but the key point is that the Jenkins listener must be bound to **localhost** and not `0.0.0.0`.

---

## 48. Final Port Architecture

```text
Port 80
   |
   +---- Nginx
   |
   +---- Redirect to HTTPS

Port 443
   |
   +---- Nginx
   |
   +---- TLS

Port 8080
   |
   +---- Jenkins
   |
   +---- localhost only
```

---

## 49. Security Boundary

The security boundary is:

```text
External/Internal Client
          |
          | HTTPS
          v
        Nginx
          |
          | localhost only
          v
        Jenkins
```

This means clients **cannot** directly connect to Jenkins through the network using port 8080.

---

## 50. Recommended File Inventory

### Nginx

```text
/etc/nginx/sites-available/jenkins
/etc/nginx/sites-enabled/jenkins
```

### Nginx runtime TLS files

```text
/etc/nginx/ssl/fullchain.pem
/etc/nginx/ssl/server.key
```

### Certificate working/staging directory

```text
/root/ssl-certificates/
```

**Typical contents:**

```text
CABUNDLE.txt
CRT.txt
PRIVATEKEY.txt
fullchain.pem
```

### Jenkins systemd override

```text
/etc/systemd/system/jenkins.service.d/override.conf
```

**Content:**

```ini
[Service]
Environment="JENKINS_LISTEN_ADDRESS=127.0.0.1"
```

---

## 51. Important Principle: Source vs Runtime Configuration

Keep these concepts separate.

```text
/root/ssl-certificates/
        |
        | New certificates received from IT
        | Verification
        | Chain creation
        v
/etc/nginx/ssl/
        |
        | Production runtime files
        v
Nginx
```

This makes troubleshooting easier.

When a certificate expires, verify both:

```text
What IT supplied
```

and:

```text
What Nginx is actually using
```

---

## 52. Useful Commands Cheat Sheet

### Nginx

```bash
nginx -t
systemctl status nginx --no-pager
systemctl reload nginx
ss -tulpn | grep -E ':80|:443'
```

### Jenkins

```bash
systemctl status jenkins --no-pager
systemctl restart jenkins
systemctl cat jenkins
ss -tulpn | grep 8080
```

### DNS

```bash
nslookup jenkins.tech-bridge.biz
getent hosts jenkins.tech-bridge.biz
```

### HTTP

```bash
curl -I http://jenkins.tech-bridge.biz
curl -I https://jenkins.tech-bridge.biz
curl -v https://jenkins.tech-bridge.biz
```

### TLS

```bash
openssl s_client \
  -connect jenkins.tech-bridge.biz:443 \
  -servername jenkins.tech-bridge.biz
```

### Certificate information

```bash
openssl x509 -in CRT.txt -noout -dates
```

```bash
openssl x509 -in CRT.txt -noout -issuer -subject
```

```bash
openssl x509 -in CRT.txt -text -noout \
  | grep -A10 "Subject Alternative Name"
```

---

## 53. Troubleshooting Methodology

When Jenkins HTTPS breaks, troubleshoot from the bottom up.

### Layer 1: DNS

```bash
nslookup jenkins.tech-bridge.biz
```

**Question:**

> Does the hostname resolve to the expected IP?

### Layer 2: Network / Port

```bash
ss -tulpn
```

**Question:**

> Is Nginx listening on 443?

### Layer 3: TLS

```bash
openssl s_client ...
```

**Questions:**

- Is the certificate presented?
- Is the certificate expired?
- Is the issuer correct?
- Are SANs correct?
- Is the certificate chain correct?

### Layer 4: HTTP

```bash
curl -I https://jenkins.tech-bridge.biz
```

**Question:**

> Is Nginx successfully reaching Jenkins?

### Layer 5: Jenkins

```bash
systemctl status jenkins
journalctl -u jenkins
```

**Question:**

> Is the backend application healthy?

---

## 54. Common Failure Scenarios

### DNS failure

**Symptom:**

```text
NXDOMAIN
```

**Fix:** Create the internal DNS record.

### Wrong DNS IP

**Symptom:** Hostname resolves, but points to the wrong server.

**Fix:** Correct the A record.

### Nginx configuration error

**Symptom:** `nginx -t` fails.

**Fix:** Read the exact configuration error before reloading.

### Certificate expired

**Symptom:** Browser or client reports expired certificate.

**Fix:** Renew certificate and reload Nginx.

### Certificate/key mismatch

**Symptom:** Nginx refuses to start/load certificate.

**Fix:** Verify modulus hashes.

### Malformed PEM

**Symptom:**

```text
bad end line
PEM_read_bio_X509_AUX() failed
```

**Fix:** Verify certificate boundaries and rebuild the chain with correct newlines.

### Wrong certificate served

**Symptom:** New certificate was installed but clients still see the old one.

**Check:**

```bash
grep "ssl_certificate" /etc/nginx/sites-available/jenkins
```

Then verify the actual live endpoint:

```bash
openssl s_client ...
```

**Remember:**

```text
/root/ssl-certificates/
```

is not necessarily the runtime path.

### Hostname mismatch

**Symptom:** Java reports hostname/SAN-related errors.

**Check:**

```bash
openssl s_client ...
```

and:

```bash
openssl x509 ...
```

The URL must match a SAN covered by the certificate.

### Jenkins unreachable behind Nginx

**Check:**

```bash
curl http://127.0.0.1:8080/login
```

If this fails, the Jenkins backend is the problem.

If this works but HTTPS fails, investigate Nginx/TLS.

---

## 55. How to Add a New Internal Tool Using Nginx

The Jenkins setup is a reusable pattern.

Suppose SonarQube is introduced.

**Assume:**

```text
SonarQube backend:
127.0.0.1:9000

Hostname:
sonarqube.tech-bridge.biz
```

The intended architecture becomes:

```text
https://sonarqube.tech-bridge.biz
                |
                v
            Nginx :443
                |
                v
          127.0.0.1:9000
                |
                v
            SonarQube
```

---

## 56. Step-by-Step Pattern for SonarQube

### Step 1: Confirm SonarQube backend port

```bash
ss -tulpn | grep 9000
```

Ensure it is healthy locally.

**Test:**

```bash
curl -I http://127.0.0.1:9000
```

### Step 2: Create internal DNS

Create:

```text
sonarqube.tech-bridge.biz -> <SonarQube server IP>
```

If SonarQube is on the same server:

```text
sonarqube.tech-bridge.biz -> 192.168.0.17
```

Otherwise point it to the actual SonarQube host.

### Step 3: Confirm certificate coverage

Because the current certificate is:

```text
*.tech-bridge.biz
```

the hostname:

```text
sonarqube.tech-bridge.biz
```

is covered.

**You can verify:**

```bash
openssl x509 \
  -in /etc/nginx/ssl/fullchain.pem \
  -text -noout \
  | grep -A5 "Subject Alternative Name"
```

### Step 4: Create an Nginx site

**Example:**

```text
/etc/nginx/sites-available/sonarqube
```

**Configuration:**

```nginx
server {
    listen 80;
    server_name sonarqube.tech-bridge.biz;

    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl http2;
    server_name sonarqube.tech-bridge.biz;

    ssl_certificate     /etc/nginx/ssl/fullchain.pem;
    ssl_certificate_key /etc/nginx/ssl/server.key;

    location / {
        proxy_pass http://127.0.0.1:9000;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;

        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}
```

---

## 57. Enable the New Tool

```bash
sudo ln -s \
  /etc/nginx/sites-available/sonarqube \
  /etc/nginx/sites-enabled/sonarqube
```

**Then:**

```bash
sudo nginx -t
```

**Only if successful:**

```bash
sudo systemctl reload nginx
```

---

## 58. Why Multiple Applications Can Share Port 443

This is a fundamental Nginx concept.

Different applications can all use:

```text
443
```

because Nginx uses the requested hostname.

**Example:**

```text
https://jenkins.tech-bridge.biz
        |
        +--> Jenkins

https://sonarqube.tech-bridge.biz
        |
        +--> SonarQube

https://nexus.tech-bridge.biz
        |
        +--> Nexus
```

All reach:

```text
Nginx :443
```

Nginx decides which backend receives the request based on:

```nginx
server_name
```

This is called **host-based routing**.

---

## 59. One Nginx, Many Internal Tools

A future architecture can look like:

```text
                         Internal Users
                              |
                              v
                         Nginx :443
                              |
          +-------------------+-------------------+
          |                   |                   |
          v                   v                   v
 jenkins.tech-bridge.biz  sonarqube.tech-bridge.biz
          |                   |
          v                   v
   127.0.0.1:8080       127.0.0.1:9000
          |                   |
       Jenkins            SonarQube
```

Additional tools can follow exactly the same model.

---

## 60. Important Rule for New Tools

Before adding a new Nginx entry, determine:

1. Tool hostname
2. Tool backend IP
3. Tool backend port
4. Whether the tool needs WebSocket/upgrade support
5. Whether the certificate covers the hostname
6. Whether internal DNS resolves correctly
7. Whether the application knows its external HTTPS URL
8. Whether the application requires special proxy headers or path handling

> ⚠️ **Do not blindly copy a Jenkins configuration into every tool.** The basic pattern is reusable, but some applications need additional directives.

---

## 61. Suggested Standard for Internal Hostnames

Use a consistent naming model:

```text
jenkins.tech-bridge.biz
sonarqube.tech-bridge.biz
nexus.tech-bridge.biz
grafana.tech-bridge.biz
harbor.tech-bridge.biz
```

**Advantages:**

- Easy to remember
- Wildcard certificate compatible
- Easy to route through Nginx
- Easier automation
- Consistent enterprise naming

---

## 62. Certificate Reuse

Because the current certificate is a wildcard:

```text
*.tech-bridge.biz
```

the same certificate can potentially be used for:

```text
jenkins.tech-bridge.biz
sonarqube.tech-bridge.biz
nexus.tech-bridge.biz
grafana.tech-bridge.biz
```

This reduces certificate management overhead.

> ⚠️ However, carefully control access to the wildcard private key because compromise of the private key can affect **all** names covered by the certificate.

---

## 63. Future Certificate Strategy

**Current model:**

```text
IT provides renewed certificate
        |
        v
/root/ssl-certificates/
        |
        v
/etc/nginx/ssl/
        |
        v
Nginx reload
```

**Future improvements could include:**

- Automated certificate deployment
- Secret management
- Automated expiry checks
- Internal PKI integration
- Central certificate management
- Controlled CI/CD-based certificate rollout

> ⚠️ Do not automate private-key distribution casually; certificates and keys should be handled as secrets.

---

## 64. Operational Checklist

### Before changing certificates

```text
[ ] Confirm the certificate expiry
[ ] Confirm SAN entries
[ ] Confirm issuer
[ ] Confirm private key matches certificate
[ ] Back up current certificate/key
[ ] Back up Nginx config
[ ] Build full chain
```

### During deployment

```text
[ ] Copy certificate to /etc/nginx/ssl/
[ ] Copy private key to /etc/nginx/ssl/
[ ] Set secure key permissions
[ ] Run nginx -t
[ ] Reload Nginx
```

### After deployment

```text
[ ] Verify live certificate
[ ] Verify expiry date
[ ] Verify SAN
[ ] Verify issuer
[ ] Test curl
[ ] Test browser
[ ] Test Jenkins agent
[ ] Check Nginx logs
```

---

## 65. What Should Never Be Changed Casually

### Do not edit the package-managed Jenkins systemd unit

**Avoid changing:**

```text
/usr/lib/systemd/system/jenkins.service
```

**Use:**

```text
/etc/systemd/system/jenkins.service.d/override.conf
```

instead.

### Do not modify Nginx and reload without testing

**Always:**

```bash
nginx -t
```

**then:**

```bash
systemctl reload nginx
```

### Do not replace certificates without backups

Always keep a rollback copy.

### Do not expose the Jenkins private key

**Treat:**

```text
server.key
```

as a secret.

---

## 66. Rollback Procedure

If a new certificate causes Nginx failure:

1. Restore the previous certificate and key.
2. Verify the Nginx configuration.
3. Reload Nginx.
4. Validate the previous endpoint.

**Example:**

```bash
sudo cp /etc/nginx/ssl/backup-YYYY-MM-DD/fullchain.pem \
  /etc/nginx/ssl/fullchain.pem
```

```bash
sudo cp /etc/nginx/ssl/backup-YYYY-MM-DD/server.key \
  /etc/nginx/ssl/server.key
```

**Then:**

```bash
sudo nginx -t
sudo systemctl reload nginx
```

> ⚠️ Do not improvise under pressure; always keep a rollback copy.

---

## 67. Final Architecture Diagram

```text
                                  Internal Network
                                         |
                                         |
                         +---------------+---------------+
                         |                               |
                         v                               v
              Internal DNS Resolution            Jenkins Agents
                         |                               |
                         |                               |
                         +---------------+---------------+
                                         |
                                         v
                           jenkins.tech-bridge.biz
                                         |
                                         |
                                    HTTPS :443
                                         |
                                         v
                              +-------------------+
                              |       Nginx       |
                              |  Reverse Proxy    |
                              | TLS Termination   |
                              +-------------------+
                                         |
                                         |
                                  HTTP localhost
                                         |
                                         v
                                127.0.0.1:8080
                                         |
                                         v
                                  +-------------+
                                  |   Jenkins   |
                                  +-------------+
```

---

## 68. Final Configuration Map

| Item | Current Configuration |
|---|---|
| Jenkins server | `192.168.0.17` |
| Jenkins backend | `127.0.0.1:8080` |
| External hostname | `jenkins.tech-bridge.biz` |
| External protocol | HTTPS |
| External port | `443` |
| HTTP port | `80` |
| HTTP behavior | 301 redirect to HTTPS |
| Reverse proxy | Nginx |
| Certificate type | Trusted wildcard certificate |
| Certificate SAN | `*.tech-bridge.biz`, `tech-bridge.biz` |
| Runtime certificate | `/etc/nginx/ssl/fullchain.pem` |
| Runtime private key | `/etc/nginx/ssl/server.key` |
| IT certificate staging directory | `/root/ssl-certificates/` |
| Nginx config | `/etc/nginx/sites-available/jenkins` |
| Enabled Nginx site | `/etc/nginx/sites-enabled/jenkins` |
| Jenkins systemd override | `/etc/systemd/system/jenkins.service.d/override.conf` |
| Jenkins agent URL | `https://jenkins.tech-bridge.biz` |

---

## 69. Mental Model to Remember

When debugging this environment, think in this order:

```text
DNS
  ↓
Network / Port
  ↓
TLS Certificate
  ↓
Nginx
  ↓
Reverse Proxy
  ↓
Jenkins
  ↓
Agent/Application
```

Or as a request:

```text
https://jenkins.tech-bridge.biz
            |
            v
        DNS lookup
            |
            v
      192.168.0.17
            |
            v
        Nginx :443
            |
            | TLS termination
            v
    127.0.0.1:8080
            |
            v
         Jenkins
```

This model is reusable far beyond Jenkins.

---

## 70. One-Line Interview Summary

> Jenkins is deployed behind an Nginx reverse proxy that terminates TLS on port 443, redirects HTTP to HTTPS, proxies traffic to Jenkins on localhost port 8080, and uses an internally resolvable hostname covered by a trusted wildcard certificate so Java 21 agents can connect securely with proper SAN validation.

---

## 71. Short Operational Summary

To remember the whole implementation:

```text
1. Install Nginx
2. Create internal DNS hostname
3. Obtain trusted certificate
4. Build fullchain.pem
5. Configure Nginx on 443
6. Redirect 80 -> 443
7. Proxy 443 -> 127.0.0.1:8080
8. Bind Jenkins to localhost
9. Set Jenkins URL to the hostname
10. Update Java 21 agents to the hostname
11. Validate DNS + TLS + HTTP + Jenkins
12. Renew certificates by replacing Nginx runtime files and reloading Nginx
```

---

## 72. Production Philosophy

The key lesson is not simply:

> "How to enable HTTPS on Jenkins."

The bigger DevOps lesson is:

> **Design the access path so that the application is not directly exposed, security is centralized at the edge, identity is defined through DNS and certificates, application traffic is isolated to localhost where appropriate, and the same architecture can be reused for additional internal services.**

That is the architectural pattern to carry forward to Jenkins, SonarQube, Nexus, Grafana, Harbor, internal APIs, and future services.

---

*End of document.*
