# WebSphere High CPU Java Process — Prod Incident Troubleshooting

---

## Interview Framing (Lead With This)

> High CPU on a WAS JVM is almost always narrowed down by correlating OS-level thread CPU usage with the JVM's own thread dump — you're mapping a native thread ID to a Java thread name and stack trace to find out exactly what code is spinning.

---

## Step 1: Identify — Confirm Which Process and How Bad

```bash
top -H -p <pid>
```

The `-H` flag is the key detail — it shows **per-thread** CPU usage within the process, not just the process total. This gives you individual **native thread IDs (TID)** consuming high CPU.

```bash
# Alternative to capture a snapshot for evidence/ticket
top -H -p <pid> -b -n 1 > high_cpu_threads.txt
```

## Step 2: Isolate the Exact Thread(s)

Note the top 3-5 TIDs by CPU%, then convert to hex (Java thread dumps reference thread IDs in hex, `nid=0x...`):

```bash
printf '%x\n' <TID>
```

## Step 3: Capture a Thread Dump at the Same Moment

```bash
kill -3 <pid>
# Output lands in native_stderr.log or javacore.<timestamp>.txt (IBM JVM)
```

**Best practice:** take **3 thread dumps, 10 seconds apart** — a single dump only shows a snapshot; three lets you confirm a thread is *consistently* stuck in the same code path (a real hang) versus just busy momentarily.

## Step 4: Correlate — The Core Skill Being Tested

```bash
grep -A 30 "0x<hex_tid>" javacore.<timestamp>.txt
```

This gives the exact **Java stack trace** for that specific native thread — showing exactly which class/method is burning CPU. Common findings:

- Infinite/tight loop in application code (e.g., a `while` loop with a bad exit condition)
- Regex catastrophic backtracking (a single bad regex pattern can pin a core at 100%)
- GC threads dominating (if the top threads are `GC Worker` threads → it's not app code, it's heap pressure — different remediation path entirely)
- Deadlock/contention causing threads to spin retrying a lock
- A poorly tuned connection pool causing threads to busy-wait

## Step 5: Differentiate — GC or Application Code?

```bash
# Check GC activity concurrently
grep -i "GC" native_stderr.log | tail -50
```

If GC is running back-to-back with little heap recovered — that's heap exhaustion driving CPU, not a code bug, and the fix is heap tuning / leak investigation, not a code deploy.

## Step 6: Immediate Mitigation (Incident Bridge Actions)

- If it's one specific request/user pattern: consider isolating traffic away from the affected cluster member (`serverStatus.sh`, remove from LB rotation) while investigating
- If it's a single runaway thread and app is otherwise healthy: **do not just restart blindly** — capture the dumps first, or you lose the evidence and the same bug recurs in prod again next week
- If truly customer-impacting and dumps are already captured: restart the JVM member, let the cluster's other members absorb traffic

## Step 7: Root Cause and Permanent Fix

- Share the stack trace with the dev team — usually traces to a specific commit/deployment
- If it's a regex issue: fix the pattern
- If it's a loop bug: code fix + unit test to catch it going forward
- If it's connection pool starvation: tune pool size/timeout and add pool exhaustion alerting

## Step 8: Prevention / Process Improvement

- Add automated periodic thread dump capture (cron + `kill -3`) triggered by a CPU threshold alert, so dumps are captured automatically instead of scrambling mid-incident
- Add CPU-per-thread monitoring dashboards (not just process-level CPU) so this is caught before it becomes customer-visible
- Document the RCA and add it to the runbook, same as the FFDC/OOM playbook

---

## Interview Closing Summary

> The methodology is always the same: use `top -H` to find the exact OS thread burning CPU, convert its TID to hex, then grep that hex ID inside a thread dump taken at the same moment to get the actual Java stack trace. That tells you definitively whether it's a code-level infinite loop, GC pressure, lock contention, or connection pool starvation — and each of those has a completely different fix, so skipping this correlation step and just guessing (or just restarting) is the wrong approach and it's usually the first thing interviewers are listening for.