# EC2 AI Agent Remediation System

## Project Scenario

Netflix’s DevOps team reported slow response times when multiple EC2 instances failed during peak traffic events. Previously, engineers had to manually extract EC2 Instance IDs from incident descriptions, search for the record, and trigger remediation scripts by hand. This manual process created bottlenecks and delayed recovery.

We built **Workflow 3** to solve this: an AI agent embedded in ServiceNow that can identify EC2 instance details from incident records, validate existence, request human approval, and execute a remediation API call automatically.

---

## Business Need

The solution needed to:
- Reduce manual remediation workload on engineers
- Speed up EC2 instance recovery during high-volume incident periods
- Improve accuracy when extracting EC2 instance information
- Maintain human-in-the-loop decision-making for accountability
- Log all actions for audit and troubleshooting

---

## Technical Workflow

### Step 1: Incident Parsing  
The agent checks new incident records in the `u_incident` table. It parses the description field using regex to identify potential EC2 instance IDs (format: `i-xxxxxxxxxxxxxxxxx`).

### Step 2: EC2 Instance Validation  
The script queries the `x_snc_ec2_monito_0_ec2_instance` table to confirm the instance exists and extract its `sys_id`.

### Step 3: Human Approval  
Once validated, the agent asks:  
_"I found EC2 instance `i-05f25ae3c5a995dc6` linked to incident `INC0012345`. Would you like to run the remediation script?"_

### Step 4: Remediation Execution  
If approved, the script runs a REST call to the AWS Integration Server using credentials linked via `sys_alias`. It triggers the existing `Call_ec2_remediation.js` script.

### Step 5: Logging  
The outcome (success or failure) is logged to the `x_snc_ec2_monito_0_remediation_log` table along with:
- Response time
- HTTP status
- Full JSON payload
- User who approved it

---

## Troubleshooting & Issues

### 1. `ReferenceError` on Line 36
**Cause**: Using a global-scoped class (`sn_cc.ConnectionInfoProvider`) inside a scoped script.  
**Fix**: Replaced the credential reference method with one compatible with the scoped app or adjusted app scope.

📍 *[Screenshot Placeholder: ReferenceError trace]*

---

### 2. Instance Not Found in Monitoring Table
**Cause**: Instance ID not found or record missing.  
**Fix**: Added defensive checks and a fallback message to notify the user if the EC2 instance wasn’t found.

📍 *[Screenshot Placeholder: Missing EC2 record]*

---

### 3. No Action After Script Execution
**Cause**: REST call made successfully, but log entry not created.  
**Fix**: Ensured the remediation log table is write-enabled and checked for `insert()` ACLs or field restrictions.

📍 *[Screenshot Placeholder: Missing log row]*

---

## Optimization & Future Enhancements

### What Worked Well
- The human approval flow gave control to engineers while automating tedious steps.
- Logging all response data allows for post-mortem review.
- Using existing tables made integration lightweight.

### What Could Be Improved

| Area | Suggested Improvement |
|------|------------------------|
| Incident Filtering | Add logic to check only recent high-priority incidents |
| ID Detection | Expand to detect **Instance Names** as a fallback when IDs aren’t present |
| Performance | Cache frequent queries to reduce DB hits in large-scale deployments |
| Scalability | Allow multiple instance IDs per incident to be queued and remediated |
| UX | Integrate into Slack or Teams to provide real-time approvals & faster response |
| Testing | Add a dry-run mode for non-prod environments |

---

## File Structure

```
/EC2-AI-Agent-Remediation/
├── README.md
├── Diagram.png
└── ec2-ai-agent-enhancement.xml
```

---

## Status

This workflow is live and functional in our ServiceNow instance, actively supporting DevOps with faster, traceable, and AI-assisted EC2 instance recovery.
