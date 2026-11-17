# Apache Smart Tuner

Apache Smart Tuner is a Bash-based capacity planner for Apache HTTP Server that favors safety and clarity for busy shared hosting stacks (cPanel/WHM and generic Apache alike). The tuner inspects live system resources, applies tiered guardrails, and produces right-sized MPM settings that respect PHP-heavy mixed workloads without risking RAM exhaustion.

## What it does
- **Resource- and PHP-aware tuning:** Uses total RAM, CPU cores, observed httpd RSS, and PHP-friendly CPU caps to calculate safe concurrency limits per MPM.
- **Tiered profiles:** LOW, LOW-MID, MID, MID-HIGH, and HIGH profiles automatically cap ServerLimit/MaxRequestWorkers (or threads) for a wide range of footprints.
- **Cautious floors and caps:** Enforces sensible minimums (128 threads for event/worker; 128 prefork workers only when the RAM budget can hold them, never below 16) while honoring tier ceilings (128/256/1024/2048/4096 by tier) to avoid runaway prefork deployments.
- **Clear visibility:** Prints current versus proposed settings, emits an annotated config block, and now supports machine-readable JSON output for automation.
- **Log health scan:** Reviews the last 24 hours of Apache error logs (or the entire file if newer) for worker saturation, ServerLimit warnings, common crash/timeout patterns, and restart frequency with checklist-style output.
- **Safe application path:** Creates timestamped backups, scrubs legacy MPM blocks, validates with `configtest`, and reloads (or optionally skips reload) only after a clean validation.
- **cPanel/WHM integration:** Prefers `pre_virtualhost_global.conf` when present, falls back to `pre_main_global.conf`, and triggers `/scripts/rebuildhttpdconf` when applying changes.
- **Stack-aware RAM plan:** Detects PHP-FPM pools and a local MySQL/MariaDB, fits Apache, PHP-FPM and MySQL into one RAM budget, and recommends `pm.max_children`, `innodb_buffer_pool_size` and `max_connections`. These are recommend-only, and Apache remains the only thing applied.
- **Operator control:** Override the RAM budget percentage when you need a custom envelope and redirect or disable on-disk logging per run.

## Usage
Download or copy `apache-tuner` to a host where Apache is installed and make it executable:

```bash
chmod +x apache-tuner
```

### Modes
- **Analyze (default):** Inspect RAM/CPU/process data and print the proposed tuning block.
- **Locate:** Show which Apache config files currently define MPM or concurrency directives.
- **Apply:** Safely write the tuned block to the appropriate include file (root required).

### Flags
```
./apache-tuner [--analyze] [--locate] [--apply] [--json] [--batch] [--export <path>] [--no-reload] [--mpm <type>] [--budget <0.x>] [--mysql-share <0.x>] [--no-stack] [--log-file <path|none>] [--version]
```
- `--json`: Return analysis/apply details as JSON (for monitoring/CM pipelines). Not supported with locate mode.
- `--batch`: Emit analysis/apply details as a single-line bash-friendly `key=value` string for fleet-wide collection.
- `--export`: Write only the tuned config block to a file without editing Apache configs.
- `--no-reload`: When combined with `--apply`, write the config but skip the reload/restart.
- `--mpm`: Force the assumed MPM type (`prefork`, `worker`, or `event`) when detection is blocked.
- `--version`: Print the current Apache Smart Tuner release.
- `--budget`: Override the tier-derived Apache RAM budget with a decimal percentage (e.g., `0.40`).
- `--log-file`: Send tuner logs to a custom file or `none` to disable filesystem logging.
- `--mysql-share`: Override the share of RAM planned for a local MySQL/MariaDB (default 0.15–0.30 by tier).
- `--no-stack`: Skip PHP-FPM/MySQL detection and size Apache from the tier budget alone (pre-1.22 behaviour).

### Examples
Run a standard analysis:

```bash
./apache-tuner
```

Produce JSON suitable for automation or dashboards:

```bash
./apache-tuner --json
```

Emit a single-line batch string for fleet-wide collection pipelines:

```bash
./apache-tuner --batch
```

Apply the recommended settings while emitting only JSON for automation pipelines:

```bash
sudo ./apache-tuner --apply --json
```

Export only the recommended block for review or CM pipelines:

```bash
./apache-tuner --export /tmp/apache-smart-tuner.conf
```

Inspect where existing MPM settings live:

```bash
./apache-tuner --locate
```

Apply tuned values on a cPanel server while deferring the reload to a maintenance window:

```bash
sudo ./apache-tuner --apply --no-reload
```

## What to expect
Text analysis output includes environment detection, an error log & stability review, current vs proposed values, and the ready-to-paste block:

