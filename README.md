# DEPLOYMENT GUIDE — SERVER HEALTH MONITORING

> Generic playbook, derived from a real implementation already validated in production. Serves as a reference for deploying this type of monitoring on **any Linux server** hosting web applications — it is not specific to any project, domain, or organization. Wherever an environment-specific value would go (domain, IP, app name, OS user), it appears as a `<PLACEHOLDER>`.

## 0. Philosophy behind this guide

Before the technical steps, the principles that guided the original implementation and that hold for any server:

1. **Never assume — validate with real data.** Metric names, component names, log formats, software versions: all of this varies between versions and installations. Practically every stage in this guide has a verification step before the next configuration step. Skipping verification is the single most common cause of rework.
2. **Build in layers, not all at once.** When assembling a new pipeline (e.g., collection → filter → transform → export), validate each layer in isolation (with a debug/log exporter) before adding the next one. A 5-stage pipeline that fails is far harder to debug than 5 single-stage pipelines tested in sequence.
3. **Calibrate alert thresholds against a real baseline, never against a "round" guessed number.** Measure the resource's current state before deciding what counts as "normal" vs. "anomalous."
4. **Test the notification path end-to-end before depending on it.** An alert that was never actually tested is an assumption, not a guarantee.
5. **Components marked experimental/beta/alpha can have real bugs, not just documented limitations.** When the behavior doesn't make sense (e.g., a counter that decreases), don't paper over it with a superficial fix — investigate whether the component is trustworthy for your use case, and have a plan B.
6. **Prefer the most technology-agnostic solution when the environment has heterogeneous stacks.** If multiple applications with different languages/frameworks sit behind the same reverse proxy, instrumenting the proxy costs one implementation; instrumenting each application costs N implementations.

---

## 1. Reference architecture

```
                    ┌──────────────────┐
   outside the      │ External uptime   │ → HTTP(s) uptime + SSL expiry
   server       →   │  check service    │    check for each hosted site
                    └─────────┬─────────┘
                              │ webhook
                              ▼
   inside the server:  ┌────────────┐        ┌──────────────────┐
   Telemetry         → │Observability│        │   Chat channel    │
   collector            │  backend   │ ─────► │     (alerts)       │
   (metrics + logs)  → │ (OTLP)     │ webhook│                   │
                        └────────────┘        └──────────────────┘
   Process manager (apps) → local log rotation/compression
```

Three independent fronts that together cover server health without requiring any code changes in the hosted applications:

| Front | What it covers | Where it runs |
|---|---|---|
| **Uptime + SSL** | Is the site responding? Is the certificate close to expiring? | Outside the server (third-party service, outside-in view) |
| **Host resources** | CPU, RAM, disk, network, swap | Inside the server, via a telemetry collector |
| **HTTP traffic (optional/advanced)** | Which routes are used most, which generate the most errors | Inside the server, intercepting at the reverse proxy |
| **Application logs** | Logs don't grow unbounded until disk fills up | Inside the server, via automatic rotation |

This document uses as its reference implementation (already validated): **OpenTelemetry Collector** (collection) + **an OTLP-compatible observability backend** (storage/visualization/alerting) + **an external uptime check service** + **a chat webhook** (notification) + **the applications' process manager** (log rotation). Swap in whatever your environment already uses for the product names — the logic and validation sequence hold regardless.

---

## 2. Prerequisites

Before starting, have ready:

- SSH access with sudo on the server.
- An account with the chosen observability backend, with ingestion credentials (endpoint + token/API key).
- A chat-channel webhook (Discord, Slack, etc.) to receive alerts — ideally dedicated to infrastructure alerts, separate from any existing application-log channel.
- An account with the chosen external uptime check service.
- A list of people who should have read/write access to the observability backend (to restrict access later).
- The public URLs of every site/application hosted on the server.

---

## 3. Part 1 — Host resource observability

### 3.1 Install the Collector

Resolve the latest version automatically instead of hardcoding it (avoids broken links and version typos):

```bash
cd /tmp
VERSION=$(curl -s https://api.github.com/repos/open-telemetry/opentelemetry-collector-releases/releases/latest | grep -Po '"tag_name": "v\K[^"]*')
curl --proto '=https' --tlsv1.2 -fOL \
  "https://github.com/open-telemetry/opentelemetry-collector-releases/releases/download/v${VERSION}/otelcol-contrib_${VERSION}_linux_<ARCHITECTURE>.deb"
sudo apt install -y "./otelcol-contrib_${VERSION}_linux_<ARCHITECTURE>.deb"
```

