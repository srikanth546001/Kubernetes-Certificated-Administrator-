## Scenario

A production API service running on 4 EC2 instances behind a Classic ELB (Elastic Load Balancer) 
experienced a ~2-hour complete outage. The root cause was a protocol mismatch — the ELB health 
check was configured to hit HTTPS:443 but the load balancer terminates SSL and forwards all traffic 
to backend instances on HTTP:80. This mismatch caused 3 of 4 instances to silently fail health 
checks throughout the entire day — a pre-existing condition that went completely undetected. The 
single remaining "healthy" instance eventually crashed due to a Java Out-of-Memory (OOM) event, 
taking down the entire service.

To make things worse, no heap dump was generated during the OOM because the JVM flag 
-XX:+HeapDumpOnOutOfMemoryError was never configured, making post-mortem root cause analysis 
on the memory issue impossible.

The team is now executing a structured 3-phase remediation plan:
- Phase 1: Fix the ELB health check protocol mismatch
- Phase 2: Fix and update CloudWatch monitoring alarms
- Phase 3: Add JVM heap dump configuration via rolling restart across all instances

---

## Questions

**Phase 1 — ELB Health Check Fix**

1. A Classic ELB health check is configured to target HTTPS:443/api/system/health-check, but 
the load balancer terminates SSL and forwards traffic to instances on HTTP:80. 
   - What is the exact technical impact of this protocol mismatch? 
   - Why would this cause intermittent failures rather than consistent 100% failures across 
     all instances? 
   - How could this condition go undetected for an entire day while instances were still 
     technically showing as InService?
   - What does this tell you about the limitations of ELB health check reporting?

2. Before applying the health check change, the engineer curled the internal ELB DNS endpoint 
and confirmed a 200 OK (90 bytes) response on HTTP:80, verifying the backend serves that path 
before making any changes.
   - Why is this pre-flight validation step critical before modifying a health check in production?
   - What specific failure scenario does this validation prevent?
   - What would have happened if the curl had returned a non-200 response?
   - Is curling the ELB DNS sufficient validation, or would you want additional checks? 
     What else would you verify?

3. As part of this fix, the health check interval was changed from 6 seconds to 30 seconds.
   - What are the operational trade-offs of running a very aggressive (6s) health check 
     interval on a production fleet?
   - What are the risks of a less aggressive (30s) interval?
   - In what specific scenarios would you keep or even reduce the interval below 6 seconds?
   - How does interval length affect the time-to-detect and time-to-recover during an 
     actual instance failure?

4. After applying the fix, the team waited for all 4 instances to be confirmed InService 
across 3 consecutive 30-second probe windows (a total 90-second monitoring period) before 
declaring the change complete.
   - Why specifically 3 consecutive windows rather than just 1 or 2?
   - What does passing 3 consecutive windows confirm that a single passing window does not?
   - How would you calculate the appropriate number of consecutive windows needed to 
     confidently validate a health check fix?
   - What would you do if only 3 out of 4 instances passed after the change?

---

**Phase 2 — CloudWatch Alarms**

5. The team discovered that an UnhealthyHostCount >= 1 alarm already existed on the load 
balancer but had no SNS action wired to it (Actions: []). The alarm was technically 
evaluating correctly but silently — it fired all day during the incident without notifying anyone.
   - How is it possible for a CloudWatch alarm to be in ALARM state all day without anyone 
     being notified?
   - What process or configuration review failure allowed this alarm to exist without a 
     notification target?
   - How would you audit your entire AWS environment to find other alarms in this same 
     broken state?
   - What is the difference between an alarm that is misconfigured vs. an alarm that is 
     correctly configured but has no action?

6. The HealthyHostCount alarm threshold was updated from < 1 (meaning zero healthy hosts) 
to < 2 (meaning fewer than 2 healthy hosts).
   - Explain the exact operational difference between these two thresholds in plain terms.
   - What specific failure scenario does the new < 2 threshold catch that the old < 1 
     threshold completely missed?
   - In the context of this incident, if the < 2 threshold had been in place originally, 
     at what point during the day would the alarm have fired?
   - How should fleet size influence how you set HealthyHostCount thresholds? 
     Give examples for fleets of 2, 4, and 10 instances.