```
------------------------------------------------
 Apache Smart Tuner Analysis
------------------------------------------------
Detected MPM:           prefork (source: binary -V)
Tier:                   LOW-MID
Total RAM:              8192 MB
CPU Cores:              4
Apache running:         yes (180 processes observed)
Apache RAM Budget:      2867 MB (0.35 of total; source: tier)
Avg httpd proc size:    10 MB
------------------------------------------------
 Error log & stability review
------------------------------------------------
  [--] Log source          /var/log/apache2/error.log (last 15000 lines)
  [OK] Worker saturation   No MaxRequestWorkers saturation observed
  [OK] ServerLimit warnings No ServerLimit notices detected
  [OK] Process crashes/segfaults No Process crashes/segfaults observed in sampled lines.
  [--] Slow CGI/PHP timeouts 3 entries related to Slow CGI/PHP timeouts.
  [OK] Client denials      No Client denials observed in sampled lines.
  [--] Apache restarts     1 restart events; max 1/day across 1 day(s).
------------------------------------------------
  Current vs Proposed (prefork) (current => proposed):
    ServerLimit           256          => 272
  MaxRequestWorkers     256          => 256
  StartServers          5            => 2
  MinSpareServers       5            => 2
  MaxSpareServers       10           => 8
  MaxConnectionsPerChild 0           => 4000
------------------------------------------------
# BEGIN APACHE_SMART_TUNER
# Apache Smart Tuner v1.20.0 (Tier: LOW-MID, MPM: prefork)
Timeout 120
KeepAlive On
MaxKeepAliveRequests 100
KeepAliveTimeout 5

<IfModule prefork.c>
    ServerLimit            272
    MaxRequestWorkers      256
    StartServers           2
    MinSpareServers        2
    MaxSpareServers        8
    MaxConnectionsPerChild 4000
</IfModule>
# END APACHE_SMART_TUNER
------------------------------------------------
```

JSON output mirrors the same data structure for pipelines:

```json
{
  "version": "1.20.0",
  "mode": "analyze",
  "mpm": "prefork",
  "mpm_source": "binary -V",
  "tier": "LOW-MID",
  "total_ram_mb": 8192,
  "cpu_cores": 4,
  "apache_running": true,
  "observed_processes": 180,
  "avg_httpd_mb": 10,
  "apache_budget_mb": 2867,
  "apache_budget_pct": 0.35,
  "apache_budget_source": "tier",
  "log_file": "/var/log/apache-smart-tuner.log",
  "export_path": "",
  "recommended_block": "# BEGIN APACHE_SMART_TUNER\n# Apache Smart Tuner v1.20.0 (Tier: LOW-MID, MPM: prefork)\nTimeout 120\nKeepAlive On\nMaxKeepAliveRequests 100\nKeepAliveTimeout 5\n\n<IfModule prefork.c>\n    ServerLimit            272\n    MaxRequestWorkers      256\n    StartServers           2\n    MinSpareServers        2\n    MaxSpareServers        8\n    MaxConnectionsPerChild 4000\n</IfModule>\n# END APACHE_SMART_TUNER\n",
  "log_review": {
    "status": "ready",
    "message": "Analyzed the last 24 hours (15000 lines)",
    "error_log_path": "/var/log/apache2/error.log",
    "sampled_lines": 15000,
    "scoreboard_hits": 0,
    "server_limit_hits": 0,
    "restarts_total": 1,
    "restart_days_observed": 1,
    "restart_max_per_day": 1,
    "common_errors": {
      "segfaults": 0,
      "timeouts": 3,
      "denied": 0
    }
  },
  "current": {
    "ServerLimit": "256",
    "MaxRequestWorkers": "256",
    "ThreadsPerChild": "",
    "StartServers": "5",
    "MinSpareThreads": "",
    "MaxSpareThreads": "",
    "MinSpareServers": "5",
    "MaxSpareServers": "10",
    "MaxConnectionsPerChild": "0"
  },
  "recommended": {
    "ServerLimit": "272",
    "MaxRequestWorkers": "256",
    "ThreadsPerChild": "",
    "StartServers": "2",
    "MinSpareThreads": "",
    "MaxSpareThreads": "",
    "MinSpareServers": "2",
    "MaxSpareServers": "8",
    "MaxConnectionsPerChild": "4000"
  }
}
```

## PHP-FPM and MySQL/MariaDB awareness (1.22)
Apache rarely has the box to itself. Since 1.22 the tuner plans one shared RAM budget across the whole web stack. The PHP-FPM and MySQL/MariaDB parts are **recommend-only**: `--apply` still writes only the Apache block, and nothing else is edited or restarted.

