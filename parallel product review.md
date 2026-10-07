# Workflow : Parallel Agent
## Concurrent Execution Pattern for Multi-Branch Processing

**Version:** 1.0 | **Last Updated:** October 2026 | **Status:** Production-Ready

---

## Overview

The Parallel Agent implements **concurrent execution patterns** where independent tasks run simultaneously, then results merge at a convergence point. This is optimal for workflows where subtasks have no interdependencies and can be processed in parallel to reduce total execution time.

### Use Cases

- **Multi-perspective analysis:** Analyze content from legal, technical, and business angles simultaneously
- **Batch processing:** Process multiple independent data items in parallel
- **Competitive comparison:** Generate responses from multiple approaches, compare outputs
- **Risk assessment:** Evaluate the same scenario across multiple dimensions in parallel
- **Content generation:** Create multiple variants (e.g., formal + casual tone) concurrently

### Key Characteristics

| Aspect | Detail |
|--------|--------|
| **Execution Model** | Concurrent; independent branches run simultaneously |
| **Synchronization** | Waits for slowest branch before proceeding |
| **Data Dependencies** | None between branches; all depend on common input |
| **Latency** | Fastest branch determines bottleneck, not cumulative |
| **Scalability** | Linear improvement up to resource limits |

---

## Architecture

```
                    Input Data
                        ↓
        ┌───────────────┼───────────────┐
        ↓               ↓               ↓
    [Branch 1]     [Branch 2]     [Branch 3]
    (Parallel)     (Parallel)     (Parallel)
        ↓               ↓               ↓
    Output A       Output B       Output C
        ↓               ↓               ↓
        └───────────────┼───────────────┘
                ↓
        [Merge/Consolidate]
                ↓
            Final Output
```

### Components

1. **Input Trigger:** Splits single input to multiple branches
2. **Branch Nodes:** Independent Gemini API calls (no communication between branches)
3. **Merge Node:** Collects all branch outputs and consolidates
4. **Google Sheets Integration:** Logs branch results for comparison/audit
5. **Output Node:** Delivers consolidated result

---

## Prerequisites

