# Debugging OOMKilled Pods in Kubernetes: The Pro's Framework

**By Kamal Yadav**

---

## The 2 AM Phone Call

It's 2 AM. Your on-call alert fires: OOMKilled pods in production. Customers are bleeding. You have minutes, not hours.

What do you do?

For years, I'd jump straight to solutions: "Increase memory limit. Scale up replicas. Restart the pods." Sometimes it worked. Sometimes it didn't. And when it didn't, I'd wasted precious time while customers suffered.

Then I learned something that changed how I approach every production incident: **the difference between treating symptoms and diagnosing problems.**

This is the framework I now use for every OOMKilled scenario. And honestly, it applies to almost every production debugging situation—not just memory.

---

## The Wrong Approach (I Used to Do This)

```
Alert fires: OOMKilled
          ↓
My thought: "App needs more memory"
          ↓
My action: Increase memory limit to 2Gi
          ↓
Result: Pod OOMKills again in 2 hours
          ↓
Me at 4 AM: "Ugh, what's going on??"
```

The problem: I was treating a *symptom* as if it were a *diagnosis*.

OOMKilled just means "the container hit its memory limit." It tells you **where** the problem manifested—not **what** the problem is. Those are completely different things.

---

## Three Different Problems, One Symptom

The app consumed too much memory and got OOMKilled. But why?

**1. Memory Leak** (app bug)
- App allocates memory on every request and never releases it
- More memory just delays the crash by a few hours
- **The fix:** Find and fix the leak, then normal memory requirements resume

**2. Traffic Surge** (load increase)
- More users = more in-memory data structures
- More replicas needed, not more memory per pod
- **The fix:** Scale horizontally, not vertically

**3. Data Model Changed** (external change)
- Payload size grew, dataset size grew, cache warming is bigger
- App is working as designed—it's just holding more data
- **The fix:** Optimize data handling or increase memory intentionally

**If you increase memory without knowing which one it is, you're just gambling.**

---

## The Pro's Framework: 7 Phases

Here's how I diagnose now:

### Phase 1: Immediate Stabilization (2 minutes)

**Don't diagnose yet. Stop the bleeding first.**

```bash
kubectl scale deployment <name> -n <namespace> --replicas=6
```

Why: Distribute load across more pods. If one OOMKills, others handle traffic. This buys you 30 minutes of stability while you investigate *without* customer impact.

**Remember:** Diagnosis can wait. Customer uptime cannot.

---

### Phase 2: Confirm the Symptom (3 minutes)

Is it really OOMKilled, or something else?

```bash
kubectl describe pod <pod-name> -n <namespace>
```

Look for:
- `Reason: OOMKilled` in the "Last State" section
- Exit code 137 (SIGKILL from OOM) vs. 143 (SIGTERM, graceful)
- How many restarts in how many minutes? (10 restarts in 5 minutes = tight loop)

**Why this matters:** CrashLoopBackOff looks similar but has different causes. Eviction is different from OOMKilled. You need to confirm the actual problem.

---

### Phase 3: Establish the Baseline (5 minutes)

**This is the question I should have asked first:**

When was the memory normal? When did it change?

Pull up Grafana/Prometheus:

```
Query: container_memory_usage_bytes{pod="<pod>", namespace="<ns>"}
```

Look back:
- **24 hours ago:** Was memory stable at 100Mi?
- **48 hours ago:** Any change there?
- **1 week ago:** What's the normal pattern?

**This single graph tells you everything:**

| Pattern | Meaning |
|---------|---------|
| Stable at 100Mi for 7 days, then suddenly climbs today | **Something changed today** |
| Gradual climb all week | **Chronic leak or slow data accumulation** |
| Saw-tooth pattern (up-down-up-down) | **Normal garbage collection—just not enough headroom** |
| Linear climb straight to limit | **Leak (not traffic, leak)** |
| Sudden spike to limit | **Burst (traffic, batch job, or allocation)** |

---

### Phase 4: Find the Culprit (5 minutes)

**Look inside the container. Which process is consuming memory?**

```bash
kubectl exec -it <pod-name> -n <namespace> -- sh

# Inside the pod:
top
# or
ps aux --sort=-%mem | head -10
# or
free -h
```

