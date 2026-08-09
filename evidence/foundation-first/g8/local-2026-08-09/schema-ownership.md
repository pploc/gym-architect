# Database ownership (G8 business check SQL)

Member local compose DB name remains `gym_member` (G5-compatible); ownership rules match target `member_db`.

Asserted in `g8-business-check.sh` step `schema ownership proof`:

| DB | gym_locations | membership_plans | pending_purchases | subscription snapshots |
|---|---|---|---|---|
| plans_db | present | present | absent (not queried; Plans-only catalog) | n/a |
| gym_member (Member) | absent (count=0) | absent (count=0) | present | plan_type_snapshot, duration_days_snapshot, price_vnd_snapshot present |
| identity_db | absent | absent | absent | n/a |

No cross-service FK proof: Member stores opaque gym_id/plan_id strings only; catalog tables live only in plans_db.