7. The team routed the UnhealthyHostCount alarm to synegen-critical (high severity) rather 
than synegen-warnings (low severity), reasoning that any unhealthy host in a 4-node fleet 
represents meaningful capacity loss with cascade risk.
   - Do you agree with this severity decision? Justify your answer.
   - How should fleet size affect alarm severity routing decisions?
   - What is cascade risk in this context, and how does losing 1 of 4 instances create it?
   - How would your severity routing decision change for a fleet of 20 instances vs 4 instances?

---

**Phase 3 — JVM Heap Audit**

8. The OOM event that caused the final outage left no heap dump file because the JVM flag 
-XX:+HeapDumpOnOutOfMemoryError was not configured on any of the 4 instances. As a result, 
post-mortem heap analysis was impossible.
   - What is the operational consequence of not having this flag set in a production JVM service?
   - Walk through what a heap dump actually contains and how you would use it during 
     post-mortem analysis.
   - Should -XX:+HeapDumpOnOutOfMemoryError be considered a default/mandatory flag for all 
     production JVM services? Why or why not?
   - What are the risks or downsides of enabling this flag? (think about disk space, 
     timing, file size)
   - How would you ensure this flag is never missing again across your entire fleet? 
     What tooling or process would you put in place?

9. The JVM heap is configured at 2GB (-Xms2048m / -Xmx2048m) on instances with approximately 
7.8GB of total RAM — roughly 25% utilization. The team recommended leaving the heap at 2GB 
rather than increasing it, arguing that the OOM was caused by the health check bug (single 
instance absorbing all traffic) rather than an undersized heap.
   - Do you agree with the decision to leave the heap at 2GB? What is the reasoning?
   - Under what specific evidence or conditions would you reconsider resizing the heap upward?
   - What is the risk of bumping -Xmx preemptively without supporting data?
   - How does the G1GC garbage collector behave differently at 2GB vs 6GB heap sizes? 
     What tuning considerations change?
   - What metrics would you monitor after Phase 3 completes to determine if 2GB is 
     actually sufficient going forward?

10. The rollout plan uses a rolling restart strategy — one instance at a time — with a 
mandatory wait condition of >= 60 seconds for ELB InService confirmation before proceeding 
to the next instance. Total estimated disruption per instance is 5 to 7 minutes, with 
N-1 capacity maintained throughout.
    - Why is a rolling restart preferred over a simultaneous restart of all instances?
    - What is the minimum healthy capacity maintained at any point during this rollout, 
      and why does that matter?
    - Why is the >= 60s InService wait condition (2 successful 30s probe windows) used 
      as the gate before moving to the next instance?
    - What would you do if an instance fails to return InService within the expected window?
    - How would you handle a situation where the 3rd instance fails mid-rollout — 
      do you proceed, pause, or roll back?

11. The rollback plan involves restoring /etc/default/tomcat9 from a .pre-SI303 backup 
file created on each instance prior to the edit, then running systemctl restart tomcat9.
    - What are the specific risks of a file-based config rollback compared to an 
      AMI-based or immutable infrastructure rollback?
    - In what scenarios would a config file rollback be insufficient or unsafe?
    - When would you choose an AMI rollback over a config rollback, and what are the 
      trade-offs in terms of speed, risk, and complexity?
    - How would you improve this rollback strategy to make it more reliable and auditable 
      at scale?

---

**Coordination & Process**

12. The change window was scheduled for Tuesday evening after the team was explicitly told 
to avoid Thursday nights and Fridays due to limited on-call coverage and a policy against 
changes that could generate support calls over the weekend.
    - What does this scheduling constraint tell you about how change windows should be 
      formally defined in an organization?
    - What criteria would you use to establish change windows in your own team or organization?
    - How do you balance the urgency of a remediation (this was a post-incident fix) against 
      the risk of executing it during a low-coverage period?
    - What additional approval gates or safeguards would you put in place for changes 
      executed during evening windows with reduced staffing?

13. Three different engineers were responsible for different phases of this remediation — 
one executed the ELB fix, one handled the CloudWatch alarms, and a third is leading the 
JVM heap rollout. Each phase was documented with pre-change state, post-change state, 
verification steps, and rollback procedures.
    - What coordination risks does splitting a remediation across multiple engineers introduce?
    - How does thorough ticket documentation (pre/post state, verification, rollback) 
      mitigate handoff risks between engineers?
    - What could go wrong if the documentation was absent or incomplete between phase handoffs?
    - In your experience, what is the most common failure mode when multiple engineers 
      are executing sequential phases of a production change?
    - How would you structure a handoff checklist between engineers for a multi-phase 
      production change like this one