**Memory plan.** Total RAM is split in this order:
1. **OS / page cache reserve.** 10–15% of RAM by tier, clamped to 512–4096 MB.
2. **MySQL/MariaDB.** When it runs locally, the reserve is the larger of its current RSS and the footprint it would need with the recommended settings.
3. **Apache.** The tier percentage is now a *ceiling*. When PHP-FPM is running and `mod_php` is not loaded, httpd children no longer carry PHP, so Apache gets 30% of the remaining web budget and PHP-FPM gets the rest. The Apache budget is only ever lowered, never raised. When that happens the source is reported as `shared-plan`.
4. **PHP-FPM.** Gets what is left after Apache's worst case, divided by the measured average worker size.

**PHP-FPM detection.**
- Running masters and pool workers are found from php-fpm process titles.
- Pool files are read from cPanel EA4 (`/opt/cpanel/ea-php*/root/etc/php-fpm.d`), RHEL/Remi, Debian and `/usr/local/etc` paths. The tuner counts pools, `pm` modes, the sum of `pm.max_children`, and pools without `pm.max_requests`.
- Worker size uses PSS when readable (so OPcache is not counted once per worker), otherwise RSS, with a 32 MB floor.
- Recent FPM error logs are scanned for `server reached pm.max_children` warnings per pool.
- Apache's loaded PHP modules (`mod_php`, `proxy_fcgi`, `suphp`, `lsapi`, `ruid2`) and cPanel's per-version handler map are reported.

**PHP-FPM recommendations.**
- A per-pool `pm.max_children`. When most pools are `ondemand`, idle sites hold no workers, so 3x overcommit across pools is allowed. Otherwise no overcommit is allowed.
- Warnings for pools that alone can exceed the FPM budget, pools hitting their limit in the logs, workers that never recycle, and many-pool servers still on `dynamic`/`static`.
- On cPanel the advice points to WHM's *System PHP-FPM Settings* (`pm_max_children` in `/var/cpanel/ApachePHPFPM/system_pool_defaults.yaml`), then `/scripts/php_fpm_config --rebuild`.

**MySQL/MariaDB.**
- Live variables are read with `mysql`, using root's `~/.my.cnf` on cPanel. When that is unavailable the tuner falls back to `my_print_defaults` plus server defaults.
- InnoDB data size is read from `information_schema`, bounded server-side to 10 seconds (`max_statement_time` / `MAX_EXECUTION_TIME`).
- The planned MySQL share of RAM is 15/20/25/25/30% by tier. Override it with `--mysql-share`.
- Recommendations:
  - `innodb_buffer_pool_size` fitted to that share, and capped at InnoDB data + 25%.
  - `max_connections`, raised when `Max_used_connections` reaches 90% of it, or lowered when the worst case would blow the plan.
  - Disabling the query cache.
  - Aligning `tmp_table_size` with `max_heap_table_size`.
  - A warning when stats cover less than 24 hours of uptime.

The text report prints the memory plan, a PHP-FPM section, a MySQL section with a suggested `[mysqld]` snippet, and a list of stack recommendations. JSON adds `memory_plan`, `php_fpm`, `mysql` and `stack_recommendations`. Batch adds `fpm_*`, `mysql_*` and plan fields.

Additional flags:
- `--mysql-share <0.x>`: share of RAM planned for MySQL/MariaDB (default 0.15–0.30 by tier).
- `--no-stack`: skip PHP-FPM/MySQL detection and use the tier Apache budget alone (pre-1.22 behaviour).

Run as root for PSS measurements and live MySQL variables.

## ServerLimit buffer and "scoreboard full" protection
Apache keeps old children alive long enough to finish in-flight requests during graceful restarts or MaxConnectionsPerChild recycling. If ServerLimit is set to the exact active process count, those lingering children can temporarily exhaust the scoreboard and block Apache from spawning the replacements, resulting in brief "Scoreboard is full" errors and dropped requests. Apache Smart Tuner pads ServerLimit by a small buffer (5% with an 8-slot minimum) above the calculated MaxRequestWorkers/process count so new workers can come online while old ones drain, without raising steady-state concurrency.

## Safety details
- **Backups:** Target include files are copied with a `.bk-YYYYmmddHHMMSS` suffix before any write.
- **Scrub legacy blocks:** Existing `prefork.c`, `worker.c`, `event.c`, and `mpm_*` blocks inside the target include are removed before the new tuner block is written.
- **Validation:** Runs `apachectl`/`apache2ctl`/`httpd -t` before and after apply; failures trigger an automatic rollback.
- **Reload control:** When Apache is running, the tuner reloads automatically unless `--no-reload` is set; if Apache is stopped, it writes the config and logs the pending start.

## Versioning
The project follows semantic versioning. The current release is **1.22.0**.

## License
GPL-3.0-or-later. Authored by **Ryan MacDonald <ryan@rfxn.com>**.
