# Purchase snapshot + replay proof

From G8 business check after fake payment completion:

- pending_purchases.status: PENDING → COMPLETED
- pending_purchases.price_vnd_snapshot: 450000 (initiation-time Plans terms)
- subscriptions.status: ACTIVE
- subscriptions.plan_type_snapshot: MONTHLY
- subscriptions.duration_days_snapshot: 30
- subscriptions.price_vnd_snapshot: 450000

Replay:
- POST fake-payment /replay with same payment_id
- subscription count for (member_id, gym_id) remains 1
- pending_purchases.status remains COMPLETED

Catalog edit after activation:
- Plans membership_plans.price_vnd updated to 999999
- Member subscription and pending price_vnd_snapshot remain 450000
- Proves completion/lifecycle never reread live catalog price