### Required Accounts
- **n8n Cloud** (free tier eligible): [n8n.io](https://n8n.io)
- **Google Account:** Gmail address for Gemini API and Sheets
- **Gemini API Key:** From [Google AI Studio](https://aistudio.google.com)

### Required Setup
- Google Sheets document for logging branch results
- Gemini API credential (shared from other workflows)

---

## Installation & Configuration

### Step 1: Import Workflow JSON
```bash
# In n8n Dashboard
1. Click menu (≡) → Workflows → Import from file
2. Select Agent_3_Parallel.json
3. Click Import (do NOT save yet)
```

### Step 2: Verify Gemini Credential
**Your Gemini credential from other workflows will be reused. Verify:**

1. Left sidebar → **Credentials**
2. Confirm `Gemini Workshop Key` exists with green dot
3. If missing, follow Step 2 from [Workflow 2: Serial Agent](./Agent_2_Serial_README.md#step-2-configure-gemini-credential)

### Step 3: Configure Workflow Nodes

#### Node: Google Sheets (Input Log)
| Field | Value | Notes |
|-------|-------|-------|
| **Authentication** | Select existing credential | `Gemini Workshop Key` |
| **Sheet ID** | Your Google Sheet ID | From: `sheets.google.com/spreadsheets/d/**[ID]**` |
| **Sheet Name** | `Agent3_Input` | Create tab if missing |

#### Branch 1: Gemini - Perspective A
| Field | Value | Notes |
|-------|-------|-------|
| **Authentication** | Select existing credential | |
| **Model** | `gemini-1.5-flash` | Recommended for cost |
| **Prompt** | "Analyze from [Perspective 1] lens: {{$json.input}}" | E.g., Legal perspective, Technical perspective, etc. |

#### Branch 2: Gemini - Perspective B
| Field | Value | Notes |
|-------|-------|-------|
| **Authentication** | Select existing credential | |
| **Model** | `gemini-1.5-flash` | Same as Branch 1 |
| **Prompt** | "Analyze from [Perspective 2] lens: {{$json.input}}" | Different viewpoint than Branch 1 |

#### Branch 3: Gemini - Perspective C
| Field | Value | Notes |
|-------|-------|-------|
| **Authentication** | Select existing credential | |
| **Model** | `gemini-1.5-flash` | |
| **Prompt** | "Analyze from [Perspective 3] lens: {{$json.input}}" | Third independent analysis angle |

#### Node: Merge Results
| Field | Value | Notes |
|-------|-------|-------|
| **Mode** | Combine (merge all branch outputs) | Concatenates results |
| **Output Structure** | `{ "analysis_a": {{branch1}}, "analysis_b": {{branch2}}, "analysis_c": {{branch3}} }` | Explicit structure for clarity |

#### Node: Google Sheets (Output Log)
| Field | Value | Notes |
|-------|-------|-------|
| **Sheet ID** | Same as input log | |
| **Sheet Name** | `Agent3_Output` | Create tab if missing |
| **Append** | Columns: `input | branch1_output | branch2_output | branch3_output | merged_result` | Enables side-by-side comparison |

### Step 4: Test & Deploy
```bash
1. Click "Execute Workflow" (play button)
2. Observe parallel execution: All three branches start simultaneously
3. Check n8n execution timeline: Confirm branches run in parallel (not sequential)
4. Verify Google Sheets populated with individual branch outputs + merged result
5. If nodes fail to run in parallel, check n8n plan (free tier supports parallel)
6. Click "Save" to persist
```

---

## Parallel Execution Verification

### How to Confirm Parallel Execution
1. Run workflow → Click execution in History
2. Expand execution log → View **Timeline**
3. Confirm all three Gemini nodes **start at similar timestamps** (within 1 second)
4. If all start sequentially (1-2 second gaps), the workflow is not parallel

### What Parallel Execution Looks Like
```
Branch 1: Start 00:00, End 00:02.5 (2.5s duration)
Branch 2: Start 00:00, End 00:03.1 (3.1s duration)  ← Slowest
Branch 3: Start 00:00, End 00:01.8 (1.8s duration)

Total Time: 3.1 seconds (slowest branch)
Sequential Time (hypothetical): 7.4 seconds (2.5 + 3.1 + 1.8)
Efficiency Gain: 57% faster than serial execution
```

---

## Data Flow & Context Passing

All branches receive the **same input**. Each transforms it independently:

```javascript
// Common Input (received by all branches)
input = { 
  topic: "Enterprise AI adoption",
  content: "AI reduces costs by 35% but requires organizational change"
}

// Branch 1 (Legal Perspective)
prompt: "From a legal/compliance standpoint, analyze: {{$json.content}}"
output: "Regulatory implications: GDPR data usage, IP ownership of AI models..."

// Branch 2 (Technical Perspective)
prompt: "From a technical standpoint, analyze: {{$json.content}}"
output: "Technical challenges: Infrastructure scaling, model selection, integration..."

// Branch 3 (Business Perspective)
prompt: "From a business/ROI standpoint, analyze: {{$json.content}}"
output: "Financial impact: 35% cost reduction requires $2M investment, 18-month payback..."

// Merge Output
{
  "input": "Enterprise AI adoption...",
  "legal_analysis": "Regulatory implications...",
  "technical_analysis": "Technical challenges...",
  "business_analysis": "Financial impact...",
  "synthesis": "Holistic view combining all three perspectives"
}
```

---

## Error Handling & Troubleshooting

### Common Issues & Resolutions

| Error | Cause | Resolution |
|-------|-------|-----------|
| **Branches not running in parallel** | n8n plan limitation or configuration | Verify you're on n8n Cloud (free tier supports parallel); check workflow structure—branches must be independent |
| **Merge node fails to combine results** | Output format mismatch between branches | Ensure all Gemini nodes output text (not JSON); use explicit merge structure |
| **One branch fails, others continue** | Independent branch failure (no cross-dependency) | Expected behavior; merge node will include failed branch output as "error". Handle gracefully in next stage |
| **Inconsistent execution times** | Gemini API response variability | Normal; latency ranges 1-4 seconds per call. Average should be stable month-over-month |
| **"Rate limit exceeded" error** | Too many parallel calls overwhelming Gemini API | Reduce to 2 branches instead of 3; add 1-second delay before parallel stage; upgrade Gemini API plan |

### Debug Strategy

1. **Check Timeline:** Execution → Expand → View start times for all branches
2. **Verify Independence:** Confirm no branch references another branch's output
3. **Test Individual Branches:** Disable 2 branches, run 1 in isolation to verify Gemini calls work
4. **Monitor API Usage:** Check [Google Cloud Console](https://console.cloud.google.com) for Gemini quota/errors
5. **Validate Merge Logic:** Manually test merge node with hardcoded branch outputs

---

## Performance Characteristics

### Latency Analysis
```
Branch 1 Duration: 2.5s (fastest)
Branch 2 Duration: 3.1s (slowest) ← Bottleneck
Branch 3 Duration: 1.8s (fast)

Parallel Total:    3.1s (time of slowest branch)
Sequential Total:  7.4s (sum of all branches)
Speedup Factor:    7.4 ÷ 3.1 = 2.4x faster
Efficiency:        (1 / 3) × 2.4 = 80% parallel efficiency
```

### Cost Profile
- **Per Branch (Gemini Call):** $0.0005–$0.002 (varies by prompt length, model)
- **3 Branches:** ~$0.003–$0.006 per execution (same cost as serial; parallel doesn't add cost)
- **Cost Benefit:** Reduce latency without proportional cost increase

### Scalability Limits

| Metric | Limit | Notes |
|--------|-------|-------|
| **Branches** | 10–15 max | Beyond this, Gemini API rate limits become binding |
| **Gemini API Quota** | 60 req/min (free tier) | 3 branches = 3 reqs, fits comfortably |
| **Google Sheets Writes** | 500 req/min per Sheet | Logging 3 branches per execution is negligible |
| **n8n Execution Concurrency** | 1 workflow instance at a time | Cannot run 2+ instances of same workflow in parallel |

---

## Best Practices

### Workflow Design
1. **True Independence:** Branches must not depend on each other's outputs; all depend only on shared input
2. **Diverse Perspectives:** Each branch should analyze from a different angle (not duplicates)
3. **Consistent Prompts:** Use same prompt structure across branches, varying only the perspective
4. **Balanced Complexity:** Branches with vastly different latencies waste parallelism benefit

### Data Handling
1. **Branch Isolation:** Each branch processes input independently; no inter-branch communication
2. **Explicit Output Structure:** Merge node should produce structured output (JSON-like) for downstream consumption
3. **Logging Strategy:** Store all branch outputs in Google Sheets for comparison/audit trail
4. **Error Tolerance:** Design merge node to handle scenarios where one branch fails

### Monitoring
1. **Execution Timeline:** Regularly check that branches truly run in parallel (not sequentially)
2. **Branch Success Rate:** Track which branches fail more often (may indicate API issues)
3. **Latency Trends:** Monitor if average branch latency increases (may indicate API degradation)
4. **Cost per Execution:** Should remain stable; unexpected spikes indicate configuration issues

---

## Example: Multi-Perspective Risk Assessment

**Scenario:** Evaluate enterprise AI adoption risk from three angles simultaneously

```
Input: "Company X plans to deploy AI for customer service; 500 agents will be affected."

Branch 1 (Compliance Perspective):
  Prompt: "What compliance/regulatory risks exist?"
  Output: "GDPR concerns (data processing), labor law (retraining obligations), 
           industry-specific regulations (finance/healthcare if applicable)"

Branch 2 (Operational Perspective):
  Prompt: "What operational risks exist?"
  Output: "Integration complexity (CRM/ERP compatibility), change management (500 
           agents need retraining), vendor lock-in risk, system reliability/SLA"

Branch 3 (Financial Perspective):
  Prompt: "What financial risks exist?"
  Output: "Implementation cost overruns (typical 30-50% variance), retraining expense 
           ($5K per agent = $2.5M for 500), ongoing licensing costs, ROI timeline uncertainty"

Merge Result:
  {
    "compliance_risks": [...],
    "operational_risks": [...],
    "financial_risks": [...],
    "executive_summary": "Multi-angle risk assessment complete; proceed with mitigation 
                         plan for top 5 identified risks across all three dimensions"
  }

Total Execution Time: 3.2 seconds (vs. 9.6s if sequential)
```

---

## Advanced Variations

### Dynamic Branch Count
Use **loop with conditional** to create N branches based on input array:
```
Input: ["Legal review", "Technical review", "Financial review", "HR review"]
Output: 4 branches executing in parallel (one per item)
```

### Branch Voting / Consensus
After all branches complete, add consolidation logic:
```
If majority of branches agree on risk level → proceed
If branches disagree → escalate for manual review
```

### Weighted Merge
Assign branch weights (Legal: 50%, Technical: 30%, Financial: 20%):
```
Final Score = (legal_score × 0.5) + (tech_score × 0.3) + (financial_score × 0.2)
```

### Fallback & Retry
If any branch fails, retry in sequence rather than failing entire workflow:
```
If branch fails → Retry branch individually
If retry still fails → Use cached result from previous execution
If no cache → Escalate to human review
```

---

## Monitoring & Analytics

### Key Metrics
- **Branch Success Rate (per branch):** Target ≥95%
- **Average Branch Latency:** Target ≤3.5 seconds
- **Parallelism Efficiency:** Actual time ÷ Sequential time (target ≥1.8x for 3 branches)
- **Execution Consistency:** Standard deviation of total time should be <0.5 seconds

### Dashboard Queries
```sql
SELECT 
  branch_name,
  COUNT(*) as executions,
  AVG(duration_ms) as avg_duration,
  STDDEV(duration_ms) as latency_variance,
  COUNT(CASE WHEN status='failed' THEN 1 END) / COUNT(*) as failure_rate
FROM workflow_executions
WHERE workflow_id = 'Agent_3_Parallel'
GROUP BY branch_name
```

---

## Support & Resources

- **n8n Parallel Execution:** [docs.n8n.io/workflows/parallel](https://docs.n8n.io/workflows)
- **Gemini API Guide:** [ai.google.dev](https://ai.google.dev)
- **n8n Community:** [community.n8n.io](https://community.n8n.io)

---

## License & Attribution

This workflow was built as part of the **BITSoM × Masai "Product Management with Generative & Agentic AI"** workshop series.

**Status:** Production-Ready | **Last Tested:** October 2026

---

## Changelog

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | Oct 2026 | Initial release; parallel execution pattern documented; verified >1.8x speedup vs. serial |

