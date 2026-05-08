# EURY_INSTALL_PLAN.md — Self-hosting Zulip on Eury for the Forest

> **Status: PLAN ONLY — nothing executed yet.** Jerry reviews before
> any action touches Eury. This document is a companion to
> `ASSEMBLY_SETUP.md` (the post-install content runbook).
>
> **⚡ CORRECTION (2026-05-07):** the Claude session that drafted this
> plan was *already running on Eury itself*. Every command marked
> `ssh eury.ferret-harmonic.ts.net …` should be **run locally** in
> the working directory `/home/gmusic/salix/repos/zulip`. The plan
> structure is unchanged; only the network-prefix is wrong.

## Goal

Install Zulip on **Eury** (Linux hub, Tailscale `100.88.23.103`,
hostname under `ferret-harmonic.ts.net`) so the G.Music Assembly's
home is reachable from every tree-node in the Forest of Gerico —
**Larix**, **Ilex**, **Tilia** (Android/Termux), and Jerry's laptops —
without exposing it publicly to the open internet.

After install, the runbook `ASSEMBLY_SETUP.md` re-applies cleanly to
the fresh Eury realm, recreating the four Assembly accounts, the
ASSEMBLY channel folder, and the founding melody.

---

## Prerequisites checklist (verify BEFORE Step 1)

