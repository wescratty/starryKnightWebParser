# One-time setup — Windows (your test box)

This is your Windows stand-in for the Mac setup — same steps, same Shopify app, adapted for PowerShell. The goal here is to get a few real runs in and see what the parsing/reporting loop actually does before this goes anywhere near her machine. I'm assuming `C:\dev\Cowork\GIT\StarryKnight_automation` is the parent workspace folder (mirroring `~/dev/Cowork/GIT` on the Mac doc), with the repo cloned as a subfolder inside it — adjust the paths below if you meant that folder to BE the clone itself.

Run these in **PowerShell**, not CMD.

---

## 1. Install Claude Code

```powershell
irm https://claude.ai/install.ps1 | iex
```

[Install docs](https://code.claude.com/docs/en/setup) — same native installer as Mac/Linux, just the PowerShell variant. Auto-updates in the background afterward.

```powershell
claude --version
claude
```

Log in via the browser prompt, then `/exit` once you see the interactive session start.

## 2. Install and authenticate GitHub CLI

```powershell
winget install --id GitHub.cli
gh auth login
```

Same prompts as the Mac version: GitHub.com → HTTPS → authenticate Git with GitHub credentials → Yes → login with a web browser.

```powershell
gh auth status
```

## 3. Create the workspace and clone the repo

```powershell
mkdir C:\dev\Cowork\GIT\StarryKnight_automation -Force
cd C:\dev\Cowork\GIT\StarryKnight_automation
gh repo clone wescratty/starryKnightWebParser
cd starryKnightWebParser
```

Confirm `handoff.md` is in this folder (copy it in if you built it somewhere else first).

## 4. Shopify: same app as the Mac doc

Use the exact same custom app from `One-time-setup.md` step 4 — same Client ID and Client Secret. Don't recreate it. [Shopify's setup docs](https://help.shopify.com/en/manual/apps/install-setup-apps) · [client credentials docs](https://shopify.dev/docs/apps/build/authentication-authorization/client-credentials-grant)

Windows doesn't have a direct CLI equivalent to macOS Keychain that's simple to script against, so for this test box, store the two values in a git-ignored local file rather than typing them into every command:

```powershell
# Make sure this pattern is in .gitignore before creating the file
Add-Content .gitignore "`nsecrets.local.ps1"

@"
`$env:SHOPIFY_CLIENT_ID = "PASTE_CLIENT_ID_HERE"
`$env:SHOPIFY_CLIENT_SECRET = "PASTE_CLIENT_SECRET_HERE"
"@ | Out-File -Encoding utf8 secrets.local.ps1
```

Load it into your session and test the token exchange, same as the Mac step:

```powershell
. .\secrets.local.ps1

$body = "grant_type=client_credentials&client_id=$($env:SHOPIFY_CLIENT_ID)&client_secret=$($env:SHOPIFY_CLIENT_SECRET)"
Invoke-RestMethod -Method Post -Uri "https://HER-SHOP-NAME.myshopify.com/admin/oauth/access_token" `
  -ContentType "application/x-www-form-urlencoded" -Body $body
```

Replace `HER-SHOP-NAME` with the real `.myshopify.com` subdomain. You should get back JSON with `access_token`, `scope`, and `expires_in` (86399 seconds ≈ 24 hours — a fresh token gets minted every run, this isn't a one-time thing).

**Don't move on until this returns a token.** If it fails here, it'll fail identically inside a scheduled run, just harder to debug.

## 5. Run it by hand first — this is the part you actually care about

Before wiring up any scheduler, just run Claude directly against the repo a few times and watch what it does:

```powershell
cd C:\dev\Cowork\GIT\StarryKnight_automation\starryKnightWebParser
. .\secrets.local.ps1
claude -p "Run the weekly order-parsing job per handoff.md." --permission-mode acceptEdits --allowedTools "Bash,Read,Edit"
```

This is where you'll actually learn things — whether it pulls orders correctly, how it tags ambiguous cases, whether the GitHub issue it opens looks right, whether `edge-cases.md` fills in sensibly. Run it a few times, read what it produces, adjust `handoff.md`'s wording based on what you see, repeat. No need to touch a scheduler yet.

## 6. Optional: test the actual wake + scheduled-trigger mechanics

Skip this until step 5 is producing good runs. If/when you want to test that the trigger itself works (not just the run logic), Windows Task Scheduler is the equivalent of the Mac's `pmset` + `launchd`, and it can do both wake-and-run in one place — no separate wake command needed:

```powershell
$Action = New-ScheduledTaskAction -Execute "powershell.exe" `
  -Argument "-NoProfile -Command `"cd C:\dev\Cowork\GIT\StarryKnight_automation\starryKnightWebParser; . .\secrets.local.ps1; claude -p 'Run the weekly order-parsing job per handoff.md.' --permission-mode acceptEdits --allowedTools 'Bash,Read,Edit' *>> run.log`""

$Settings = New-ScheduledTaskSettingsSet -WakeToRun

$Trigger = New-ScheduledTaskTrigger -Weekly -DaysOfWeek Sunday -At 12:00PM

Register-ScheduledTask -TaskName "StarryKnightOrderParse" -Action $Action -Settings $Settings -Trigger $Trigger
```

`-WakeToRun` is Task Scheduler's built-in equivalent of the Mac's `pmset repeat wakeorpoweron` — one flag does both. Test it manually before trusting the schedule:

```powershell
Start-ScheduledTask -TaskName "StarryKnightOrderParse"
Get-Content .\run.log -Wait
```

To remove it later: `Unregister-ScheduledTask -TaskName "StarryKnightOrderParse" -Confirm:$false`

---

## What's different from the Mac version

- No Keychain — using a git-ignored local `.ps1` file instead. Fine for a test box; don't carry that habit over to her Mac, where Keychain is the real answer.
- Windows can wake-and-schedule in one step (`-WakeToRun` on the task itself); macOS needs `pmset` and `launchd` as two separate pieces.
- Everything else — the Claude install, `gh` auth, the Shopify credentials, the actual `claude -p` run command — is identical. That's the point: whatever you learn from these test runs about how `handoff.md` should read applies directly to her machine.
