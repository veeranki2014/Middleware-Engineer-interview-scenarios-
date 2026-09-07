# Java Thread Dump States — Reference Notes

---

## Standard Java Thread States (from `Thread.State` — HotSpot/OpenJ9 `jstack`)

| State | Meaning |
|---|---|
| **NEW** | Thread object created but `start()` not yet called — rarely seen in a dump since it hasn't begun execution |
| **RUNNABLE** | Actively executing on a CPU, or ready and waiting for CPU scheduling — this is the state you're hunting for in a high-CPU investigation |
| **BLOCKED** | Waiting to acquire a monitor lock (`synchronized` block/method) that another thread currently holds — the lock contention/deadlock indicator |
| **WAITING** | Waiting indefinitely for another thread to perform a specific action (`Object.wait()` with no timeout, `Thread.join()`, `LockSupport.park()`) |
| **TIMED_WAITING** | Same as WAITING but with a timeout (`Thread.sleep()`, `Object.wait(timeout)`, `LockSupport.parkNanos()`) |
| **TERMINATED** | Thread has completed execution — rarely relevant in analysis |

---

## IBM JVM (`javacore`) Thread States — WebSphere Traditional

IBM's thread dumps use a slightly different notation, shown as a state flag next to each thread:

| Flag | State | Meaning |
|---|---|---|
| **R** | Runnable | Actively running or ready to run |
| **CW** | Condition Wait | Waiting on a condition variable (similar to WAITING/TIMED_WAITING) |
| **B** | Blocked | Waiting on a monitor lock held by another thread |
| **P** | Parked | Thread parked via `LockSupport.park()` |
| **S** | Suspended | Explicitly suspended (rare, usually debugging-related) |
| **Z** | Zombie/Dying | Thread in the process of terminating |
| **D** | Deadlocked/Debug-related | Thread involved in a JVM-detected deadlock |

**Example line from a `javacore.txt`:**
```
3XMTHREADINFO      "WebContainer : 12" J9VMThread:0x00007F... j9thread_t:0x00007F..., state:R, prio=5
```

---

## What Each State Tells You During an Investigation

### RUNNABLE / R — High-CPU Suspects

If a hot native TID (from `top -H`) correlates to a thread showing RUNNABLE across multiple consecutive dumps taken 10 seconds apart — that's an infinite loop or CPU-bound code.

### BLOCKED / B — Lock Contention and Deadlocks

```
"Thread-A" #45 prio=5 os_prio=0 tid=0x... nid=0x1a2 waiting for monitor entry [0x...]
   java.lang.Thread.State: BLOCKED (on object monitor)
        at com.myapp.OrderService.processOrder(OrderService.java:112)
        - waiting to lock <0x000000076ab62208> (a com.myapp.OrderLock)
        - locked <0x000000076ab62300> (a com.myapp.InventoryLock)
```

If Thread-A is BLOCKED waiting for a lock that Thread-B holds, and Thread-B is simultaneously BLOCKED waiting for a lock Thread-A holds — that's a **deadlock**, and both HotSpot's `jstack` and IBM's `javacore` will explicitly flag this at the bottom of the dump ("Found one Java-level deadlock").

### WAITING / TIMED_WAITING / CW — Usually Benign, But Watch the Volume

Most application server threads sitting idle in a pool will show WAITING/TIMED_WAITING on the thread pool's work queue — completely normal. The concern is if you see **hundreds of threads** stuck in TIMED_WAITING on the same resource (e.g., a connection pool's `getConnection()` call) — that's pool exhaustion, not a healthy idle state.

### PARKED / P — Often Connection/Thread Pool Related

Common in modern concurrent utilities (`java.util.concurrent` locks, `ExecutorService` queues). If many threads are parked waiting on the *same* object, it points to contention on a shared resource — worth grepping the stack trace for what that object actually is.

---

## Practical Grep Patterns for Thread Dump Triage

```bash
# Count threads by state (IBM javacore)
grep "3XMTHREADINFO" javacore.*.txt | grep -o "state:[A-Z]*" | sort | uniq -c

# Count threads by state (HotSpot jstack)
grep "java.lang.Thread.State" threaddump.txt | sort | uniq -c

# Find all BLOCKED threads and what they're waiting on
grep -B 2 -A 5 "BLOCKED" threaddump.txt

# Check explicitly for JVM-detected deadlocks
grep -A 20 "Found one Java-level deadlock" threaddump.txt
```

---

## Interview Summary

> The main states you're looking for are RUNNABLE, BLOCKED, WAITING, and TIMED_WAITING — IBM's javacore uses slightly different single-letter codes (R, B, CW, P) but the concepts map one-to-one. RUNNABLE across multiple consecutive dumps on the same thread points to a CPU-bound loop. BLOCKED tells you about lock contention, and if two threads are mutually BLOCKED on each other's locks, that's a deadlock — which both `jstack` and `javacore` will actually call out explicitly. A high volume of threads sitting in TIMED_WAITING on the same resource, like a connection pool, is usually your signal for pool exhaustion rather than a healthy idle state.