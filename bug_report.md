# CoWork Bug Report (Tie-Breaker Submission)

## 1. Timezone Offset Truncation
- **File:** `app/timeutils.py` (Line 12)
- **What the bug was & why it caused incorrect behavior:** `parse_input_datetime` stripped the `tzinfo` from incoming ISO 8601 strings without converting them to UTC first. If a client sent a time with a `+05:00` offset, it simply dropped the offset and saved the naive time, resulting in a fundamentally incorrect UTC representation (violating Rule 1).
- **How it was fixed:** Updated the logic to first convert the datetime to UTC using `dt.astimezone(timezone.utc)` before calling `.replace(tzinfo=None)`.

## 2. Duplicate Username Success Response
- **File:** `app/routers/auth.py` (Line 37)
- **What the bug was & why it caused incorrect behavior:** When registering, if a duplicate username was found within the organization, the code silently returned the existing user's data with a 200/201 response instead of returning an error, violating Rule 15.
- **How it was fixed:** Replaced the return statement with `raise AppError(409, "USERNAME_TAKEN", "Username already taken")`.

## 3. Refresh Token Reuse
- **File:** `app/routers/auth.py` (Line 83) & `app/auth.py` (Lines 85, 96)
- **What the bug was & why it caused incorrect behavior:** The `/auth/refresh` endpoint never checked if a refresh token was previously used, nor did it invalidate a token upon use. Furthermore, the `_revoked_tokens` logic in `auth.py` incorrectly verified the `sub` (user id) against the set, while it only inserted the `jti` (unique token ID) into the set upon logout, rendering all revocations completely useless. This violated Rule 8.
- **How it was fixed:** Fixed `app/auth.py` to check and revoke the token's `jti`. Then updated the `/refresh` endpoint to check if the incoming token's `jti` is revoked, and immediately added it to the revoked list after decoding it.

## 4. Back-to-Back Booking Conflict
- **File:** `app/routers/bookings.py` (Line 48)
- **What the bug was & why it caused incorrect behavior:** The conflict logic checked `b.start_time <= end and start <= b.end_time`. The `<= ` operator falsely flagged back-to-back bookings (e.g. 10:00-11:00 and 11:00-12:00) as overlapping, strictly forbidding them. Rule 3 explicitly allows back-to-back bookings.
- **How it was fixed:** Changed the condition to strictly less than (`<`).

## 5. Pagination Logic Flaws
- **File:** `app/routers/bookings.py` (Line 137)
- **What the bug was & why it caused incorrect behavior:** The `GET /bookings` endpoint hardcoded the limit to `10`, calculated the offset as `page * limit` (which skipped the entire first page), and sorted items by `desc()` instead of `asc()`. All of these violated Rule 11.
- **How it was fixed:** Corrected the chain to `.order_by(Booking.start_time.asc())`, `.offset((page - 1) * limit)`, and `.limit(limit)`.

## 6. Single Booking Payload Override
- **File:** `app/routers/bookings.py` (Line 166)
- **What the bug was & why it caused incorrect behavior:** `GET /bookings/{id}` overwrote the `start_time` field in the response with `booking.created_at`.
- **How it was fixed:** Removed the line `response["start_time"] = iso_utc(booking.created_at)`.

## 7. Cancellation Refund Tiers & Rounding
- **File:** `app/routers/bookings.py` (Line 196)
- **What the bug was & why it caused incorrect behavior:** The notice tier for 100% refund was set to `> 48` instead of `>= 48`. The lowest tier (< 24 hrs) wrongly granted a 50% refund instead of 0%. Furthermore, the refund rounding utilized Python's `round()`, which applies Banker's Rounding (to the nearest even number), while Rule 6 explicitly requested rounding half-cents up.
- **How it was fixed:** Corrected the conditions to `>= timedelta(hours=48)`, updated the else block to `refund_percent = 0`, and replaced `round()` with integer math: `int((booking.price_cents * refund_percent + 50) // 100)`.

## 8. Multi-Tenancy Data Leak in Admin Export
- **File:** `app/services/export.py` (Line 25)
- **What the bug was & why it caused incorrect behavior:** `fetch_bookings_raw(db, room_id)` queried only by `room_id`. When an admin requested an export with `include_all=true` and provided a `room_id` belonging to a different organization, the API cheerfully returned it, exposing cross-tenant data and violating Rule 9.
- **How it was fixed:** Updated `fetch_bookings_raw` to accept the admin's `org_id`, joined the `Room` table, and applied the filter `Room.org_id == org_id`. Updated `generate_export` to pass the `org_id` parameter.

## 9. Cache Invalidation Omissions
- **File:** `app/routers/bookings.py` (Lines 125, 218)
- **What the bug was & why it caused incorrect behavior:** When creating a booking, the usage report cache wasn't invalidated. When cancelling a booking, the room availability cache wasn't invalidated. This led to endpoints serving stale data (violating Rules 12 & 13 requiring immediate reflection of state).
- **How it was fixed:** Added `cache.invalidate_report(user.org_id)` to `create_booking` and `cache.invalidate_availability(booking.room_id, ...)` to `cancel_booking`.

## 10. Assorted Race Conditions & Hidden Deadlock
- **File:** Multiple (`app/routers/bookings.py`, `app/services/ratelimit.py`, `app/services/reference.py`, `app/services/stats.py`, `app/services/notifications.py`)
- **What the bug was & why it caused incorrect behavior:** Simulated network/I/O pauses (`time.sleep()`) were placed indiscriminately inside read-modify-write flows without synchronization mechanisms. Concurrent API requests could bypass the booking quota, double-book a room, grant double refunds, bypass rate-limiting buckets, generate identical reference codes, and corrupt room statistics. Finally, the `notifications.py` locks were acquired in reverse order (`email`->`audit` on creation, `audit`->`email` on cancellation), guaranteeing a deadlock if two requests happened simultaneously (violating Liveness Rule 16).
- **How it was fixed:** Introduced threading locks (`threading.Lock()`) across all affected files. 
  - Wrapped conflict/quota checks in `bookings.py` within `with _booking_lock`.
  - Shifted pauses outside critical sections in `stats.py`, `ratelimit.py`, and `reference.py` while wrapping state mutations in `with _lock`.
  - Reordered the lock acquisition in `notifications.py`'s `notify_cancelled` to strictly adhere to the `_email_lock` -> `_audit_lock` hierarchy to eliminate deadlocks.

## 11. Grace Window in Booking Start Time
- **File:** `app/routers/bookings.py` (Line 89)
- **What the bug was & why it caused incorrect behavior:** The code allowed a booking to be created even if its start time was in the past (up to 5 minutes ago) by checking `start <= now - timedelta(seconds=300)`. This violated Rule 2, which states there should be no grace window.
- **How it was fixed:** Removed the `timedelta` completely, changing the condition to `if start <= now:`.
