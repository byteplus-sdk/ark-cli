# Alerts and scheduling

`arkcli doctor monitor alert list` exits nonzero with alert_api_not_available. Listing, creation, mutation, deletion and scheduling are unavailable. Do not call invented commands or treat foreground watch as a persistent service.

For an alert request, prepare explicit endpoints, metric and units, lookback, threshold, persistence duration, missing-data policy, recovery condition and recipients. Label it a proposal awaiting implementation. Claim activation only with real write and readback evidence.

For scheduled checks, explain that the foreground CLI does not schedule work. Prepare task inputs/outputs; integrate an authorized host scheduler only if one exists, then record its ID, timezone and validation. Do not invent a created schedule.