**Critical distinction:**

- If the **app process** is consuming 850Mi/1Gi → **App-level issue**
- If a **sidecar** (log shipper, metrics exporter) is the culprit → **Configuration issue**
- If memory is **spread across many processes** → **System overhead**

Example output:
```
PID    USER    %MEM   COMMAND
100    root    85     java -jar app.jar       ← The app itself is the culprit
201    root    10     /prometheus exporter    ← Or the sidecar is
```

---

### Phase 5: Check Logs for Context (3 minutes)

```bash
kubectl logs <pod-name> -n <namespace> --tail=200
```

Look for:
- Errors or warnings
- Batch operations (bulk data loads, cache warming)
- Database query patterns
- Config changes
- Deployment timestamps

**Example red flags:**
- "Cache loaded 50GB of data" → Expected memory growth
- "ERROR: OutOfMemory exception" → Actual error to investigate
- "Connected to database" → Did DB connection parameters change?

---

### Phase 6: Form Your Hypothesis (5 minutes)

Now you have data. Form a hypothesis—with a verification plan.

**Example (Good):**
> "Memory climbed linearly, no traffic increase, app process consuming 850Mi. My hypothesis: **memory leak in the app**. To verify: I'd check the commit history for recent changes, enable heap dump on OOM, and look for unbounded collections or caches."

**Example (Bad):**
> "It's probably a memory leak. Let me increase memory limit."

Notice the difference? One has a *falsifiable test*. The other doesn't.

---

### Phase 7: The Fix (Informed Decision)

Once you know the cause, the fix becomes obvious:

| Root Cause | Indicator | Fix |
|---|---|---|
| Memory leak in app | Linear climb, app process is culprit, no recent deployment | Work with app team; enable profiling; plan code fix |
| Insufficient headroom | Saw-tooth pattern, normal GC, just no room to breathe | Increase memory limit by 20-30%; tune GC |
| Traffic surge | Sudden spike, request metrics correlate | Scale horizontally; check HPA settings |
| Data growth | Linear climb, database row counts 3x higher | Optimize queries; implement pagination; increase memory intentionally |
| Bad sidecar config | Sidecar consuming memory, not app | Reduce log verbosity; increase sidecar limits; or remove sidecar |

---

## The Real Lesson: Educated Guesses vs. Hypothesis-Driven Diagnosis

Here's the mindset shift that matters:

**Intermediate approach:**
"Memory is high, so maybe it's a leak. I'll increase the limit and see."
- You're guessing
- You're not verifying
- You're burning time

**Pro approach:**
"Memory climbed linearly. If it's a leak, I'd see [X] in the logs. If it's data growth, I'd see [Y] in the database. Let me check [Y] first."
- You're still guessing (you have a hypothesis)
- But you're testing it systematically
- You find the real problem fast

**The difference:** Structure. Verification. Falsifiability.

---

## What I Should Remember

1. **Symptoms ≠ Diagnosis.** OOMKilled is where the problem showed up, not what the problem is.

2. **Stabilize before diagnosing.** Scale up replicas first. Investigate second. Keep customers happy.

3. **Establish the baseline.** When did this start? That's your first clue about what changed.

4. **Look inside the container.** Which process is the culprit? This tells you who to blame (app, sidecar, system).

5. **Hypothesis + Verification.** Don't just guess. Guess *with a test plan*.

6. **Act informed, not reactive.** Understand the root cause before you apply the fix.

7. **Prevention.** Memory profiling in production. Alerts at 60% utilization, not OOM. Heap dumps on failure. Automated regression testing.

---

## The Hard Truth

The difference between a DevOps engineer and a **platform architect** isn't that architects know more commands. It's that architects think in systems:

- What can fail?
- How will I know when it fails?
- What's the *actual* problem vs. the *symptom*?
- How do I verify before I act?
- How do I prevent it next time?

That's what separates 2 AM panic from 2 AM confidence.

---

**Next time you get an alert, don't jump to solutions. Take 15 minutes. Gather data. Form a hypothesis. Verify it. Then act.**

That's pro-level debugging.

---

## Questions?

Drop them in the comments. And if you've faced similar scenarios, I'd love to hear how you approached them differently.

**Happy debugging.** 🚀
