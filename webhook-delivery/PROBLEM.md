# Webhook Delivery System

Design a system that reliably delivers events from internal platforms to external services via HTTP webhooks.

When an event occurs (e.g. payment completed, account updated), the system must send an HTTP POST to a URL registered by the downstream service. Delivery must be reliable — if the target is unavailable, retry. If retries are exhausted, alert the team.
