# Linux `locate` vs `find` Commands — Reference Notes

---

## `locate` — Fast, but From a Cached Database

```bash
locate httpd.conf
```

**How it works:** `locate` doesn't scan the filesystem in real time — it searches a pre-built index database (`/var/lib/mlocate/mlocate.db` or `plocate.db`) that's usually updated once a day via a cron job (`updatedb`).

**Pros:**
- Extremely fast — even across the entire filesystem — because it's just a database lookup
- Simple syntax, no need to specify a starting path

**Cons:**
- Can return stale results — a file created 5 minutes ago won't show up until the next `updatedb` run
- Not available by default on all minimal server installs (`mlocate` package often needs installing)
- No filtering by permissions, size, modification time, etc.

**Force a fresh index before searching (if you suspect staleness):**
```bash
sudo updatedb
locate plugin-cfg.xml
```

**Useful flags:**
```bash
locate -i httpd.conf      # case-insensitive
locate -c httpd.conf      # count matches instead of listing them
```

---

## `find` — Slower, but Real-Time and Far More Powerful

```bash
find /opt/IBM/WebSphere -name "SystemOut.log"
```

**How it works:** `find` walks the actual filesystem tree in real time, starting from the path you give it, checking every file against your criteria.

**Pros:**
- Always accurate — sees files created seconds ago
- Extremely flexible: filter by name, type, size, modification time, permissions, owner, and combine with actions like `-exec`
- Works even without any indexing daemon running (zero dependency on background services)

**Cons:**
- Slower on large filesystems since it's a live directory walk
- Syntax is more verbose

### Practical `find` Examples for WAS/IHS Troubleshooting

```bash
# Find all SystemOut.log files across all profiles
find /opt/IBM/WebSphere -name "SystemOut.log"

# Find files modified in the last 30 minutes (great during an active incident)
find /opt/IBM/WebSphere/AppServer/profiles/AppSrv01/logs -mmin -30

# Find files larger than 500MB (disk-full triage — huge log/heap dump files)
find / -type f -size +500M 2>/dev/null

# Find and delete old thread dumps older than 7 days
find /opt/IBM/WebSphere -name "javacore.*.txt" -mtime +7 -delete

# Find files owned by a specific user (checking who owns stray files after a bad deploy)
find /opt/IBM/WebSphere -user wasadmin

# Find and run a command on each match (e.g., check file permissions)
find /opt/IBM/HTTPServer/ssl -name "*.kdb" -exec ls -l {} \;

# Find empty directories (leftover from a failed cleanup script)
find /opt/IBM -type d -empty
```

---

## Side-by-Side Comparison

| | `locate` | `find` |
|---|---|---|
| Speed | Very fast (indexed) | Slower (live scan) |
| Accuracy | Can be stale (daily index) | Always current |
| Filtering | Name only | Name, size, time, type, permissions, owner, and more |
| Actions on results | None built-in | `-exec`, `-delete` |
| Dependency | Needs `updatedb`/cron running | None |

---

## Which One to Reach For, In Practice

- **"Where is this config file on the whole system?"** → `locate` (fast, and staleness rarely matters for static config files)
- **"Find that heap dump created in the last hour during this incident"** → `find` (must be real-time, `locate`'s index won't have it yet)
- **"Clean up old dumps/logs older than N days as part of a maintenance script"** → `find` with `-mtime` and `-delete` or `-exec`, since this is exactly the kind of task you'd wire into a cron job or Ansible task alongside patching automation.