`<ARCHITECTURE>` is `amd64` or `arm64` — confirm first with `uname -m` (`x86_64` → `amd64`; `aarch64` → `arm64`).

> **Ignore any suggestion to instrument the applications themselves** (language SDKs, `OTEL_EXPORTER_OTLP_ENDPOINT` env vars pointing at a local receiver) that your observability backend's onboarding assistant may suggest at this stage. That's for application telemetry (APM/tracing), a separate project — here the goal is only host health.

### 3.2 Export credentials — never in plain text in the main config

The package creates an environment file referenced by the systemd unit (confirm with `systemctl cat otelcol-contrib | grep -i EnvironmentFile`). Add the credentials there, not in the YAML config:

```bash
AUTH=$(echo -n "<INSTANCE_ID>:<API_KEY>" | base64 -w0)

sudo tee -a /etc/otelcol-contrib/otelcol-contrib.conf > /dev/null <<EOF
OTEL_EXPORTER_ENDPOINT="<endpoint_provided_by_the_backend>"
OTEL_EXPORTER_AUTH_HEADER="Basic $AUTH"
EOF

unset AUTH
sudo chown root:root /etc/otelcol-contrib/otelcol-contrib.conf
sudo chmod 600 /etc/otelcol-contrib/otelcol-contrib.conf
```

> **Common mistake #1 — heredoc escaping.** If you use `$VAR` inside an unquoted heredoc (`<<EOF`, no quotes), it's expanded at write time — correct. If you accidentally type `\$VAR` (backslash before the `$`), the file receives the literal text `$VAR`, not the value — and authentication fails silently until you inspect the generated file. After any heredoc with a variable, **check the file's actual contents** (`cat`) before assuming it worked.

> **Common mistake #2 — unsubstituted placeholder.** When copying a command containing `<something>`, it's easy to run it literally with the text `<something>` instead of the real value — this produces an invalid credential that also fails silently (an authentication error, not a syntax error). Where possible, prefer building the command with the real values already in place rather than leaving the placeholder for a last-minute manual substitution.

### 3.3 Configure the host metric scrapers

Start with the essentials (CPU, memory, disk) and add the rest as needed:

```yaml
receivers:
  host_metrics:
    collection_interval: 30s
    scrapers:
      cpu:
        metrics:
          system.cpu.utilization:
            enabled: true
      memory:
        metrics:
          system.memory.utilization:
            enabled: true
      load: {}
      filesystem:
        metrics:
          system.filesystem.utilization:
            enabled: true
        exclude_mount_points:
          match_type: regexp
          mount_points:
            - "^/(dev|proc|sys|run|snap|var/lib/docker).*"
        exclude_fs_types:
          match_type: strict
          fs_types: ["tmpfs", "devtmpfs", "squashfs", "overlay"]
      disk: {}

processors:
  batch: {}
  resource_detection:
    detectors: [system]
    system:
      hostname_sources: [os]

exporters:
  otlp_http:
    endpoint: "${env:OTEL_EXPORTER_ENDPOINT}"
    headers:
      authorization: "${env:OTEL_EXPORTER_AUTH_HEADER}"

service:
  pipelines:
    metrics:
      receivers: [host_metrics]
      processors: [resource_detection, batch]
      exporters: [otlp_http]
```

**Points of attention, validated in practice:**

