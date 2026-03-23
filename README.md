# DNS Fallback for Asuswrt-Merlin

Simple, reliable cron script (1min) for Pi-hole/AdGuard/Unbound failover on Asuswrt-Merlin routers. Monitors resolver with ping+dig, auto-switches LAN DHCP DNS to public fallbacks (Cloudflare/Google/etc.) on failure. Restores with anti-flap hysteresis (consecutive OK checks + minimum fallback window). Log rotation & optional email included. No dependencies beyond Merlin+JFFS.

## Features
- **Real DNS test**: dig + ping
- **Safe**: Uses native NVRAM dhcp_dns1_x/2, no hacks
- **Lightweight**: <4s exec, zero deps
- **Docker-ready**: Works with containerized Pi-hole/Unbound
- **Anti-flap**: Fast failover + controlled restore

## Quick Install
```bash
# Download
curl -O https://raw.githubusercontent.com/fdezdaniel/dns-fallback-merlin/main/dns-fallback
chmod +x dns-fallback
mv dns-fallback /jffs/scripts/
```

# Cron (every 1min)

```bash
# Add this line to your router's cron (runs every minute):
*/1 * * * * /jffs/scripts/dns-fallback
```

# Customize
Edit top vars:

```bash
RESOLVER_IP="192.168.1.100"  # Your Pi-hole/AdGuard/Unbound/others IP
FALLBACK_DNS1="1.1.1.1"      # Primary fallback
FALLBACK_DNS2="8.8.8.8"      # Secondary
DNS_TEST_DOMAIN="google.com"  # Domain used for DNS check
```

Anti-flap/hysteresis defaults (already in the script):

```bash
FAILOVER_FAILS_REQUIRED=1      # Switch to fallback after 1 failed run
RESTORE_SUCCESSES_REQUIRED=2   # Restore after 2 consecutive OK runs
MIN_FALLBACK_SECONDS=90        # Keep fallback at least this long
STATE_FILE="/jffs/dns-fallback.state"
```

If you are on calls/meetings and want less flap: keep failover at 1, tune restore/window only.

### Optional: email notifications

This script can send email alerts on DNS failure/restore using the router's built‑in email function configured via **amtm**:

- Make sure amtm email is configured (`amtm` → `em`).
- On some setups, the mail helper is exposed as `/jffs/scripts/email-notify` or `<alias-name>-email-notify`.
- If needed, create a small wrapper `/jffs/scripts/email-notify` that calls your actual amtm mail script (so `/jffs/scripts/dns-fallback` can use `/jffs/scripts/email-notify "subject"`).

If email is not configured, you can safely remove or comment out the `email-notify` lines.


# Logs
`/jffs/dns-fallback.log` auto-rotates @50KB. Example:

```text
[2026-02-14 11:35:12] Resolver DOWN (ping:0 dns:1), switching to fallback...
[2026-02-14 11:36:02] Resolver OK but holding fallback for anti-flap window (24s/90s)
[2026-02-14 11:37:40] Resolver OK (ping:0 dns:0), restoring as sole LAN DNS
```

State file:

```text
/jffs/dns-fallback.state
```

Stores counters/timestamps between cron runs (`FAIL_STREAK`, `OK_STREAK`, `LAST_SWITCH_EPOCH`).

# Test
```bash
/jffs/scripts/dns-fallback  # Run once to test (may change NVRAM and restart dnsmasq)
nvram get dhcp_dns1_x       # Check current
cat /jffs/dns-fallback.state
```

Quick rollback:

```bash
cp /jffs/scripts/dns-fallback.bak.<timestamp> /jffs/scripts/dns-fallback
chmod +x /jffs/scripts/dns-fallback
rm -f /jffs/dns-fallback.state
```

This repository uses safe generic defaults. Use local overrides for router-specific IPs, DNS servers, and notification hooks.

# Why?
Prevents total DNS outage if resolver crashes. Better than complex subiface hacks. Battle-tested with Docker Pi-hole + Unbound DNSSEC issues.

# License:
MIT

# Inspired:
[SNBForums dns fallback](https://www.snbforums.com/threads/simple-script-for-dns-fallback.88556/)
