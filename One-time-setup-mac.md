# One-time setup — on her Mac, in Terminal

Do these once, at her computer, in order. Each step includes what to run and what to check before moving on. I looked up current docs for each piece rather than going from memory, since a couple of things (Shopify's token flow especially) changed recently — see the links.

---

## 1. Install Claude Code

```bash
curl -fsSL https://claude.ai/install.sh | bash
```

This is the current official native installer (auto-updates itself in the background afterward). [Install docs](https://code.claude.com/docs/en/setup)

Verify it worked:

```bash
claude --version
```

Then log in — this requires her own Claude account (Pro, Max, Team, or Enterprise; the free claude.ai plan doesn't include Claude Code):

```bash
claude
```

It'll open a browser for login. Once you see the interactive prompt, type `/exit` to leave — you don't need to do anything inside it yet.

## 2. Install and authenticate GitHub CLI

To answer your question directly: you don't do anything on github.com first. `gh auth login` is a real Terminal command — it opens a browser for you at the right moment.

```bash
brew install gh
gh auth login
```

When prompted:
- **What account do you want to log into?** → GitHub.com
- **Preferred protocol** → HTTPS (this matters — it means no separate SSH key to manage; `gh` becomes git's credential helper automatically)
- **Authenticate Git with your GitHub credentials?** → Yes
- **How would you like to authenticate?** → Login with a web browser

Verify:

```bash
gh auth status
```

Should show logged in as `wescratty` (or whichever account has access to the repo) with the `repo` scope.

## 3. Create the workspace and clone the repo

No `ssh-keygen` needed — `gh` from step 2 handles auth for both cloning and later opening issues.

```bash
mkdir -p ~/dev/Cowork/GIT
cd ~/dev/Cowork/GIT
gh repo clone wescratty/starryKnightWebParser
cd starryKnightWebParser
```

Confirm `handoff.md` (and `edge-cases.md`, once created) live in this folder — that's the path her Claude will be pointed at.

## 4. Shopify: create a custom app and get API credentials

Heads up — Shopify moved this to their "Dev Dashboard" and changed how tokens are issued. It's no longer a simple copy-paste of a token from the admin; there's one extra step (a `curl` call) each time you need a fresh token. Not hard, just worth knowing going in. [Shopify's setup docs](https://help.shopify.com/en/manual/apps/install-setup-apps) · [client credentials docs](https://shopify.dev/docs/apps/build/authentication-authorization/client-credentials-grant)

In her Shopify admin:

1. **Settings** → **Apps** → **Develop apps** → **Build apps in Dev Dashboard**
2. **Create app**, give it a name like `order-parser-readonly`
3. In the app's **Access** section, add the scope `read_orders` (only that scope — no write access, no other data)
4. Click **Release**, confirm
5. Back in Dev Dashboard, open the app → **Installs** → **Install app** → select her store → **Install**
6. Find the app's **Client ID** and **Client Secret** (in the app's API credentials section) — copy both

Store them in the Mac's Keychain rather than a plaintext file:

```bash
security add-generic-password -a "starryknight" -s "shopify-client-id" -w "PASTE_CLIENT_ID_HERE"
security add-generic-password -a "starryknight" -s "shopify-client-secret" -w "PASTE_CLIENT_SECRET_HERE"
```

Each weekly run mints a fresh access token from these two values (tokens expire after 24 hours, so this happens every run, not once):

```bash
CLIENT_ID=$(security find-generic-password -a "starryknight" -s "shopify-client-id" -w)
CLIENT_SECRET=$(security find-generic-password -a "starryknight" -s "shopify-client-secret" -w)

curl -s -X POST "https://HER-SHOP-NAME.myshopify.com/admin/oauth/access_token" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "grant_type=client_credentials&client_id=${CLIENT_ID}&client_secret=${CLIENT_SECRET}"
```

Replace `HER-SHOP-NAME` with her actual `.myshopify.com` subdomain. That returns a short-lived `access_token` good for reading orders via the Admin API. Claude's run script (built separately, once we design the actual weekly-run prompt) will do this automatically — you're just confirming here that the credentials work before we wire it up.

**Test it once by hand** before moving on — if this curl call doesn't return an `access_token`, nothing downstream will work, so worth catching now rather than at noon on a Sunday.

## 5. Set the Mac to wake up before the run

```bash
sudo pmset repeat wakeorpoweron U 11:55:00
```

`U` = Sunday, `wakeorpoweron` (not just `wake`) so it also works if the Mac got fully shut down rather than just asleep. This wakes it 5 minutes before noon. [pmset reference](https://ss64.com/mac/pmset.html)

Verify it's set:

```bash
pmset -g sched
```

Also: **System Settings → General → Login Items & Extensions** — make sure Claude (and anything else the run needs open) is set to open at login, so it's actually running once the Mac wakes.

## 6. Create the Sunday-noon scheduled run

This uses a macOS LaunchAgent (not `cron` — `cron` doesn't reliably survive sleep/wake the way `launchd` does).

Create the plist:

```bash
mkdir -p ~/Library/LaunchAgents
cat > ~/Library/LaunchAgents/com.starryknight.orderparse.plist << 'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.starryknight.orderparse</string>
    <key>ProgramArguments</key>
    <array>
        <string>/bin/zsh</string>
        <string>-lc</string>
        <string>cd ~/dev/Cowork/GIT/starryKnightWebParser && claude -p "Run the weekly order-parsing job per handoff.md." --permission-mode acceptEdits --allowedTools "Bash,Read,Edit" >> ~/dev/Cowork/GIT/starryKnightWebParser/run.log 2>&1</string>
    </array>
    <key>StartCalendarInterval</key>
    <dict>
        <key>Weekday</key>
        <integer>0</integer>
        <key>Hour</key>
        <integer>12</integer>
        <key>Minute</key>
        <integer>0</integer>
    </dict>
    <key>StandardErrorPath</key>
    <string>~/dev/Cowork/GIT/starryKnightWebParser/run-error.log</string>
</dict>
</plist>
EOF

launchctl load ~/Library/LaunchAgents/com.starryknight.orderparse.plist
```

`Weekday 0` = Sunday. `--permission-mode acceptEdits` lets Claude write files (like updates to `edge-cases.md`) without a permission prompt popping up with nobody there to answer it; `--allowedTools "Bash,Read,Edit"` scopes what it can touch without asking — notably no auto-approved network/push access beyond what `gh` and the Shopify curl calls need, which the handoff.md instructs it to use directly. [Headless-mode CLI reference](https://code.claude.com/docs/en/headless)

**Test it by hand first**, don't wait for Sunday to find out it's broken:

```bash
launchctl start com.starryknight.orderparse
tail -f ~/dev/Cowork/GIT/starryKnightWebParser/run.log
```

If something's wrong, `launchctl unload` the plist, fix it, and `load` again.

---

## What's still open (not covered here)

- The exact wording of the run prompt inside `handoff.md` will get refined once we've seen a real run or two.
- The email-drafting assistant (Etsy/website/Shopify inboxes) isn't part of this setup — separate piece, still being designed.
