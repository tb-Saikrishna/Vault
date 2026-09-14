# Jenkins SSL Certificate Renewal – Simple Steps

### 1. Put the new files in the staging directory

```text
/root/ssl-certificates/
```

You should have:

```text
CRT.txt
CABUNDLE.txt
PRIVATEKEY.txt
```

---

### 2. Check the new certificate

Check expiry:

```bash
openssl x509 -in /root/ssl-certificates/CRT.txt -noout -dates
```

Check SAN:

```bash
openssl x509 -in /root/ssl-certificates/CRT.txt -text -noout | grep -A5 "Subject Alternative Name"
```

You should see your hostname/wildcard, such as:

```text
DNS:*.tech-bridge.biz
DNS:tech-bridge.biz
```

---

### 3. Verify the private key matches the certificate

```bash
openssl x509 -noout -modulus \
-in /root/ssl-certificates/CRT.txt | openssl md5
```

```bash
openssl rsa -noout -modulus \
-in /root/ssl-certificates/PRIVATEKEY.txt | openssl md5
```

The two hashes **must be the same**.

---

### 4. Create the full certificate chain

```bash
cd /root/ssl-certificates

(
    cat CRT.txt
    echo
    cat CABUNDLE.txt
) > fullchain.pem
```

Verify:

```bash
grep -n "BEGIN CERTIFICATE\|END CERTIFICATE" fullchain.pem
```

Make sure `BEGIN` and `END` are on separate lines.

---

### 5. Backup the currently active certificate

```bash
sudo mkdir -p /etc/nginx/ssl/backup-$(date +%F)

sudo cp /etc/nginx/ssl/fullchain.pem /etc/nginx/ssl/backup-$(date +%F)/
sudo cp /etc/nginx/ssl/server.key /etc/nginx/ssl/backup-$(date +%F)/
```

---

### 6. Copy the new certificate to Nginx

```bash
sudo cp /root/ssl-certificates/fullchain.pem /etc/nginx/ssl/fullchain.pem
```

```bash
sudo cp /root/ssl-certificates/PRIVATEKEY.txt /etc/nginx/ssl/server.key
```

Secure the private key:

```bash
sudo chmod 600 /etc/nginx/ssl/server.key
sudo chown root:root /etc/nginx/ssl/server.key
```

---

### 7. Verify Nginx is pointing to the correct files

```bash
grep "ssl_certificate" /etc/nginx/sites-available/jenkins
```

It should show:

```nginx
ssl_certificate /etc/nginx/ssl/fullchain.pem;
ssl_certificate_key /etc/nginx/ssl/server.key;
```

---

### 8. Test Nginx configuration

**Always do this before reloading.**

```bash
sudo nginx -t
```

You want:

```text
syntax is ok
test is successful
```

If it fails, **do not reload**.

---

### 9. Reload Nginx

```bash
sudo systemctl reload nginx
```

### Important

**Do NOT restart Jenkins.**

The certificate is being handled by Nginx, so:

```text
New Certificate
      ↓
Nginx reload
      ↓
New HTTPS certificate active
```

Jenkins keeps running.

---

### 10. Verify the live certificate

```bash
openssl s_client \
-connect jenkins.tech-bridge.biz:443 \
-servername jenkins.tech-bridge.biz </dev/null 2>/dev/null \
| openssl x509 -noout -subject -issuer -dates
```

Check that the `notAfter` date is the new expiry date.

Also verify SAN:

```bash
openssl s_client \
-connect jenkins.tech-bridge.biz:443 \
-servername jenkins.tech-bridge.biz </dev/null 2>/dev/null \
| openssl x509 -noout -ext subjectAltName
```

---

### 11. Test Jenkins

```bash
curl -I https://jenkins.tech-bridge.biz
```

Then open:

```text
https://jenkins.tech-bridge.biz
```

Also verify one Jenkins agent connects successfully.

---

# Your 30-Second Memory Version

Whenever IT gives you:

```text
CRT
CABUNDLE
PRIVATE KEY
```

remember:

```text
1. Verify CRT
2. Verify SAN
3. Verify KEY matches CRT
4. CRT + CABUNDLE → fullchain.pem
5. Backup old files
6. Copy fullchain.pem + private key to /etc/nginx/ssl/
7. nginx -t
8. systemctl reload nginx
9. Verify live certificate
10. Test Jenkins + one agent
```

And the most important rule:

> **Certificate renewal = Nginx reload, not Jenkins restart.**