- **"Utilization" metrics (0–1 ratio) are usually disabled by default.** The `cpu`/`memory`/`filesystem`/`paging` scrapers tend to expose only raw metrics (cumulative CPU time, memory bytes) unless you explicitly enable the matching `*.utilization` metric, as in the example above. Without this, you'd have to compute a percentage from raw counters in the query — more work and more room for error.
- **Exclude virtual filesystems/devices from the `filesystem` scraper**, or loop devices (common on systems using Snap/Flatpak), `tmpfs`, `overlay`, etc. will be included in the disk-usage calculation and skew the real percentage — which breaks the calibration of any disk alert based on that metric.
- **The `disk` scraper has its own filter, independent of the `filesystem` one.** It identifies by block device (`sda`, `dm-0`, `loop0`...), not by mount point — the same loop devices will show up here again and need their own exclusion (`device!~"loop.*"` in the query, since this scraper doesn't have a native exclusion filter in every version).
- **Network**: exclude the loopback interface (`lo`) from traffic queries — it always reflects the machine's own internal traffic, symmetric between inbound and outbound, and doesn't represent real network traffic.
- **Monitoring specific processes (e.g., a database, an application runtime) requires a separate `process` scraper**, with two gotchas:
  - Without a filter, it creates one metric series **per process on the entire system** (systemd, sshd, cron, etc.) — always restrict with `include`/`exclude` by name, listing only the processes that matter.
  - The Collector normally runs as an unprivileged system user — to read metrics for processes owned by **other** users (e.g., the database runs as a different OS user), the binary needs a specific capability granted: `sudo setcap cap_sys_ptrace,cap_dac_read_search+ep /usr/bin/otelcol-contrib`. **This must be reapplied every time the package is upgraded** (the upgrade replaces the binary and resets the capability).
- **Component names have old and new aliases** (e.g., `hostmetrics` vs. `host_metrics`) — newer versions usually still accept the old name but log a deprecation warning; prefer the current name.
- **Not every documented metric key exists in every version.** Before relying on an obscure configuration key (e.g., a recently added optional metric), validate it with the tool's own validation command (section 3.4) — an "invalid key" error is much better discovered there than in production.

### 3.4 Start the service and validate before trusting it

**Always validate the syntax with environment variables loaded before restarting the service** — running the validator without loading the `.conf` file will complain about a "missing endpoint" even with correct config, because it has no way to read a `600 root:root` file without being root:

```bash
sudo bash -c 'set -a; source /etc/otelcol-contrib/otelcol-contrib.conf; set +a; /usr/bin/otelcol-contrib validate --config=/etc/otelcol-contrib/config.yaml'
echo "exit code: $?"
```

Exit code `0` and no output = valid. Only then:

```bash
sudo systemctl enable --now otelcol-contrib
sudo systemctl status otelcol-contrib
sudo journalctl -u otelcol-contrib --since "2 minutes ago" -o cat --no-pager | grep -iE "warn|error"
```

A common error at this stage: `401`/`403` on export — this is almost always the escaping/placeholder issue from section 3.2, not a network problem. Check the credentials file before suspecting anything else.

### 3.5 Confirm visibility in the observability backend

Don't assume the final metric name — your backend's ingestion gateway normalizes OTel names to its own internal convention (e.g., `_ratio`, `_total`, `_bytes` suffixes). Use the backend's ad-hoc exploration tool (e.g., "Explore") to:
1. Confirm the metrics show up, with their real names and labels (not the names predicted from the OTel spec).
2. Compare the reported value against what you see locally on the server (`df -h`, `free -h`, `top`) — if it doesn't match, the most likely cause is a poorly calibrated exclusion filter (section 3.3).
3. Build a minimal dashboard with the essential panels (CPU, RAM, disk), using the **confirmed** names, not the predicted ones.

---

## 4. Part 2 — Threshold alerts

### 4.1 Calibrate against the real baseline

Before picking a threshold number, measure the resource's actual current state (via the dashboard from section 3.5, or directly on the server). A "round" threshold with no relation to the environment's real baseline (e.g., 90% disk on a server that already chronically runs at 85%) is useless — it either never fires, or fires constantly.

### 4.2 Alert rule — two tiers

Recommended pattern: two rules (warning/critical) on the same metric, or a single rule with different routes per severity, depending on what the alerting tool supports:

- **Warning**: lower threshold, "start paying attention."
- **Critical**: higher threshold, "act now."
- **Pending period ("for")**: a few minutes, so it doesn't fire on a single transient spike in one evaluation cycle.

If the monitored resource can legitimately have no data at all during a period (common for rare-event counts, like errors — see section 7.7), explicitly configure the rule's **"no data"** behavior: by default, many alerting tools treat "no data" as "fire," which creates a **permanent false positive** when "no data" actually means "everything's fine, nothing happened." Set this to "Normal"/"OK" when that's the case.

Also configure the **"keep firing for"** period after the condition clears, if available — this prevents the notification channel from flapping between "firing"/"resolved" repeatedly when the condition oscillates near the threshold.

### 4.3 Notification channel

1. Create a contact point pointing at the chosen chat channel's webhook.
2. Test delivery via the tool's native test button, if it has one — this only confirms delivery, not the full rule flow (see 4.4).
3. Wire up the notification policy (the default policy, if this is the only kind of alert so far; a policy with a dedicated route/matcher, if you need to split by severity/team).

### 4.4 End-to-end test — don't skip this step

An alert that was never tested is an assumption. Two valid ways to test it:

- **Organic validation**: if the resource genuinely crosses the threshold during implementation (this happened, for example, with disk usage during a log cleanup/rotation), that itself counts as a real test — confirm that both the firing notification **and** the resolved notification arrived, with the annotation template values correctly filled in (template variables in annotations are only populated once the rule is actually in a "Firing" state — in the rule editor, before it fires, they show up as the raw template text; that's expected, not a bug).
- **Controlled manual test**: temporarily lower the threshold to a value below the current state, force it to fire, confirm the notification, **restore the real value**, and confirm the resolved notification too.

---

## 5. Part 3 — External monitoring (uptime + SSL)

### 5.1 HTTP(s) uptime

For each public site/application, an HTTP(s) monitor in the chosen external uptime check service, checking from outside the server (important: this catches problems the internal monitoring can't, such as a misconfigured firewall, DNS issues, or the server being unreachable entirely).

### 5.2 SSL certificate expiry

If the uptime check service's free tier doesn't include this (common — free-tier features change over time), a no-cost alternative: a cron script on the server itself, using `openssl` to check each certificate's validity and post to the same notification channel if it's close to expiring:

```bash
#!/usr/bin/env bash
set -euo pipefail
source /etc/<config_dir>/webhook.env

DOMAINS=("<domain1>" "<domain2>" "<domain3>")
ALERT_DAYS=" 30 14 7 3 1 0 "   # only alert on specific milestones, avoids a daily repeat notification

send_notification() {
  local payload
  payload=$(jq -n --arg title "$1" --arg desc "$2" '{embeds: [{title: $title, description: $desc}]}')
  curl -s -H "Content-Type: application/json" -d "$payload" "$WEBHOOK_URL" > /dev/null
}

for domain in "${DOMAINS[@]}"; do
  expiry_date=$(echo | openssl s_client -servername "$domain" -connect "$domain:443" 2>/dev/null \
    | openssl x509 -noout -enddate 2>/dev/null | cut -d= -f2 || true)
  [ -z "$expiry_date" ] && { send_notification "SSL check failed" "Could not retrieve the certificate for $domain"; continue; }
  days_left=$(( ($(date -d "$expiry_date" +%s) - $(date +%s)) / 86400 ))
  if [ "$days_left" -lt 0 ] || [[ "$ALERT_DAYS" == *" $days_left "* ]]; then
    send_notification "SSL certificate expiring" "$domain expires in $days_left days ($expiry_date)"
  fi
done
```

Schedule it via `crontab -e` (e.g., `0 8 * * * /path/to/script.sh`), keeping the webhook URL in a separate `600 root:root` environment file, never in the script itself.

---

## 6. Part 4 — Application log management

Unrotated application logs are one of the most common causes of a full disk on production servers. If applications are managed by a process manager (e.g., PM2), use its native log-rotation module; otherwise, the operating system's own `logrotate`.

```bash
<process_manager> install <logrotate_module>
<process_manager> set <module>:max_size 10M
<process_manager> set <module>:retain 14
<process_manager> set <module>:compress true
<process_manager> save
```

**Don't assume automatic rotation/compression works — validate with a real file.** After installing:
1. Confirm there's at least one log file bigger than the configured `max_size` (otherwise the test won't trigger anything).
2. Wait for one cycle of the module's internal worker (usually configurable, seconds to minutes) and check whether the file got renamed/rotated.
3. **Separately confirm that compression (`.gz`) actually happened** — rotation and compression can be internally distinct steps, and one of them can fail silently (no error line in the module's own log) even with the option enabled. If rotation works but compression doesn't, the original problem (disk filling up) wasn't solved, just disguised — you'll accumulate large, uncompressed files. If this happens, compress manually (`gzip <file>`) as an immediate mitigation and track it as a bug to follow up on, not as resolved.

---

## 7. Part 5 (advanced/optional) — HTTP traffic observability via reverse proxy

This part answers a different question than the previous ones: not "is the server up," but **"what's being used, and where are the errors."** It's more involved and has more failure points — treat it as an enhancement, not as part of baseline monitoring.

### 7.1 Why instrument the proxy, not each application

If the server hosts multiple applications with different backend stacks, an application-level middleware would need to be reimplemented for each stack. The reverse proxy (e.g., Nginx) in front of all of them is the only truly common point, independent of language/framework — a deliberate choice to keep the solution technology-agnostic while reusing the same collection infrastructure already installed in Part 1.

### 7.2 Structured logging per site

Change (or confirm) the reverse proxy's `log_format` to **JSON**, with a dedicated log file per vhost/site (not a single shared global file — that makes it harder to separate by site later). Minimum fields: timestamp, HTTP method, URI (without query string), status, response time, host.

```nginx
log_format access_json escape=json '{"time":"$time_iso8601","method":"$request_method","uri":"$uri","status":$status,"request_time":$request_time,"host":"$host"}';
```

For each vhost: `access_log /var/log/nginx/<site>-access.log access_json;`

> **Always validate syntax before reloading a production proxy** (`nginx -t` or equivalent) — and edit one vhost at a time, validating between each edit, not all at once. This is live production traffic; an undetected syntax error could take every site down on reload.

### 7.3 Cardinality control — validate before implementing

"Most-used route" needs to mean the route's **pattern** (e.g., `/api/resources/:id`), not the literal URL with an identifier embedded in it — otherwise every distinct identifier (a database ID, a UUID) creates a new data series, and cardinality explodes.

**Before building any collection pipeline**, validate your normalization strategy against a real sample of production logs, with a simple, throwaway script (any scripting language works):
1. Extract method + path from each real log line.
2. Apply the planned normalization (regex replacing ID-looking segments with a placeholder).
3. Count how many **distinct** route patterns remain.

If the number of distinct patterns is still very high even after normalization, the most likely cause isn't the normalization itself, but **traffic that isn't from the application**: bots/scanners probing hundreds of random paths (e.g., attempts to find popular login endpoints, `/.env`, `/admin`), and single-page-application front-end routes that don't represent real business API calls. The fix is usually simpler than tweaking the regex: **filter by a prefix that reliably identifies business API routes** (e.g., every API endpoint lives under `/api/` or `/backend/`) before even normalizing — this eliminates most of the noise in one shot.

### 7.4 Collection pipeline: read, filter, normalize

Reusing the same Collector from Part 1, built **in layers, each validated in isolation** with a debug exporter (which just writes the processed data to the Collector's own log, without sending it anywhere) before adding the next one:

1. **File-tailing receiver** (tails the log files, parses JSON per line).
2. **Filter processor** (keeps only lines matching the business-API prefix decided in section 7.3).
3. **Transform processor** (normalizes the path: replaces ID-shaped segments — numeric, fixed-length hexadecimal, UUID — with a placeholder).

At each layer, generate real test traffic (one request of each type you want to validate) and check the Collector's own log to see whether the result is what you expect — **don't stack all 3 layers at once only to later discover which one is wrong.**

> **Be careful testing against concurrent real traffic**: a production server has real user traffic happening all the time. A short observation window can capture both your test and real concurrent traffic, making it hard to isolate cause and effect. Prefer testing against a rare/test-exclusive path, not a popular application path.

### 7.5 Pre-aggregated metric vs. raw log — a lesson learned

There's a temptation to have the Collector itself turn these filtered/normalized log lines directly into a count metric (a "connector" that sums occurrences and produces a counter). **Test this behavior carefully before trusting it**: counters need a well-defined accumulation semantic (an ever-increasing total since the beginning, never decreasing) to be compatible with most metrics backends. If the component used to generate the metric is marked experimental/beta/alpha in the tool's documentation, explicitly validate that the reported value **never decreases** across repeated test requests — a value that fluctuates or decreases is a sign of a real implementation bug, not a configuration issue, and shouldn't be "fixed" by merely suppressing the backend's validation error (that masks the symptom without fixing the cause, silently delivering wrong data).

**A more robust alternative, if your observability backend supports log storage (e.g., Loki and similar) in addition to metrics**: export the already filtered/normalized lines as **structured logs** (not as a pre-computed metric), and do the counting **at query time** (e.g., with a count-over-time function, directly in the dashboard). This eliminates the accumulation-semantics problem entirely — each log line is a discrete event, with no cumulative state involved.

**A query-syntax gotcha specific to this path**: fields that arrive as per-line log metadata (rather than an indexed "label" of the whole log stream) usually need to be filtered **after** the main stream selector, not inside it — mixing the two up is a common syntax mistake that results in "no data" with no clear error message.

### 7.6 Usage and error dashboards

With the data available (metric or log), two panels per site/application:

- **Top N most-used routes**: a ranking — compute it as a snapshot of the visible time range (not recalculated at every point along the timeline, or different routes may enter/leave the ranking at different points in time and the panel ends up showing more items than the configured N).
- **Error rate by route**: same logic, filtered by error status.

For **error rate as a percentage over time**, use a **fixed aggregation window** (e.g., the last 10 minutes), not a window that scales with the dashboard's zoom level — with low traffic, a short window may contain only 1-2 requests total, and if by coincidence all of them are errors (including an error generated by your own manual test), the panel shows "100% error" in a way that's mathematically correct but statistically meaningless. A larger, fixed window dilutes this small-sample noise.

### 7.7 Error alerting — design decisions

- **Alert on server errors (5xx), not client errors (4xx).** Client errors (unauthenticated, resource not found) are expected day-to-day traffic for any internet-facing application — including traffic generated by bots/scanners. Using that as an alert trigger produces constant noise. A server error is always a sign of a real backend problem.
- **Alert on an absolute count over a period, not a percentage.** Same small-sample problem as section 7.6 — 1 error out of 1 request is "100%," but may be irrelevant; 50 errors in 5 minutes is relevant regardless of the percentage.
- **One rule covering all sites, grouped by a site identifier**, instead of one rule per site — if the alerting tool supports multiple instances of the same rule (one per distinct value of the grouping field), a problem on one site fires independently of the others, without duplicating the rule definition.
- Remember the "no data" adjustment from section 4.2 — this type of alert (counting a rare event) frequently returns "no data" in the healthy state (zero errors = nothing to count), and by default many tools treat that as "fire," producing a permanent false alarm until the adjustment is made.

---

## 8. Generic deployment checklist

- [ ] Telemetry collector installed, authentication validated, no new port exposed publicly
- [ ] CPU/RAM/disk metrics confirmed in the observability backend (real name, not the predicted one)
- [ ] Minimal dashboard with the 3 essential resources
- [ ] Threshold alerts calibrated against a real baseline, with "no data" behavior reviewed
- [ ] Notification channel tested end-to-end (both firing **and** resolved)
- [ ] Access to the observability backend restricted to those who need it
- [ ] HTTP(s) uptime for every site monitored from outside the server
- [ ] SSL expiry monitored (native or own script)
- [ ] Uptime/SSL notifications wired into the same alert channel
- [ ] Application log rotation installed, with rotation **and** compression confirmed separately
- [ ] (Optional) Per-route HTTP traffic observability, with cardinality validated before implementation
- [ ] (Optional) Per-site 5xx error alert

---

## 9. Common pitfalls catalog

| Symptom | Likely cause | How to confirm |
|---|---|---|
| Authentication fails (401/403) on export, config looks right | Environment variable holding the literal text `\$VAR` instead of the value (heredoc escaping), or an unsubstituted placeholder | `cat` the credentials file — it should show the real value, not `$VAR` or `<placeholder>` |
| "Invalid configuration key" on validation | That metric/option name doesn't exist in this version of the software | Run the tool's component/version listing command before using an uncommon key |
| Process metric is always zero, with no identification of which process | Process belongs to a different OS user and the Collector lacks cross-user read capability | Grant the specific capability to the Collector binary; reapply it after every package upgrade |
| Disk/CPU percentage doesn't match what the OS reports | Virtual filesystems/devices (loop devices, tmpfs) being included in the calculation | Explicitly exclude them in the corresponding scraper — disk and filesystem have independent filters |
| Alert shows unrendered template text (`{{ variable }}`) in the rule editor | Expected behavior — the annotation is only populated once the rule is actually in a firing state | Not a bug; confirm correct rendering during the real end-to-end test |
| "Top N" dashboard panel shows more items than N | Query recalculated at every point in time instead of once for the whole period | Change the panel's query type to instant |
| Percentage panel shows an unstable 0% or 100% | Aggregation window too dynamic, too few samples per point | Switch to a fixed, wider window |
| Rare-event count alert stays permanently firing even when the event never happens | The alerting tool's default behavior on receiving "no data" | Explicitly set the rule's "no data" behavior to "Normal" |
| A counter metric built via an observability pipeline shows values that decrease over time | The component generating the metric isn't honoring cumulative-accumulation semantics (common in components marked experimental) | Don't rely on that component for this use case; consider an alternative path (e.g., raw logs + query-time counting) |
| A third-party uptime/SSL service no longer offers for free a feature that used to be free | The vendor changed its plan policy, outside your control | Keep a simple plan B documented (your own script), don't depend on a single vendor for everything |
