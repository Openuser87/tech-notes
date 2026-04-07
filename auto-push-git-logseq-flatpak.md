# Logseq Auto-Push Setup (Flatpak)

We have a Flatpak-based Logseq installation and want to configure automated pushing to a self-hosted (e.g. Gitea) repo.

Repo is cloned to a dir `~/graph`. The original Logseq's `~/.logseq/.git` "infrastructure" is not used.

We also have self-signed certificates on our repo management webapp.

We create a `post-commit` hook under `~/graph/.git/hooks` and enable auto-commit in Logseq (time based + on exit)

But - although we see commits in `git log -p`, automatic push does not work.

## Preliminary Checks

* **Ensure the hook is executable:**
  ```
  chmod +x ~/graph/.git/hooks/post-commit
  ```
* **Verify the Flatpak has network access:** e.g. use flatseal or
  ```
  flatpak info --show-permissions com.logseq.Logseq
  ```

## Add Debug Entries to the Hook

* **Use relative paths or paths inside the graph directory for the log file.** It must be inside the Flatpak sandbox or a directory it has permission to access.
    * Note: Using absolute paths like ~/ may result in no output if the sandbox cannot write there.
* **Redirect both stdout and stderr to a file within the graph directory:**

```bash
#!/bin/bash
echo "--- Hook triggered at $(date) ---" >> git_debug.log
git push origin main >> git_debug.log 2>&1
echo "Exit code: $?" >> git_debug.log
```
For extended output using the host's network stack and CA certificates:
```bash
#!/bin/bash

# Use flatpak-spawn to utilize the host's git and environment
TOKEN="your_auth_token_here"
REPO_URL="your.server.address/repo.git"

flatpak-spawn --host /usr/bin/git -C "$PWD" push https://$TOKEN@$REPO_URL main
```
## Troubleshooting "ServiceUnknown" Error

If flatpak-spawn fails with
```
Portal call failed: org.freedesktop.DBus.Error.ServiceUnknown, enable the D-Bus interface:
```
enable permission:
```
flatpak override --user --talk-name=org.freedesktop.Flatpak com.logseq.Logseq
```
To disable permission (when no longer needed):
```
flatpak override --user --no-talk-name=org.freedesktop.Flatpak com.logseq.Logseq
```
## Add Certificate in Flatpak Environment

If using a self-signed certificate, the Flatpak environment requires the CA bundle.

### Option A: Pass host certificates to the sandbox

```
flatpak override --user --filesystem=xdg-run/keyring com.logseq.Logseq
```

### Option B: Disable SSL verification for the specific repo (less secure)

```
git config http.sslVerify false
```

### Option C: Add certificate to dugite

Retrieve your certificate (e.g. from your CA server).

Append the certificate to the Flatpak bundle:

```
sudo vim /var/lib/flatpak/app/com.logseq.Logseq/current/active/files/logseq/resources/app/node_modules/dugite/git/ssl/cacert.pem
```

## Remote Authentication

Embed an Access Token in the remote URL to bypass credential manager prompts.

1. Generate a Token:
    * In your Git server (e.g., Gitea/GitHub), go to Settings > Applications.
    * Generate a token with repo (write) scope.
2. Update the Git configuration:
    * Edit .git/config within your graph directory:
    url = https://<TOKEN_HERE>@your.server.address/user/repo.git

## Check Flatpak Permissions

* Host Communication:
  `flatpak override --user --talk-name=org.freedesktop.Flatpak com.logseq.Logseq`
* Filesystem Access: (If the hook writes to directories outside the standard sandbox)
    `flatpak override --user --filesystem=/path/to/target/dir com.logseq.Logseq`
* Visual Audit: Use Flatseal to review and manage all active overrides.
