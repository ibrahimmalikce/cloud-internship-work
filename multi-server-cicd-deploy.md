# Multi-Server CI/CD Deploy — admin-api (GitHub Actions)

## Context

`admin-api` (Java/Maven, Tomcat) needed to run on three servers:

- `dev.wirepick.com` — dev server, self-hosted GitHub Actions runner already registered here
- `vps3427251` — production, Tomcat at `/opt/tomcat/webapps/`, shares the box with 4 other live apps
- `vps3460762` — production, Tomcat also at `/opt/tomcat/webapps/`, shares the box with 5 other live apps

Goal: one push to `master` builds once (on server 1) and rolls the same WAR out to the other two, without needing a self-hosted runner on every box.

## Approach

- **Job 1** (`build-and-deploy-server1`) runs on the existing self-hosted runner on `dev.wirepick.com`. Builds with Maven, deploys locally by copying the WAR into that server's webapps folder, then uploads the WAR as a GitHub Actions artifact.
- **Job 2** (`deploy-remaining-servers`) runs on a normal GitHub-hosted runner, downloads that artifact, and SCPs it out to each of the other two servers over SSH. Both of those servers have `autoDeploy="true"` in `server.xml`, so Tomcat picks up the new WAR on its own — no restart step needed.

## Gotchas hit along the way

**1. Wildcard delete had already burned us once**
An earlier version of the workflow used `rm -f webapps/*.war` before copying in the new WAR, which deleted every other app's WAR file on the shared server, not just ours. Fixed by scoping every file operation (both `cp` and the SCP `source`) to the exact WAR filename — never a folder-wide wildcard.

**2. `secrets` context isn't allowed inside `strategy.matrix`**
First attempt put per-server host/user secrets directly inside the matrix definition:
```yaml
strategy:
  matrix:
    target:
      - name: vps3427251
        host: ${{ secrets.VPS3427251_HOST }}
```
This fails with `Invalid workflow file... Unrecognized named-value: 'secrets'`. GitHub evaluates matrix values before the secrets context is available there. Fix: keep the matrix to plain names only, and resolve the actual secret values inside the step with a conditional expression:
```yaml
strategy:
  matrix:
    target: [vps3427251, server3]
steps:
  - uses: appleboy/scp-action@v0.1.7
    with:
      host: ${{ matrix.target == 'vps3427251' && secrets.VPS3427251_HOST || secrets.SERVER3_HOST }}
```

**3. SCP'd files land with the wrong owner**
Files copied in via SCP are owned by whatever the SSH deploy user resolves to (e.g. `1001:1001` or `1001:cpanelsuspended`), not the app's expected owner (`root:root` on vps3427251, `tomcat:tomcat` on vps3460762). Deployment still worked since Tomcat could read the file, but it's inconsistent with every other WAR in the folder and a latent risk if Tomcat or another process ever needs to overwrite/delete it as its usual service user. Fixed by adding a step that SSHs back in after the copy and runs `chown` (and `chown -R` on the exploded directory) using the ownership convention already in use on that server.

**4. Private key exposure**
While generating the SSH keypair, the private key contents got shown on-screen in a screenshot shared for troubleshooting. Any private key that's been displayed outside of secure secret storage should be treated as compromised — rotate it (new keypair, swap `authorized_keys` on both servers, update the `DEPLOY_SSH_KEY` GitHub secret) rather than relying on deleting the exposed copy after the fact.

## Result

First full run succeeded end-to-end: build + local deploy on `dev.wirepick.com`, then SCP + auto-deploy to both `vps3427251` and `vps3460762`, with every other app's files left untouched and ownership now matching each server's convention.
