These two alerts are like looking at a highway from two different angles: one counts **the number of cars** (Requests), and the other counts **the size of the trucks** (Tokens).

To manage Gemini 2.5 Pro effectively in **Project1**, you should ideally use both. Here is the breakdown for your documentation and technical understanding.

---

### 1. Spillover Request Ratio (Frequency Metric)

This alert tracks **how often** your users are being pushed out of the "Dedicated" lane into the "Pay-as-you-go" lane.

* **Logic:** (Number of Spillover Requests) / (Number of Dedicated Requests)
* **What it tells you:** "20% of our total interactions are currently overflowing."
* **Best For:** User Experience. If spillover requests have higher latency (which they often do in 2026 due to shared pool contention), this tells you that 1 in 5 users might be having a slower experience.
* **The Documentation Entry:**
> **Alert Name:** `[Project1] [Warning] Gemini-2.5-Pro | High Spillover Request Frequency (>20%)`
> **Description:** This alert fires when more than 20% of individual model calls are being handled by Pay-as-you-go instead of Provisioned Throughput.
> **Action:** Check if a specific user or service is "spamming" the endpoint with many small requests.



---

### 2. Spillover Token Ratio (Volume & Cost Metric)

This is the **Recommended** alert because Provisioned Throughput (PT) is sold in **Tokens per second**, not requests.

* **Logic:** (Total Spillover Tokens) / (Total Dedicated Tokens)
* **What it tells you:** "20% of our actual compute volume is overflowing."
* **Best For:** Budget and Capacity Planning. One single request with a 1-million-token PDF could overflow your entire capacity, even if 99% of your other requests stay in the Dedicated lane.
* **The Documentation Entry:**
> **Alert Name:** `[Project1] [Critical] Gemini-2.5-Pro | High Spillover Token Volume (>20%)`
> **Description:** This alert monitors the actual data volume (tokens). If tokens are spilling over, it means our GSUs (Generative AI Scale Units) are physically maxed out.
> **Action:** This is a financial signal. We are paying a premium for these overflow tokens. Consider purchasing more GSUs or implementing "context caching" to reduce token burn.



---

### 📊 Comparison Table

| Feature | Request Ratio Alert | Token Ratio Alert (Recommended) |
| --- | --- | --- |
| **Unit** | Individual API Calls (Count) | Model Data Units (Tokens/Characters) |
| **Primary Risk** | User Latency / Experience | **Direct Cloud Billing / Costs** |
| **Sensitivity** | High for "chatty" apps (many small pings) | High for "heavy" apps (long docs/video) |
| **Why it fires** | Too many people clicking "Send" at once. | The prompts/responses are too long for our GSU limit. |

---

### Can I use both? (The "Two-Lens" Approach)

**Yes, and you should.** Using both gives you a complete picture:

1. **If ONLY Requests fire:** You have many users sending tiny messages (e.g., "Hello", "Thanks"). You might not be spending much money on spillover, but your PT overhead is high.
2. **If ONLY Tokens fire:** You have a few users sending massive datasets. Your costs are spiking, even though your "Request Count" looks normal.
3. **If BOTH fire:** Your system is genuinely undersized for its current popularity. **Time to buy more GSUs.**

### Final Alert Naming Recommendation

For your **Project1** setup, I would name them like this to tell them apart instantly in a Slack/Email notification:

* **Request-based:** `Project1 | Gemini-2.5-Pro | Spillover-REQ-Ratio > 20%`
* **Token-based:** `Project1 | Gemini-2.5-Pro | Spillover-TOKEN-Volume > 20%`

**Would you like me to provide the exact PromQL for the "Token Ratio" version so you can paste it into the Cloud Monitoring editor?**


## To monitor PT (Provisioned Throughput) vs. Spillover, we use the throughput_type label. In Vertex AI, the system categorizes traffic into three types:

# dedicated: This is your reserved Provisioned Throughput (PT).

# spillover: This is traffic that exceeded your PT and "spilled over" into the standard pay-as-you-go pool.

# shared: Standard pay-as-you-go traffic (not related to a PT order).

## The Alert: Spillover Request Ratio
# This alert calculates the ratio of Spillover requests to PT (Dedicated) requests. If you have 100 successful PT requests and more than 20 Spillover requests, this alert will fire.

