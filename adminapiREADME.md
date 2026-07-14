# CI/CD Setup — admin-api on dev.wirepick.com

Documentation of the CI/CD pipeline set up for the `admin-api` (wpkadmin-api) Spring Boot application on a new dev server, using a GitHub Actions self-hosted runner deploying to Apache Tomcat.

## Overview

- **App:** `wpkadmin-api` — Spring Boot (WAR packaging) admin portal, Java 21, Maven
- **Repo:** `admin-java` (private)
- **Server:** dev.wirepick.com — AlmaLinux, Java 22, Apache Tomcat 11.0.2, Apache ActiveMQ Artemis
- **Deploy target:** WAR dropped into Tomcat's `webapps/` directory (Tomcat auto-deploys on file drop)
- **Access:** internal only (`localhost:8080`) — external traffic is routed by existing Nginx/Apache reverse proxies

## Architecture

```
GitHub push (master) OR manual trigger
      │
      ▼
Self-hosted GitHub Actions runner (systemd service, runs as `ghrunner`)
      │
      ├─ Checkout code
      ├─ Set up JDK 21
      ├─ mvn clean package (Maven build → WAR named from pom.xml version)
      └─ Deploy: stop Tomcat → copy WAR to webapps/ → start Tomcat
```

## Server-side setup

### 1. Dedicated service user for the runner
The GitHub Actions runner should never run as root. A dedicated user was created:

```bash
useradd -m -s /bin/bash ghrunner
```

### 2. Self-hosted runner
Registered against the repo and installed as a systemd service so it survives reboots and SSH disconnects:

```bash
su - ghrunner
mkdir actions-runner && cd actions-runner
curl -o actions-runner-linux-x64-<version>.tar.gz -L <download-url>
tar xzf ./actions-runner-linux-x64-<version>.tar.gz
./config.sh --url https://github.com/<org>/admin-java --token <runner-token>
```

Installed as a service (run as root):
```bash
cd /home/ghrunner/actions-runner
./svc.sh install ghrunner
./svc.sh start
```

### 3. Tomcat as a systemd service
Tomcat was previously only startable manually. Added a proper unit so it runs reliably:

```ini
[Unit]
Description=Apache Tomcat 11
After=network.target

[Service]
Type=forking
User=tomcat
Group=tomcat
Environment=JAVA_HOME=/opt/java
Environment=CATALINA_HOME=/opt/apache-tomcat-11.0.2
Environment=CATALINA_PID=/opt/apache-tomcat-11.0.2/temp/tomcat.pid
ExecStart=/opt/apache-tomcat-11.0.2/bin/startup.sh
ExecStop=/opt/apache-tomcat-11.0.2/bin/shutdown.sh
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

### 4. Scoped permissions for the runner
Rather than giving the runner user full root/sudo, it was granted only what it needs:

```bash
# /etc/sudoers.d/ghrunner-tomcat
ghrunner ALL=(root) NOPASSWD: /bin/systemctl restart tomcat, /bin/systemctl stop tomcat, /bin/systemctl start tomcat
```

```bash
usermod -aG tomcat ghrunner
chmod -R g+w /opt/apache-tomcat-11.0.2/webapps
```

> **Note:** these permissions can be reset by other manual operations on the server (this happened once during setup, briefly breaking deploys with a `Permission denied` error). If a deploy fails with `cp: cannot create regular file ... Permission denied`, re-run the `chmod -R g+w` command above.

## Workflow file

`.github/workflows/deploy.yml`:

```yaml
name: Build and Deploy admin-api

on:
  push:
    branches: [master]
  workflow_dispatch: # manual trigger, with branch selection built into GitHub's UI

jobs:
  deploy:
    runs-on: self-hosted
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'

      - name: Build
        run: mvn clean package -DskipTests

      - name: Deploy WAR to Tomcat
        run: |
          sudo systemctl stop tomcat
          cp target/*.war /opt/apache-tomcat-11.0.2/webapps/
          sudo systemctl start tomcat
```

> **Manual deploys:** with `workflow_dispatch` added, the **Actions** tab now shows a **Run workflow** button, with a branch dropdown built into GitHub's UI — no custom input needed. This allows redeploying any branch on demand, without requiring a commit/push to trigger it.

The WAR is copied **without renaming** — Maven names it directly from `pom.xml`'s `artifactId` + `<version>` (e.g. `wpkadmin-api-1.3.8.war`). Bumping the version in `pom.xml` automatically produces and deploys a correctly-named WAR on the next build; no workflow changes are needed when the app version changes.

## Issues encountered & resolutions

| Issue | Cause | Fix |
|---|---|---|
| `mvn: command not found` | Maven was never installed on the server | `dnf install -y maven` |
| `Could not find artifact org.beyondsystems:email-data-model:jar:1.2` | Version 1.2 isn't published on the internal Maven repo (`repo.wirepick.com`) | Located a copy already bundled in a previous manual deployment (`WEB-INF/lib/`) and installed it into the runner's local `.m2` repo via `mvn install:install-file` |
| Workflow stuck indefinitely at "Waiting for a runner" | A manually-run `./run.sh` test session conflicted with the systemd-managed runner session (`SessionConflictException`), silently killing the service | Killed the stray process, restarted the systemd service cleanly |
| **Other apps' WAR files deleted from `webapps/`** | Original deploy step used a blanket wildcard (`rm -f webapps/*.war`) instead of scoping to this app only, wiping unrelated apps' deployed WARs | Removed the wildcard line; deploy step now only ever touches this app's own files |
| `cp: ... Permission denied` on deploy | `webapps/` folder permissions were reset by an unrelated manual operation on the server | Restored with `chmod -R g+w /opt/apache-tomcat-11.0.2/webapps` |

## Known limitations

- `email-data-model:1.2` is only resolvable because it's cached in the runner's local `.m2` — it is **not** actually published on the internal Maven repository (known/acknowledged, not yet fixed upstream). If this server's Maven cache is ever cleared, or a second CI runner is added, this dependency will need to be properly published first.
- App is intentionally not exposed on port 8080 externally; internal-only by design, with Nginx/Apache handling reverse-proxied external access.
- Old WAR versions are never automatically cleaned from `webapps/` — they accumulate over time (matches existing team convention, but worth monitoring disk usage).

## Verifying a deployment

```bash
curl -I http://localhost:8080/wpkadmin-api/
```

A `401` response with a `SESSION` cookie set indicates the app is up and Spring Security is active as expected (not an error).
