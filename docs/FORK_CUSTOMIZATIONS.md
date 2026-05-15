# Fork Customizations (JLandersZen/gstack)

This document describes intentional divergences from upstream (garrytan/gstack).
These are permanent architectural decisions, not patches waiting to be upstreamed.

## 1. Persistent Chromium Profile for Headless Mode

**File:** `browse/src/browser-manager.ts`
**Commit:** `54179a5`

### What changed

Upstream's `launch()` (headless auto-start) uses `chromium.launch()` +
`browser.newContext()`, which creates an ephemeral in-memory context. Cookies,
localStorage, and session state vanish when the daemon restarts.

Our fork switches `launch()` to `launchPersistentContext()` with
`resolveChromiumProfile()` — the same persistent profile directory that headed
mode already uses. Cookies from headed sessions are now visible in headless mode
automatically.

### Why

Three reasons, all security and operational:

1. **Keychain decryption triggers security alerts.** Upstream's `/setup-browser-cookies`
   skill decrypts cookies from your real browser via the OS keychain. In a corporate
   environment with endpoint monitoring (CrowdStrike, SentinelOne, etc.), this is
   indistinguishable from credential theft. Security teams flag it and contact you.
   Our approach never touches your real browser's cookie store.

2. **Principle of least privilege.** The upstream approach dumps cookies from your
   daily-driver browser into the agent session. Our approach gives the agent its own
   dedicated Chromium profile. You log in only to the specific sites the agent needs,
   in headed mode, using a separate browser identity. The agent never sees your Gmail,
   Slack, or anything you didn't explicitly grant.

3. **Eliminate unnecessary complexity.** The entire `/setup-browser-cookies` workflow
   (detect browsers, open picker UI, decrypt cookies, inject into session, re-import
   after every restart) becomes unnecessary. Log in once in headed mode (`/browse` or
   `/open-gstack-browser`), and headless mode sees the same cookies. No decryption,
   no import, no re-import.

### How to authenticate sites for the agent

1. Run `/browse` or `/open-gstack-browser` to launch headed Chromium
2. Log in to the sites you want the agent to access
3. Close the headed browser or switch to headless — sessions persist
4. All browse commands (`/qa`, `/scrape`, `/design-review`, etc.) now have access

This is the same Chromium profile, just a separate browser from your daily driver.
The agent has its own identity and its own sessions.

## 2. `/setup-browser-cookies` Skill Removed

**Reason:** No longer needed. The persistent profile approach (above) replaces
cookie import entirely. Removing it prevents accidental use that would trigger
security alerts in corporate environments.

The upstream skill files (`setup-browser-cookies/SKILL.md` and
`setup-browser-cookies/SKILL.md.tmpl`) are deleted from this fork. The
`/merge-upstream` skill is configured to keep them deleted when merging.

### What to do instead

Anywhere upstream docs reference `/setup-browser-cookies`, the equivalent in
this fork is: launch headed mode, log in, done.
