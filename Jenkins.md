# Jenkins

This document serves as my personal knowledge base for everything I learn and contribute about jenkins to at my organization. I will continuously update it as I encounter new tools, concepts, and challenges.

## Table of Contents

1. [Basics (Theory Only)](#1-basics-theory-only)
2. [Examples & Stuff (Basic Codes)](#2-examples--stuff-basic-codes)
3. [Advanced (Theory Only)](#3-advanced-theory-only)
4. [Examples & Stuff (Advanced)](#4-examples--stuff-advanced)
5. [Troubleshooting (Theory & Solutions)](#5-troubleshooting-theory--solutions)

---

## 1. Basics (Theory Only)

*Core concepts I have learned from scratch.*

### Jenkins Fundamentals
- **Agents**: A node (machine/container) that executes Jenkins pipelines. Two types:
    - **Master/Controller**: Manages scheduling, job tracking, and UI.
    - **Node/Agent**: Does the actual build/execution work.
- **Nodes**: Any machine that is part of the Jenkins environment (including the master).
- **Credential Store**: Built-in secure vault to store usernames/passwords, SSH keys, secrets, and tokens. Never hardcode credentials in a Jenkinsfile.
- **Multi-branch Pipelines**: A pipeline that automatically discovers, manages, and executes branches (e.g., `main`, `develop`, `feature/*`) based on a Jenkinsfile in each branch.
- **Jenkinsfile**: A text file (written in Groovy DSL or Declarative syntax) that defines your entire CI/CD pipeline as code.

---

## 2. Examples & Stuff (Basic Codes)

*Simple, working code snippets from my early learning.*

### Basic Declarative Pipeline (Single Agent)
```groovy
pipeline {
    agent any  // Runs on any available agent
    stages {
        stage('Build') {
            steps {
                echo 'Building the application...'
            }
        }
        stage('Test') {
            steps {
                echo 'Running tests...'
            }
        }
    }
}
```

### Adding a Credential (Password) in Jenkinsfile
*(Make sure credential ID 'my-password-id' exists in Jenkins > Credentials)*

```groovy
pipeline {
    agent any
    environment {
        MY_SECRET = credentials('my-password-id')
    }
    stages {
        stage('Use Secret') {
            steps {
                sh 'echo $MY_SECRET'  // Note: Jenkins masks output automatically
            }
        }
    }
}
```

> **Placeholder**: Add your own basic examples here (e.g., using `sh`, `bat`, `withEnv`, simple conditionals).

---

## 3. Advanced (Theory Only)

*Deeper concepts I have learned after hands-on work.*

### Advanced Jenkins Concepts
- **Pipeline as Code (Declarative vs Scripted)**:
    - *Declarative*: Simpler, structured (the `pipeline { ... }` block). Best for most use cases.
    - *Scripted*: More flexible, uses Groovy code (`node { ... }`). Use when declarative is too restrictive.
- **Shared Libraries**: Reusable Groovy code that can be imported across multiple pipelines. Reduces duplication.
- **Parallel Stages**: Run multiple stages concurrently for faster execution.
- **Lockable Resources**: Prevent concurrent access to shared resources (e.g., a database, a test device).
- **Agent Labels**: Tag agents (e.g., `label 'linux'`) to ensure specific jobs run on the right OS/architecture.

### Environment & Variable Nuances
- **Importance of `.env` files**: Externalizing environment variables (non-secret configs) makes pipelines portable and secure. Secrets still go to Credential Store.
- **String Interpolation Pitfalls**: Single quotes (`'`) in Groovy treat `$var` literally; double quotes (`"`) interpolate. Example: `sh 'echo $PATH'` vs `sh "echo $MY_VAR"`.

### Permission Issues (Theory)
- **User/Role-based Access**: Jenkins uses matrix-based security or project-based roles.
- **Agent Permissions**: The Jenkins process on an agent must have filesystem permissions to read/write workspace, install tools, etc.
- **Credential Permissions**: Credentials can be scoped to "Global" (any job) or "System" (restricted).

---

## 4. Examples & Stuff (Advanced)

*Complex code from real projects I have contributed to.*

### Multi-branch Pipeline with Parallel Stages
```groovy
pipeline {
    agent { label 'linux' }
    stages {
        stage('Build') {
            steps { sh 'make build' }
        }
        stage('Parallel Testing') {
            parallel {
                stage('Unit Tests') {
                    steps { sh 'make test-unit' }
                }
                stage('Integration Tests') {
                    steps { sh 'make test-integration' }
                }
            }
        }
        stage('Deploy Staging') {
            when { branch 'main' }
            steps { sh 'deploy-staging.sh' }
        }
    }
    post {
        failure { 
            echo "Pipeline failed. Check logs."
        }
    }
}
```

### Using Shared Library (pseudo-code)
*(Assuming you have `vars/myLib.groovy` in a shared library)*

```groovy
@Library('my-shared-library') _
pipeline {
    agent any
    stages {
        stage('Custom Step') {
            steps {
                myLib.sayHello('DevOps')
            }
        }
    }
}
```

### Handling String Interpolation Correctly
```groovy
pipeline {
    environment {
        APP_VERSION = '1.2.3'
    }
    stages {
        stage('Wrong') {
            steps {
                // This will print $APP_VERSION literally
                sh 'echo $APP_VERSION'
            }
        }
        stage('Correct') {
            steps {
                // This interpolates the variable
                sh "echo $APP_VERSION"
            }
        }
    }
}
```

> **Placeholder**: Add your actual Jenkinsfiles, complex `sh` scripts, credential usage patterns, or `.env` loading examples here.

---

## 5. Troubleshooting (Theory & Solutions)

*Errors I have faced and how I solved them.*

### Permission Issues

**Symptom**: `Permission denied` when trying to write to workspace, run a script, or access a socket.

**Theory**: The Jenkins agent process runs as a specific OS user (e.g., `jenkins`, `tomcat`). That user lacks permissions on the target file/directory.

**Solutions**:
1. **Fix filesystem permissions**:
   ```bash
   sudo chown -R jenkins:jenkins /path/to/workspace
   ```
2. **Add user to required group** (e.g., `docker`):
   ```bash
   sudo usermod -aG docker jenkins
   ```
3. **Use an agent with higher privileges** (careful!) or run specific steps with `sudo` (if configured).

### String Interpolation Issues

**Symptom**: A variable prints as `$MY_VAR` instead of its value, or pipeline fails to resolve environment variables.

**Theory**: 
- Groovy single quotes `'` = no interpolation (literal string).
- Double quotes `"` = interpolation.
- In `sh` steps, using `'` passes the raw string to shell; using `"` lets Jenkins evaluate first.

**Solution**: Use double quotes for interpolated strings, single quotes for static ones.

```groovy
// Wrong
sh 'echo $JOB_NAME'

// Correct
sh "echo ${env.JOB_NAME}"
```

### Importance of `.env` Files (Real Example)

**Problem**: Hardcoding config (e.g., `DB_URL=localhost`) in Jenkinsfile made it impossible to move between dev/staging/prod.

**Solution**:
1. Create `.env.dev`, `.env.prod` (not committed to Git!).
2. Load them dynamically in pipeline:
   ```groovy
   script {
       def envFile = readFile(".env.${env.BRANCH_NAME}")
       envFile.split('\n').each { line ->
           def (key, value) = line.split('=', 2)
           env[key] = value
       }
   }
   ```

### Credential Store Best Practices

**Problem**: Credentials accidentally printed in logs or committed to Git.

**Solution**:
- Always use `credentials('id')` or `usernamePassword()`.
- Never use `withCredentials` around `sh` that might echo output.
- Use `maskPasswords` plugin for extra safety.

### Agent/Node Connection Errors

**Symptom**: `Connection refused`, `Agent goes offline`, `Timeout waiting for agent`.

**Solutions**:
- Check network/firewall (master needs to reach agent on its port).
- Verify agent Java version matches master.
- Restart agent service (`systemctl restart jenkins-agent`).
- Increase timeout in pipeline: `options { timeout(time: 1, unit: 'HOURS') }`.

---

## Future Updates

*(I will add new topics as I learn them, following the same 5-section structure.)*

- [ ] Kubernetes integration with Jenkins
- [ ] Blue Ocean UI
- [ ] Pipeline performance optimizations
- [ ] More shared library examples
- [ ] Security hardening (Role-based access)
