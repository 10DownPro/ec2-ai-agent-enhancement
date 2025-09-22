# EC2 AI Agent Remediation System 🚀

## Project Scenario

Netflix's DevOps team struggled with slow incident response whenever multiple EC2 instances failed during peak streaming hours. Engineers had to:

1. Manually scan incident descriptions for instance IDs
2. Search the EC2 Instance table
3. Trigger remediation scripts by hand

This created bottlenecks, wasted engineering cycles, & slowed recovery.

**Solution:** An AI-powered ServiceNow agent that parses incidents, validates EC2 records, requests human approval, & executes remediation API calls automatically. Automation where it counts, human control where it matters.

---

## Business Need

The system needed to:

- Reduce manual remediation workload for engineers
- Speed up EC2 instance recovery during traffic spikes
- Improve accuracy when extracting instance details
- Keep human-in-the-loop approval for accountability
- Log everything for audit, compliance, & debugging

---

## Technical Workflow

### Step 1: Incident Parsing
- AI agent scans new `u_incident` records
- Regex pulls EC2 IDs in format `i-xxxxxxxxxxxxxxxxx` from short descriptions

### Step 2: EC2 Instance Validation
- Looks up the ID in `x_snc_ec2_monito_0_ec2_instance`
- If found, returns the record's `sys_id` & current `instance_status`
- If missing, returns a clean error (no crashes)

### Step 3: Human Approval
- AI agent confirms findings with the user:
  > "I found EC2 instance i-05f25ae3c5a995dc6 from incident INC0012345. Run remediation?"
- Proceeds only on explicit approval

### Step 4: Remediation Execution
- On approval, script uses the Connection & Credential Alias `AWS Integration Server C C Alias` → resolves to the HTTP Connection `AWS Integration Server Connection` with Credential `AWS Integration Server Credentials`
- REST POST goes to: `https://codon-staging.emaginelc.com/api/v1/queue/start`
- Body includes the approved `instance_id`

### Step 5: Logging

All outcomes are written to `x_snc_ec2_monito_0_remediation_log`, including:
- HTTP status & response body
- Request payload
- Timestamp & total response time (ms)
- Approver identity
- Attempted status (e.g., OFF → ON)

---

## Troubleshooting 🛠️ 
*(Annoying issues I ran into)*

### 1) Cannot convert null to an object

**Root cause:** `connectionInfo` was null because the alias had no linked Connection record.

**Fix:** Created Connection `AWS Integration Server Connection` (HTTPS host `codon-staging.emaginelc.com`, base path `/api/v1/queue/start`), linked it to Alias `AWS Integration Server C C Alias`, & attached Credential `AWS Integration Server Credentials` (Basic Auth).

### 2) Record exists but query returns nothing

**Root cause:** Code assumed a record will always be found; using fields without checking `.next()` led to null access.

**Fix:** Guarded queries with `.next()` & return clear messages when no match is found.

```javascript
// Safe pattern used in helpers
var gr = new GlideRecord('x_snc_ec2_monito_0_ec2_instance');
gr.addQuery('instance_id', instanceId);
gr.query();
if (!gr.next()) {
  return { success: false, message: 'Instance not found: ' + instanceId };
}
```

### 3) REST call OK but no logs

**Root cause:** Log table ACLs/permissions blocked inserts.

**Fix:** Enabled write access for `x_snc_ec2_monito_0_remediation_log` in the scoped app & verified `insert()` works; added defensive checks & full error strings on failure.

---

## Optimization & Future Enhancements

| Area | Next Step |
|------|-----------|
| Incident Filtering | Only scan P1/P2 or recent incidents during peak windows |
| ID Detection | Fallback to instance_name if ID is missing |
| Resiliency | Add retry with exponential backoff for 5xx/timeout |
| Scalability | Batch remediate multiple IDs per incident |
| Integrations | Slack/Teams approval buttons for faster response |
| Testing | "Dry-run" mode for non-prod with no external API call |

---

## File Structure

```
/EC2-AI-Agent-Remediation/
├── README.md
├── Diagram.png                       # High-level architecture
└── ec2-ai-agent-enhancement.xml      # Update set for ServiceNow
```