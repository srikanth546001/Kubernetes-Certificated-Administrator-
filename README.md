ROLE
You are a senior Azure cloud engineer specializing in Azure Monitor / Log Analytics, KQL, and operating in restricted enterprise networks. Give me a complete, production-ready implementation — not just guidance.

CONTEXT
- Azure Log Analytics workspace name: wfee2producpsmplaw
- Goal: Export ALL distinct ISAAC container log entries from the last 24 hours into a single local output file (JSONL or CSV — your pick, justify it).
- Estimated volume: ~131,000–134,000 distinct rows over 24h.
- Source tables: KubePodInventory (filter PodLabel has "isaac") joined to ContainerLog on ContainerID.
- Required output fields: TimeGenerated, Namespace, PodName (Name), LogEntry.

CONSTRAINTS (these are the hard problems — your solution MUST handle all of them)
1. Volume exceeds a single query response. Azure per-query limits are 500,000 rows / 64 MB / 10 min. My ~134K-row dataset with distinct/order-by pushes the 64 MB cap, so a single query returns partial results.
2. Corporate proxy on my local machine BLOCKS api.loganalytics.io. That kills `az monitor log-analytics query`, direct REST, and PowerShell `Invoke-RestMethod` / `Invoke-AzOperationalInsightsQuery` when run locally. Assume I cannot get the proxy changed quickly.
3. Do NOT use `serialize | row_number()` for pagination — it triggers E_LOW_MEMORY_CONDITION on datasets this size. Use TimeGenerated-based time-window chunking instead.
4. The new endpoint api.loganalytics.azure.com may or may not be blocked by the same proxy rule. Treat it as a fallback worth testing, not a guarantee.

WHAT I WANT YOU TO PRODUCE
Deliver a numbered, end-to-end runbook with all of the following, fully written out (no "fill this in later"):

1. Decision tree: in plain English, which of these execution environments I should use and why — Azure Cloud Shell (primary), local machine with new endpoint (fallback), Data Export Rule to Storage (long-term).

2. A complete Bash script for Azure Cloud Shell that:
   - Looks up the workspace customerId from the workspace name + resource group.
   - Loops through 24 one-hour windows over the last 24 hours (UTC).
   - Runs the KQL join per window with no `distinct` and no `serialize | row_number` (dedup happens at file level after).
   - Saves each chunk to its own JSON file.
   - Merges chunks and dedupes by (TimeGenerated, LogEntry) into a single JSONL file.
   - Includes basic retry-on-failure (3 attempts per chunk, exponential backoff) and logs which windows succeeded/failed.
   - Prints final row count and file size.
   - Includes a commented-out 30-min-window variant for the case where any 1h window still returns partial results.

3. The exact KQL the script should send, parameterized for start/end datetime.

4. A PowerShell equivalent of the same script (using `Invoke-AzOperationalInsightsQuery` or the REST API with a bearer token from `Get-AzAccessToken`) for users who prefer PowerShell in Cloud Shell.

5. A "local laptop fallback" version that targets https://api.loganalytics.azure.com (the new endpoint) with bearer-token auth via az CLI, in case my proxy only blocks the old hostname. Include how to test reachability first (curl one-liner).

6. A separate, smaller section on setting up a continuous Data Export Rule (workspace → Storage Account) for the same table, so future pulls don't repeat this pain. Include the exact `az monitor log-analytics workspace data-export create` command.

7. Validation steps after the export: how to confirm row count matches expectation, how to spot partial-result warnings in the JSON response, how to convert JSONL → CSV with `jq` if I want CSV.

8. A short troubleshooting table: symptom → likely cause → fix. Cover at minimum: partial query failure, 401/403 auth errors, proxy connection refused, empty chunks, clock skew between windows.

FORMAT
- Use code blocks for every script, command, and KQL snippet.
- Inline-comment the scripts heavily — assume the next person to run this isn't me.
- No marketing fluff, no "as an AI" preamble, no recap of my prompt back to me. Start with section 1.

ASSUME
- I have Reader + Log Analytics Reader role on the workspace.
- The workspace lives in a resource group I'll fill in as <RG>.
- jq and az CLI are available in Cloud Shell by default.
- Today is the day I run this — use the last 24h relative to "now".
