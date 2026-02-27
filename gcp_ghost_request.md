The "Ghost Request" Alert (Client-Side Cancellations)
Why add it: In Gemini applications, a high rate of 499 errors (Client Closed Request) usually means your model is responding so slowly that your users (or your frontend timeout) are giving up and hitting refresh.

Alert Name: [Project1] High Client-Cancel Rate (>10%) - Gemini-2.5-Pro

Code snippet
(
  sum(rate(aiplatform_googleapis_com:publisher_online_serving_model_invocation_count{
    response_code="499", 
    model_user_id="gemini-2.5-pro", 
    resource_container="project1"
  }[15m]))
  /
  sum(rate(aiplatform_googleapis_com:publisher_online_serving_model_invocation_count{
    model_user_id="gemini-2.5-pro", 
    resource_container="project1"
  }[15m]))
) > 0.10
