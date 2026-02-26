To monitor PT (Provisioned Throughput) vs. Spillover, we use the throughput_type label. In Vertex AI, the system categorizes traffic into three types:

dedicated: This is your reserved Provisioned Throughput (PT).

spillover: This is traffic that exceeded your PT and "spilled over" into the standard pay-as-you-go pool.

shared: Standard pay-as-you-go traffic (not related to a PT order).

The Alert: Spillover Request Ratio
This alert calculates the ratio of Spillover requests to PT (Dedicated) requests. If you have 100 successful PT requests and more than 20 Spillover requests, this alert will fire.
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

Alternative: Spillover Token Ratio (Recommended)
While the request count is what you asked for, PT is billed and limited by tokens, not requests. A single "spillover" request could be a massive 500k token document that costs significantly more than 10 small PT requests.

If you want to monitor the cost/capacity impact, use this Token-based version:
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

Why this works
The 5m Window: We use a [5m] rate because Provisioned Throughput in Vertex AI (2026) uses a flexible window (usually 1–5 minutes) to handle bursts. A shorter window like [1m] might cause "flapping" alerts.

The Division Logic: In PromQL, if the denominator (dedicated traffic) is zero, the result is dropped (it won't fire a false alarm). This is safer than a subtraction-based alert.

Resource Container: Explicitly filtering by project ensures that if you ever clone this alert to a different project, it won't accidentally aggregate metrics from both.

⚠️ Critical Note on "No Data"
If you have zero spillover right now, the metric throughput_type="spillover" might not exist in your project yet. This will cause the PQL editor to throw an error saying the metric/label is invalid.

The Fix: Force a small burst of traffic that exceeds your PT limit (or temporarily lower your PT limit in the console) to generate at least one spillover request. Once that happens, the label becomes "active" and the alert can be saved.
