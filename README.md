## aztools

Small collection of Bash scripts to work with Azure Bastion tunnels and Azure Monitor maintenance windows across multiple tenants/subscriptions.

### Prerequisites

```
- Azure CLI (az)
- fzf              # used by azbastion
- dialog           # only needed for the legacy azbastion.old
- wslu             # only on WSL
```

The `az network bastion ssh` extension (`ssh`) is installed automatically by the scripts on first run if missing.

---

### azbastion

Interactive tunnel/SSH connection through Azure Bastion, across several tenants.

Flow: pick a **tenant** → pick an **environment** (Production/Test) → pick a **server** → it logs into that tenant (if needed) and opens the connection with `az network bastion ssh`.

Each tenant has its own isolated Azure CLI login, stored in a separate `AZURE_CONFIG_DIR` (`~/.azure_IIIM`, `~/.azure_UNDP`, `~/.azure_UNDPBIO`, `~/.azure_WOMEN`, `~/.azure_UNICC`), so logging into one tenant never affects the others' sessions. The tenant picker shows ✔ (green) for tenants with a currently valid session and ✖ (red) otherwise.

Tenants/servers are defined in two arrays at the top of the script (`TENANT_MAP`, `AZURE_RESOURCES`) — add or remove a server by editing one line there, no need to touch the rest of the script.

Before first use, edit the `azuser` variable in the script to your own username, and make sure you've run `az login` (or let the script do it) for each `AZURE_CONFIG_DIR` you plan to use.

```
./azbastion
```

**azbastion.old** — the original `dialog`/ncurses-based version, kept for reference. Same tenants/servers, but every server required its own hardcoded `dialog` menu + `case` block, making it tedious to maintain. Superseded by `azbastion`.

---

### azmaintenance

Enables or disables Azure Monitor alert-processing "maintenance mode" rules for a set of servers, per organization/environment, or for web URLs monitored by App Insights web tests (UNICC Cloud Web Hosting Monitoring, UN Women and UNDP).

```
azmaintenance -o organization -e environment -a action [-s start_date] [-f finish_date] [-d description]
azmaintenance -w url [-w url ...] -a action [-s start_date] [-f finish_date] [-d description] [-F]

  -o   Organization: IIIM | IIMM
  -e   Environment: prod | test
  -w   Web URL (repeatable). Finds the "<web test> - Web Maintenance Mode" rule of the
       App Insights web test for that host and updates it in place, keeping its Terraform tags.
       If the test has no such rule (most UN Women tests), enable creates it
  -a   Action: enable | disable
  -s   Maintenance start date (UTC, "yyyy-mm-dd hh:mm:ss"). Default: now
  -f   Maintenance finish date (UTC, "yyyy-mm-dd hh:mm:ss"). Default: start + 1 hour
  -d   Description (enable only)
  -F   With -w, overwrite the window of webs already in maintenance. Without it, enable
       skips a web whose rule is enabled now or scheduled, and shows its current window
```

Examples:

```
azmaintenance -o IIMM -e prod -a enable -s "2026-09-14 12:00:00" -f "2026-09-14 13:00:00" -d "Monthly patching"
azmaintenance -w https://bangkok.ohchr.org -w cambodia.ohchr.org -a enable -f "2026-11-03 10:00:00" -d "Until PHP is patched"
azmaintenance -w bangkok.ohchr.org -a disable
```

Gives a 10-second countdown before applying the change, and prints a per-server (or per-rule) success/error status.
