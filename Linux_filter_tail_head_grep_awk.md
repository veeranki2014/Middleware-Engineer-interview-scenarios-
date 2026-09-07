# Linux Filter Commands — head, tail, grep, awk

---

## `head` — Show the Beginning of a File

By default shows the first **10 lines**.

```bash
head SystemOut.log
```

### Flags

| Flag | Meaning |
|---|---|
| `-n N` | Show first N lines |
| `-c N` | Show first N bytes |
| `-q` | Quiet — suppress filename headers when reading multiple files |
| `-v` | Verbose — always print filename headers, even for a single file |

### Examples

```bash
# First 20 lines
head -n 20 SystemOut.log

# First 100 bytes (useful to peek at binary/log file headers)
head -c 100 heapdump.hprof

# First 5 lines across multiple log files, with filename headers
head -n 5 server1/SystemOut.log server2/SystemOut.log
```

---

## `tail` — Show the End of a File

By default shows the last **10 lines**. This is the one used constantly for live log monitoring.

```bash
tail SystemOut.log
```

### Flags

| Flag | Meaning |
|---|---|
| `-n N` | Show last N lines |
| `-f` | Follow — keep the file open and print new lines as they're appended (live tailing) |
| `-F` | Follow, but also re-attach if the file is rotated/recreated (safer than `-f` for logs that rotate) |
| `-c N` | Show last N bytes |

### Examples

```bash
# Last 50 lines
tail -n 50 SystemOut.log

# Live tail during an incident
tail -f SystemOut.log

# Live tail that survives log rotation (best for prod monitoring)
tail -F SystemOut.log

# Tail multiple logs at once, live
tail -f SystemOut.log SystemErr.log
```

---

## `grep` — Search for Patterns in Text

The core text-search tool — filters lines matching a pattern.

```bash
grep "OutOfMemoryError" SystemOut.log
```

### Flags

| Flag | Meaning |
|---|---|
| `-i` | Case-insensitive search |
| `-v` | Invert match — show lines that do NOT match |
| `-n` | Show line numbers of matches |
| `-c` | Count matching lines instead of printing them |
| `-r` / `-R` | Recursive search through directories |
| `-l` | List only filenames that contain a match (not the matching lines) |
| `-A N` | Show N lines **after** each match |
| `-B N` | Show N lines **before** each match |
| `-C N` | Show N lines of context on **both** sides |
| `-E` | Extended regex (enables `+`, `?`, `|`, `()` without escaping) |
| `-w` | Match whole word only |
| `--color` | Highlight matches in color |

### Examples

```bash
# Case-insensitive search
grep -i "error" SystemOut.log

# Show line numbers
grep -n "Exception" SystemOut.log

# Count how many times an error occurred
grep -c "OutOfMemoryError" SystemOut.log

# Recursive search across all logs in a directory
grep -r "ConnectionPoolTimeout" /opt/IBM/WebSphere/profiles/AppSrv01/logs/

# Show 10 lines of context after a match (great for stack traces)
grep -A 10 "OutOfMemoryError" native_stderr.log

# Show 5 lines before and after (full context around an error)
grep -C 5 "FFDC" SystemOut.log

# Exclude noise — show everything EXCEPT INFO-level lines
grep -v "INFO" SystemOut.log

# Extended regex — match either pattern
grep -E "OutOfMemoryError|StackOverflowError" SystemOut.log

# Combine with tail -f for live error watching
tail -f SystemOut.log | grep -i "exception"
```

---

## `awk` — Pattern Scanning and Field-Based Text Processing

`awk` treats each line as a set of fields (columns), split by whitespace by default. Far more powerful than `grep` when you need to extract or compute values from structured output.

```bash
awk '{print $1}' access.log
```

### Key Concepts / Flags

| Syntax | Meaning |
|---|---|
| `$1`, `$2`, ... | Reference field 1, field 2, etc. |
| `$0` | The entire line |
| `-F ':'` | Set field separator (default is whitespace) |
| `NR` | Current line/record number |
| `NF` | Number of fields in the current line |
| `'pattern {action}'` | Only run action on lines matching pattern |
| `BEGIN {}` | Run once before processing starts |
| `END {}` | Run once after all lines are processed |

### Examples

```bash
# Print the first column (e.g., PID from a ps output)
ps -ef | awk '{print $2}'

# Set a custom field separator (colon-separated file like /etc/passwd)
awk -F ':' '{print $1}' /etc/passwd

# Print specific columns (username and UID)
awk -F ':' '{print $1, $3}' /etc/passwd

# Print line numbers alongside content
awk '{print NR, $0}' SystemOut.log

# Only print lines matching a pattern, then extract a field
awk '/OutOfMemoryError/ {print $1, $2}' SystemOut.log

# Sum a numeric column (e.g., total bytes from a log's 10th field)
awk '{sum += $10} END {print sum}' access.log

# Print lines where a field exceeds a threshold (e.g., response time > 5000ms)
awk '$NF > 5000 {print $0}' access.log

# Combine with grep: filter then extract a specific field
grep "ERROR" SystemOut.log | awk '{print $1, $2, $NF}'

# Count occurrences grouped by a field (e.g., count requests per status code)
awk '{count[$9]++} END {for (code in count) print code, count[code]}' access.log
```

---

## Side-by-Side Summary

| Command | Best For |
|---|---|
| `head` | Peek at the start of a file (headers, first few log entries) |
| `tail` | Watch the end of a file, live monitoring with `-f`/`-F` |
| `grep` | Search for a text pattern, filter lines, show context around matches |
| `awk` | Extract/compute values from structured/columnar text, field-based processing |

---

## Common Combined Pattern for WAS/IHS Troubleshooting

```bash
# Live-tail SystemOut.log, filter only ERROR/Exception lines, extract timestamp and message
tail -F SystemOut.log | grep -i --line-buffered "error\|exception" | awk '{print $1, $2, $0}'
```