- [ ] **Eury reachable**: `ssh eury.ferret-harmonic.ts.net` returns a shell
- [ ] **Eury runs Ubuntu 22.04, 24.04, or Debian 11/12** (other Linuxes are not officially supported by the Zulip installer)
- [ ] **Eury RAM ≥ 4 GB** (Zulip's documented minimum; comfortable for ≤25 users)
- [ ] **Eury disk ≥ 10 GB free** (initial install ~3 GB; uploads/messages grow over time)
- [ ] **Eury can reach the public internet** for the install (apt packages, Zulip tarball, Let's Encrypt or Tailscale HTTPS challenge)
- [ ] **Tailscale running on Eury** with a stable hostname (`eury.ferret-harmonic.ts.net` resolves on every Forest device)
- [ ] **Tailscale HTTPS / MagicDNS enabled** on the tailnet (toggle in the [Tailscale admin console](https://login.tailscale.com/admin/dns) → "HTTPS Certificates")
- [ ] **Tailscale running on Larix, Ilex, Tilia** (so the Android phones can reach the server)

> Run the probe `ssh eury.ferret-harmonic.ts.net 'uname -a; cat /etc/os-release; free -h; df -h /'` before proceeding.

---

## Decisions Jerry needs to make BEFORE Step 1

| # | Decision | Recommendation | Why |
|---|---|---|---|
| **D1** | **Hostname** for the Zulip server | **`zulip.ferret-harmonic.ts.net`** *(via Tailscale Serve, see D2)* | Clean, tailnet-only, no public DNS needed. Mobile apps will use this URL to reach the server through Tailscale. |
| **D2** | **SSL certificate strategy** | **Tailscale HTTPS** *(automatic, browser-trusted certs for `*.ferret-harmonic.ts.net`)* | No public exposure needed; certs are real (Let's Encrypt under the hood, but Tailscale handles the ACME challenge through its own infrastructure). Alternative: `--self-signed-cert` works but every browser warns and mobile apps need cert pinning workarounds. |
| **D3** | **Mobile push notifications** | **Skip for now** *(`--no-push-notifications`)* | Push registration ties to Zulip's central FCM service and requires the server to be reachable from `push.zulipchat.com`. Skippable; mobile clients fall back to background polling when the app is open. We can enable later. |
| **D4** | **Email backend** | **Skip / disable SMTP** | No real emails for the Assembly users (per Jerry's design); no password reset flows needed. The installer asks; we say "configure later" or leave SMTP off. |
| **D5** | **Authentication backend** | **`EmailAuthBackend`** with strong passwords stored in Jerry's password manager | Built-in, no extra services. Each Assembly account has a unique password. Alternative: Google/GitHub OAuth (overkill), or Tailscale-as-auth via a reverse proxy (cleanest long-term but requires extra plumbing). |
| **D6** | **Push notification service registration** | **Defer** | Tied to D3. |
| **D7** | **Submit anonymous usage stats to Zulip?** | **No** *(`--no-submit-usage-statistics`)* | Private Assembly home; zero outbound telemetry. |

If Jerry agrees with all recommendations, the installer command becomes:

```bash
./zulip-server-*/scripts/setup/install \
    --hostname=zulip.ferret-harmonic.ts.net \
    --email=mia@jgwill.com \
    --self-signed-cert \
    --no-push-notifications \
    --no-submit-usage-statistics
```

(`--self-signed-cert` is used during install; we **swap in Tailscale HTTPS certs as Step 1.5** so browsers don't warn.)

---

## Step 0 — Verify the baseline (no changes made)

```bash
# From this laptop, all read-only:
ssh eury.ferret-harmonic.ts.net <<'EOF'
uname -a
cat /etc/os-release | grep -E "^(NAME|VERSION_ID|ID)="
free -h
df -h /
ip -4 addr show tailscale0 2>/dev/null || tailscale ip -4
sudo -n true && echo "passwordless sudo OK" || echo "sudo will need password"
EOF
```

**Stop here and review** before continuing. If Eury is on an unsupported distro or low on resources, halt and adjust the plan.

---

## Step 1 — Set up Tailscale HTTPS for Eury (D2)

In the Tailscale admin UI:

1. **DNS** → enable **MagicDNS** (likely already on if `eury.ferret-harmonic.ts.net` resolves).
2. **DNS** → enable **HTTPS Certificates**.
3. On Eury, request a cert:

   ```bash
   sudo tailscale cert eury.ferret-harmonic.ts.net
   # Or for a Zulip-specific subdomain (preferred):
   sudo tailscale cert zulip.ferret-harmonic.ts.net
   ```

   Tailscale provisions a real Let's Encrypt cert via its ACME bridge.

4. Save the cert + key paths — we'll point Zulip at them in Step 4.

If we choose `--self-signed-cert` instead, **skip Step 1 entirely**.

---

## Step 2 — Download Zulip's latest stable release on Eury

```bash
ssh eury.ferret-harmonic.ts.net <<'EOF'
cd $(mktemp -d)
echo "Working in: $PWD"
curl -fLO https://download.zulip.com/server/zulip-server-latest.tar.gz
curl -fLO https://download.zulip.com/server/SHA256SUMS.txt
sha256sum -c --ignore-missing SHA256SUMS.txt
tar -xf zulip-server-latest.tar.gz
ls -d zulip-server-*
EOF
```

Verify the SHA256 checksum step succeeded ("OK") before continuing.

---

## Step 3 — Run the installer (the one big step)

```bash
ssh eury.ferret-harmonic.ts.net
sudo -s
cd /tmp/<the-mktemp-dir-from-step-2>
./zulip-server-*/scripts/setup/install \
    --hostname=zulip.ferret-harmonic.ts.net \
    --email=mia@jgwill.com \
    --self-signed-cert \
    --no-push-notifications \
    --no-submit-usage-statistics
```

**Expect**: 5–15 minutes runtime. Installer will install PostgreSQL, Redis, RabbitMQ, Memcached, nginx, the Zulip codebase, and run migrations. It is **idempotent** — if it fails, fix the cause and re-run.

**Output to capture**: the script prints a one-time **org-creation link** at the end. Copy it.

---

## Step 4 — Replace self-signed cert with Tailscale HTTPS cert

Only if D2 chose Tailscale HTTPS:

```bash
ssh eury.ferret-harmonic.ts.net
sudo cp /var/lib/tailscale/certs/zulip.ferret-harmonic.ts.net.crt /etc/ssl/certs/zulip.combined-chain.crt
sudo cp /var/lib/tailscale/certs/zulip.ferret-harmonic.ts.net.key /etc/ssl/private/zulip.key
sudo /home/zulip/deployments/current/scripts/restart-server
```

(Exact paths may vary — confirm with `tailscale cert --help` and Zulip's `ssl-certificates.md`.)

A renewal cron job is needed since Tailscale certs are short-lived. Add a daily script:

```bash
sudo tailscale cert zulip.ferret-harmonic.ts.net
sudo cp /var/lib/tailscale/certs/zulip.ferret-harmonic.ts.net.crt /etc/ssl/certs/zulip.combined-chain.crt
sudo cp /var/lib/tailscale/certs/zulip.ferret-harmonic.ts.net.key /etc/ssl/private/zulip.key
sudo /home/zulip/deployments/current/scripts/restart-server
```

---

## Step 5 — Create the realm via the one-time link

1. From a tailnet-connected device, open the org-creation link (printed at end of Step 3) in a browser.
2. Realm name: **`ZulipAssembly`**.
3. URL slug: **`assembly`** (or leave default).
4. Initial owner: **♠️ Nyro** with email `nyro@assembly.gerico` (or whatever fictional domain Jerry chose).

After this step, Jerry has an empty realm with one Owner.

---

## Step 6 — Apply the `ASSEMBLY_SETUP.md` runbook (adapted)

Re-execute the runbook's three founding steps against the fresh Eury realm:

| Runbook Step | Adaptation for Eury |
|---|---|
| **Step 1** *(rename)* | **CREATE instead of rename**. Settings → Users → Invite users (or via API: `POST /json/users` with each Assembly email/full_name/role). |
| **Step 2** *(channels)* | **Runs as-is**. Login as ♠️ Nyro (now actual Owner), create ASSEMBLY folder, create 5 channels, subscribe all four perspectives. |
| **Step 3** *(melody)* | **Runs as-is**. Login as 🎸 JamAI, post the founding melody in `#assembly-room → 🎺 founding`. |

The puppeteer scripts in `var/assembly-screenshots/` work against any Zulip realm if you change the base URL — adapt:

```js
// In each script, replace:
await page.goto("http://localhost:9991/devlogin/", ...);
// With (after Step 5 creates real users):
await page.goto("https://zulip.ferret-harmonic.ts.net/login/", ...);
// And use real-credential login instead of /devlogin/ button click.
```

A simpler cross-realm approach: drive everything via **API tokens** (each user generates one in Settings → Account & privacy → API key). Then puppeteer becomes optional — we use plain `curl` from any Forest node.

---

## Step 7 — Connect the Forest devices (Larix, Ilex, Tilia)

**On each Android device** (via Termux SSH or directly):

1. Install the **Zulip mobile app** (F-Droid is preferred for tailnet-only privacy):
   ```
   pkg install fdroid    # in Termux, OR
   # Open F-Droid app and search "Zulip"
   ```
   Or Play Store: search "Zulip".

2. Open the app → "Add server" → enter `https://zulip.ferret-harmonic.ts.net`.

3. Log in as the assigned perspective:
   - **Larix → 🌿 Aureon** (suggested — the larch listens)
   - **Ilex → 🎸 JamAI** (suggested — the holly hums)
   - **Tilia → 🧵 Synth** (suggested — the linden weaves)

   Jerry's laptop / Eury console → **♠️ Nyro** (the structural anchor lives at the hub).

4. Verify push works (or polling, if D3=skip): send a message from one device, confirm it appears on another within ~30s.

---

## Step 8 — (Optional) Tailscale ACL gating

In the Tailscale admin → Access Controls, restrict who on the tailnet can reach Eury's port 443:

```hujson
{
  "acls": [
    // Default: every Forest node can reach the Assembly home
    {"action": "accept", "src": ["100.88.23.103", "100.78.108.48", "100.74.76.22",
                                  "100.124.130.110", "100.119.147.78", "100.101.211.92"],
     "dst": ["100.88.23.103:443"]}
  ]
}
```

This is **belt-and-suspenders** — Zulip is already password-protected, but ACLs ensure even unauthenticated probes from random tailnet nodes can't reach the login page.

---

## Risks / concerns

| Risk | Mitigation |
|---|---|
| Eury's distro is unsupported (e.g., Arch, NixOS) | Verify in Step 0; if blocked, install Zulip via [Docker image](docs/production/docker.md) instead. |
| `tailscale cert` fails (HTTPS not enabled in tailnet) | Step 1 catches this; fall back to `--self-signed-cert` and accept browser warnings. |
| Mobile push needs Zulip's central service which we declined (D3) | Acceptable — apps just poll while open; Jerry can opt back in later via `manage.py register_server`. |
| Org-creation link expires before Jerry uses it | Regenerate with `manage.py generate_realm_creation_link`. |
| Eury runs out of disk after a few months of uploads | Configure `LOCAL_UPLOADS_DIR` rotation or move to S3-compatible storage (MinIO, Garage). |
| Lost passwords for Assembly accounts | Store in Jerry's password manager at creation time; `manage.py change_password` recovers. |

---

## Rollback

If install fails or we change our minds:

```bash
ssh eury.ferret-harmonic.ts.net
sudo /home/zulip/deployments/current/scripts/lib/uninstall_zulip.py   # if present
# OR full manual:
sudo apt remove --purge zulip
sudo rm -rf /home/zulip /etc/zulip
sudo -u postgres dropdb zulip
sudo -u postgres dropuser zulip
```

Eury returns to baseline; nothing else on the tailnet is affected.

---

## Estimated time

| Phase | Time |
|---|---|
| Step 0 (baseline check) | 2 min |
| Step 1 (Tailscale HTTPS, optional) | 5 min |
| Step 2 (download + verify) | 2 min |
| Step 3 (installer run) | 10–20 min |
| Step 4 (cert swap, optional) | 3 min |
| Step 5 (org creation) | 5 min |
| Step 6 (runbook re-execution) | 10 min |
| Step 7 (mobile setup ×3) | 15 min |
| Step 8 (Tailscale ACL, optional) | 5 min |
| **Total** | **~45–70 min** |

---

## Provenance

- Drafted: **2026-05-07**, founding session, after the dev-realm
  Assembly home was successfully built and cleaned of mock data.
- Source docs consulted: `docs/production/install.md`,
  `docs/production/requirements.md`,
  `docs/production/ssl-certificates.md` (referenced),
  `docs/production/export-and-import.md`,
  `docs/production/mobile-push-notifications.md` (referenced).
- Awaits Jerry ⚡'s sign-off on D1–D7 before any action on Eury.