(
  sum(rate(aiplatform_googleapis_com:publisher_online_serving_model_invocation_count{
    throughput_type="spillover", 
    model_user_id="gemini-2.5-pro", 
    resource_container="YOUR_PROJECT_ID"
  }[5m]))
  /
  sum(rate(aiplatform_googleapis_com:publisher_online_serving_model_invocation_count{
    throughput_type="dedicated", 
    model_user_id="gemini-2.5-pro", 
    resource_container="YOUR_PROJECT_ID"
  }[5m]))
) > 0.20

## Alternative: Spillover Token Ratio (Recommended)
# While the request count is what you asked for, PT is billed and limited by tokens, not requests. A single "spillover" request could be a massive 500k token document that costs significantly more than 10 small PT requests.

# If you want to monitor the cost/capacity impact, use this Token-based version:
(
  sum(rate(aiplatform_googleapis_com:publisher_online_serving_consumed_token_throughput{
    throughput_type="spillover", 
    model_user_id="gemini-2.5-pro", 
    resource_container="YOUR_PROJECT_ID"
  }[5m]))
  /
  sum(rate(aiplatform_googleapis_com:publisher_online_serving_consumed_token_throughput{
    throughput_type="dedicated", 
    model_user_id="gemini-2.5-pro", 
    resource_container="YOUR_PROJECT_ID"
  }[5m]))
) > 0.20


## Why this works
# The 5m Window: We use a [5m] rate because Provisioned Throughput in Vertex AI (2026) uses a flexible window (usually 1–5 minutes) to handle bursts. A shorter window like [1m] might cause "flapping" alerts.

# The Division Logic: In PromQL, if the denominator (dedicated traffic) is zero, the result is dropped (it won't fire a false alarm). This is safer than a subtraction-based alert.

# Resource Container: Explicitly filtering by project ensures that if you ever clone this alert to a different project, it won't accidentally aggregate metrics from both.

## ⚠️ Critical Note on "No Data"
# If you have zero spillover right now, the metric throughput_type="spillover" might not exist in your project yet. This will cause the PQL editor to throw an error saying the metric/label is invalid.

# The Fix: Force a small burst of traffic that exceeds your PT limit (or temporarily lower your PT limit in the console) to generate at least one spillover request. Once that happens, the label becomes "active" and the alert can be saved.

## Alert Naming
# Category 1: GSU Burndown & Capacity Alerts
# Focus: Monitoring how close you are to hitting your reserved Provisioned Throughput (PT) limits.

# GSU Capacity > 95% | Gemini-2.5-Pro

SRE-Alert | Model-GSU | Burndown-Threshold-Exceeded | project1

Gemini-GSU-Capacity | WARNING | 80%-Saturation-Detected

Model-Throughput | Provisioned-GSU-Near-Limit | High-Severity

Vertex-Observability | GSU-Utilization-Spike | Gemini-2.5

AI-Infra | Gemini-2.5-Pro | Provisioned-Throughput-Exhaustion

Quota-Alert | Vertex-GSU-Burndown | Sustained-High-Usage

Gemini-Service | Capacity-Planning-Required | GSU-90-Percent

SRE | Gemini-2.5-Pro | GSU-Quota-Approaching-Critical

Vertex-Engine | Token-Throughput | Provisioned-Limit-Warning

## Category 2: PT vs. Spillover Ratio Alerts
# Focus: Monitoring when costs might spike because traffic is "spilling over" from your fixed-price PT into the expensive pay-as-you-go pool.

# Spillover-Ratio > 20% | Gemini-2.5-Pro -

Gemini-Traffic | Ratio-Alert | Spillover-Exceeds-PT-Buffer

FinOps | High-Spillover-Detected | Gemini-2.5-Pro | project1

SRE-Alert | Model-Traffic-Anomaly | Spillover-to-Dedicated-Ratio

Vertex-Endpoint | Spillover-Volume-High | PT-Capacity-Exceeded

Gemini-GSU | Spillover-Rate-Critical | 20-Percent-Threshold

AI-Ops | Cost-Efficiency | Excessive-Spillover-Traffic

Model-Scaling | Warning | PT-Insufficient-Spillover-Triggered

Vertex-Observability | Gemini-2.5 | Spillover-vs-Dedicated-Mismatch

Billing-Guardrail | Spillover-Utilization-Spike | High-Impact
