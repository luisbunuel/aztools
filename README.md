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

All tenants share the default Azure CLI profile (`~/.azure`): `az login --tenant` adds each tenant's subscriptions to it, and every `az` call in the script passes `--subscription`. The exception is the tunnel token: the bastion extension takes it from the default subscription, so the script runs `az account set` right before connecting. `az` reads the default subscription once, when it starts, so sessions and tunnels already open are not affected. The tenant picker shows ✔ (green) for tenants with a currently valid session and ✖ (red) otherwise.

Tenants/servers are defined in two arrays at the top of the script (`TENANT_MAP`, `AZURE_RESOURCES`) — add or remove a server by editing one line there, no need to touch the rest of the script.

Before first use, edit the `azuser` variable in the script to your own username, and make sure you've run `az login --tenant <id>` (or let the script do it) for each tenant you plan to use.

```
./azbastion        # SSH session
./azbastion -t     # tunnel to port 22 on the first free local port from 50022, for scp/sftp
```

**azbastion.old** — the original `dialog`/ncurses-based version, kept for reference. Same tenants/servers, but every server required its own hardcoded `dialog` menu + `case` block, making it tedious to maintain. Superseded by `azbastion`.

---

### azmaintenance

Enables or disables Azure Monitor alert-processing "maintenance mode" rules for servers and for web URLs monitored by App Insights web tests. It searches every subscription of the tenants listed in the `tenants` array of the script (UNICC, UN Women, UNDP and UNCTAD), all in the default Azure CLI profile.

```
azmaintenance [-w url ...] [-v server ...] -a action [-s start_date] [-f finish_date] [-d description] [-F]

  -w   Web URL (repeatable). Finds the "<web test> - Web Maintenance Mode" rule of the
       App Insights web test for that host and updates it in place, keeping its Terraform tags.
       If the test has no such rule (most UN Women tests), enable creates it
  -v   Server name (repeatable). Finds its "<server> - Server Maintenance Mode" rule and
       updates it in place. A missing server rule is not created
  -a   Action: enable | disable
  -s   Maintenance start date (UTC, "yyyy-mm-dd hh:mm:ss"). Default: now
  -f   Maintenance finish date (UTC, "yyyy-mm-dd hh:mm:ss"). Default: start + 1 hour
  -d   Description (enable only)
  -F   Overwrite the window of webs or servers already in maintenance. Without it, enable
       skips a rule that is enabled now or scheduled, and shows its current window
```

Examples:

```
azmaintenance -v UNWMEUSLMSP01 -v UNWMEUSLMSD01 -a enable -s "2026-09-14 12:00:00" -f "2026-09-14 13:00:00" -d "Monthly patching"
azmaintenance -v UNWMEUSLMSP01 -a disable
azmaintenance -w https://bangkok.ohchr.org -w cambodia.ohchr.org -a enable -f "2026-11-03 10:00:00" -d "Until PHP is patched"
azmaintenance -w bangkok.ohchr.org -a disable
```

Lists the current state of each rule, gives a 10-second countdown before applying the change, and prints a per-rule success/error status.
