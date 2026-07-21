# PM2 Removal from Docker Containers

## Objective
Standardize all Node.js Docker containers by removing **PM2** and allowing **Docker** to be the single process manager.

---

## Why We Changed

### Previous Runtime

```text
Docker
└── bash (PID 1)
    ├── redis-server --daemonize yes
    ├── PM2
    │   └── Node.js
    └── tail -f /dev/null
```

Problems:
- Two process managers (Docker + PM2)
- `bash` became PID 1 instead of Node.js
- `tail -f /dev/null` artificially kept containers alive
- `docker ps` showed containers as running even when the application had crashed
- Engineers had to enter containers and inspect PM2
- Larger images and unnecessary dependencies

---

## New Runtime

```text
Docker
└── Node.js (PID 1)
```

Docker now handles:
- Process lifecycle
- Restart policy
- Logging
- Signal forwarding
- Health visibility

---

## Migration Steps

### 1. Dockerfile

Old:

```dockerfile
CMD ["bash","-c","redis-server --daemonize yes && pm2 start --name tbsoar-backend \"node dist/bundle.js\" && tail -f /dev/null"]
```

New:

```dockerfile
CMD ["node","dist/bundle.js"]
```

---

### 2. Remove PM2

Remove:
- PM2 installation
- PM2 runtime
- ecosystem.config.js
- PM2 scripts

---

### 3. Configure Docker Logging

```yaml
logging:
  driver: json-file
  options:
    max-size: "500m"
    max-file: "5"
```

---

### 4. Configure Restart Policy

```yaml
restart: unless-stopped
```

(or `always` if appropriate)

---

## Benefits

- Simpler runtime architecture
- Docker is the single source of truth
- Accurate container health
- Native Docker logging
- Native restart handling
- Smaller images
- Easier troubleshooting
- Better signal handling
- Follows Docker best practices

---

## Validation Checklist

- Container starts successfully
- `docker logs` shows application logs
- No PM2 installed
- No `tail -f /dev/null`
- Node.js is PID 1
- Restart policy works after crash
- Graceful SIGTERM handling verified

---

## Key Takeaway

**Containers should run the application—not a shell wrapper.**

Avoid:

```text
Docker → bash → PM2 → Node.js
```

Prefer:

```text
Docker → Node.js
```

Docker already provides the capabilities that PM2 was previously used for, making PM2 inside a single-process container unnecessary.
