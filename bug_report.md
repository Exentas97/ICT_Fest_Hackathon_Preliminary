## Bug 1: Timezone Offset Stripped Without UTC Conversion

### What is the main problem
When an API client sends a datetime with a non-UTC timezone offset, the application discards the offset instead of converting the time value to UTC, silently storing the wrong timestamp.

### Why it is a problem according to the business rules
Business Rule 1 states that all input datetimes carrying a UTC offset must be converted to UTC before storage or comparison. Stripping the offset without conversion means a booking sent as 18:00+06:00 is stored as 18:00 UTC, which is six hours off. Every downstream comparison, conflict check, and quota window will operate on a corrupted timestamp.

### Which code was changed in which file
File: `app/timeutils.py`, function `parse_input_datetime`, approximate line 11.

### Exact changes
Before Code:
```python
    dt = datetime.fromisoformat(value)
    if dt.tzinfo is not None:
        dt = dt.replace(tzinfo=None)
    return dt
```
After Code:
```python
    dt = datetime.fromisoformat(value)
    if dt.tzinfo is not None:
        dt = dt.astimezone(timezone.utc).replace(tzinfo=None)
    return dt
```

---

## Bug 2: Access Token Expiry Calculated in Minutes Instead of Seconds

### What is the main problem
The access token lifetime is computed by passing the product of `ACCESS_TOKEN_EXPIRE_MINUTES * 60` to the `minutes` parameter of `timedelta`, producing a token that lives for 900 minutes rather than 900 seconds.

### Why it is a problem according to the business rules
Business Rule 8 requires that the difference between `exp` and `iat` in every access token is exactly 900 seconds. A grader asserting `exp - iat == 900` will fail on every single token issued by the broken code.

### Which code was changed in which file
File: `app/auth.py`, function `create_access_token`, approximate line 50.

### Exact changes
Before Code:
```python
    lifetime = timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES * 60)
```
After Code:
```python
    lifetime = timedelta(seconds=ACCESS_TOKEN_EXPIRE_MINUTES * 60)
```

---

## Bug 3: Token Revocation Checks User ID Instead of Token ID

### What is the main problem
The logout blacklist stores unique token identifiers (`jti`), but the validation gate checks whether the user's database ID (`sub`) appears in that blacklist, so revoked tokens are never actually blocked.

### Why it is a problem according to the business rules
Business Rule 8 states that logout must immediately invalidate the presented access token for all further use. Because the wrong field is compared, any logged-out token continues to pass authentication, allowing terminated sessions to access protected endpoints indefinitely.

### Which code was changed in which file
File: `app/auth.py`, function `get_token_payload`, approximate line 97.

### Exact changes
Before Code:
```python
    if payload.get("sub") in _revoked_tokens:
        raise AppError(401, "UNAUTHORIZED", "Token has been revoked")
```
After Code:
```python
    with _token_lock:
        is_revoked = payload.get("jti") in _revoked_tokens
    if is_revoked:
        raise AppError(401, "UNAUTHORIZED", "Token has been revoked")
```

---

## Bug 4: Duplicate Username Registration Returns Success Instead of Conflict

### What is the main problem
When a caller registers a username that already exists within an organization, the endpoint silently returns the existing user record with a `201 Created` status instead of rejecting the request.

### Why it is a problem according to the business rules
Business Rule 15 explicitly states that a duplicate username within an org must return `409 USERNAME_TAKEN`. The broken code lets an attacker enumerate existing users and bypasses the error contract that the grader will assert.

### Which code was changed in which file
File: `app/routers/auth.py`, function `register`, approximate line 37.

### Exact changes
Before Code:
```python
    if existing is not None:
        return {
            "user_id": existing.id,
            "org_id": org.id,
            "username": existing.username,
            "role": existing.role,
        }
```
After Code:
```python
    if existing is not None:
        raise AppError(409, "USERNAME_TAKEN", "Username already taken")
```

---

## Bug 5: Refresh Tokens Can Be Reused Unlimited Times

### What is the main problem
The token refresh endpoint decodes the incoming refresh token and issues new tokens without recording the presented token as consumed, allowing any refresh token to be replayed indefinitely until it expires.

### Why it is a problem according to the business rules
Business Rule 8 states that refresh tokens are single-use: once presented to `POST /auth/refresh`, the token must be invalidated and any subsequent reuse must return `401`. The broken code allows session hijacking through token replay.

### Which code was changed in which file
File: `app/routers/auth.py`, function `refresh`, approximate line 84. A thread-safe `check_and_revoke_token` helper was also added to `app/auth.py` at approximately line 24.

### Exact changes
Before Code:
```python
    data = decode_token(payload.refresh_token)
    if data.get("type") != "refresh":
        raise AppError(401, "UNAUTHORIZED", "Wrong token type")
    user = db.query(User).filter(User.id == int(data["sub"])).first()
```
After Code:
```python
    data = decode_token(payload.refresh_token)
    if data.get("type") != "refresh":
        raise AppError(401, "UNAUTHORIZED", "Wrong token type")
    if not check_and_revoke_token(data.get("jti")):
        raise AppError(401, "UNAUTHORIZED", "Token has been revoked")
    user = db.query(User).filter(User.id == int(data["sub"])).first()
```

---

## Bug 6: A Five-Minute Grace Window Allows Past Bookings

### What is the main problem
The booking creation endpoint uses `start <= now - timedelta(seconds=300)` as its past-time guard, which means any start time within the last five minutes silently passes validation.

### Why it is a problem according to the business rules
Business Rule 2 states that `start_time` must be strictly in the future at request time with no grace window of any size. The grader will submit requests with a start time one second in the past and expect a `400 INVALID_BOOKING_WINDOW` response; the broken code returns `201` instead.

### Which code was changed in which file
File: `app/routers/bookings.py`, function `create_booking`, approximate line 86.

### Exact changes
Before Code:
```python
    if start <= now - timedelta(seconds=300):
        raise AppError(400, "INVALID_BOOKING_WINDOW", "start_time must be in the future")
```
After Code:
```python
    if start <= now:
        raise AppError(400, "INVALID_BOOKING_WINDOW", "start_time must be in the future")
```

---

## Bug 7: Zero-Duration and Sub-Minimum Bookings Are Not Rejected

### What is the main problem
The duration validator only checks whether the duration exceeds the eight-hour maximum and never verifies that it is at least one hour or that `end_time` is actually after `start_time`.

### Why it is a problem according to the business rules
Business Rule 2 requires a whole-number duration of at minimum 1 hour, a maximum of 8 hours, and that `end_time` is strictly after `start_time`. The broken code allows zero-hour and negative-hour bookings to be committed to the database.

### Which code was changed in which file
File: `app/routers/bookings.py`, function `create_booking`, approximate line 89.

### Exact changes
Before Code:
```python
    duration_hours = (end - start).total_seconds() / 3600
    if duration_hours != int(duration_hours):
        raise AppError(400, "INVALID_BOOKING_WINDOW", "duration must be a whole number of hours")
    duration_hours = int(duration_hours)
    if duration_hours > MAX_DURATION_HOURS:
        raise AppError(400, "INVALID_BOOKING_WINDOW", "duration out of range")
```
After Code:
```python
    duration_hours = (end - start).total_seconds() / 3600
    if duration_hours != int(duration_hours):
        raise AppError(400, "INVALID_BOOKING_WINDOW", "duration must be a whole number of hours")
    duration_hours = int(duration_hours)
    if duration_hours < MIN_DURATION_HOURS or duration_hours > MAX_DURATION_HOURS:
        raise AppError(400, "INVALID_BOOKING_WINDOW", "duration out of range")
```

---

## Bug 8: Members Can Read Other Members' Booking Details

### What is the main problem
The single-booking detail endpoint only verifies that the booking's room belongs to the caller's organization, but does not verify that a non-admin caller owns the booking.

### Why it is a problem according to the business rules
Business Rule 10 states that members may read only their own bookings, and that presenting another member's booking ID must behave as non-existent and return `404 BOOKING_NOT_FOUND`. The broken code exposes full booking details of any organization member to any other member.

### Which code was changed in which file
File: `app/routers/bookings.py`, function `get_booking`, approximate line 150.

### Exact changes
Before Code:
```python
    booking = (
        db.query(Booking)
        .join(Room, Booking.room_id == Room.id)
        .filter(Booking.id == booking_id, Room.org_id == user.org_id)
        .first()
    )
    if booking is None:
        raise AppError(404, "BOOKING_NOT_FOUND", "Booking not found")
```
After Code:
```python
    booking = (
        db.query(Booking)
        .join(Room, Booking.room_id == Room.id)
        .filter(Booking.id == booking_id, Room.org_id == user.org_id)
        .first()
    )
    if booking is None:
        raise AppError(404, "BOOKING_NOT_FOUND", "Booking not found")
    if user.role != "admin" and booking.user_id != user.id:
        raise AppError(404, "BOOKING_NOT_FOUND", "Booking not found")
```

---

## Bug 9: Booking Detail Response Returns Creation Timestamp as Start Time

### What is the main problem
After correctly serializing all booking fields, the detail endpoint overwrites the `start_time` key in the response dictionary with the booking's `created_at` value before returning.

### Why it is a problem according to the business rules
The API contract for `GET /bookings/{id}` specifies that `start_time` is a required field representing when the room reservation begins. The grader compares the returned `start_time` against the value submitted during booking creation; the broken code returns the wrong timestamp every time.

### Which code was changed in which file
File: `app/routers/bookings.py`, function `get_booking`, approximate line 166.

### Exact changes
Before Code:
```python
    response = serialize_booking(booking)
    response["start_time"] = iso_utc(booking.created_at)
```
After Code:
```python
    response = serialize_booking(booking)
```

---

## Bug 10: Refund Tier Boundaries and the Zero-Refund Case Are Both Wrong

### What is the main problem
The cancellation logic uses `notice_hours > 48` (exclusive) instead of `notice >= timedelta(hours=48)` (inclusive) for the full-refund tier, and the final `else` branch assigns 50% instead of 0% to cancellations with less than 24 hours notice.

### Why it is a problem according to the business rules
Business Rule 6 defines three explicit tiers: notice of 48 hours or more earns 100%, notice between 24 and 48 hours earns 50%, and notice below 24 hours earns 0%. The broken code gives a booking cancelled exactly 48 hours early only a 50% refund, and gives every last-minute cancellation a 50% refund instead of zero, inflating refunded amounts across the board.

### Which code was changed in which file
File: `app/routers/bookings.py`, function `cancel_booking`, approximate line 198.

### Exact changes
Before Code:
```python
    now = datetime.utcnow()
    notice = booking.start_time - now
    notice_hours = int(notice.total_seconds() // 3600)
    if notice_hours > 48:
        refund_percent = 100
    elif notice >= timedelta(hours=24):
        refund_percent = 50
    else:
        refund_percent = 50
```
After Code:
```python
    now = datetime.utcnow()
    notice = booking.start_time - now
    if notice >= timedelta(hours=48):
        refund_percent = 100
    elif notice >= timedelta(hours=24):
        refund_percent = 50
    else:
        refund_percent = 0
```

---

## Bug 11: Refund Amounts Are Calculated with Mismatched Rounding

### What is the main problem
The cancellation response computes `refund_amount_cents` using Python's banker's rounding (`round()`), while the database ledger entry in `log_refund` computes the same amount using integer truncation (`int()`), meaning the two values diverge for any booking whose price produces a half-cent result.

### Why it is a problem according to the business rules
Business Rule 6 requires half-cents to round up (arithmetic rounding), and states explicitly that the amount returned in the cancel response must equal the amount stored in the RefundLog. The grader will read both and assert equality; the broken code produces two different numbers for odd-cent prices like 1001.

### Which code was changed in which file
File: `app/routers/bookings.py`, function `cancel_booking`, approximate line 208. File: `app/services/refunds.py`, function `log_refund`, approximate line 15.

### Exact changes
Before Code (`app/routers/bookings.py`):
```python
    refund_amount_cents = round(booking.price_cents * (refund_percent / 100.0))
```
Before Code (`app/services/refunds.py`):
```python
    dollars = booking.price_cents / 100.0
    refund_dollars = dollars * (percent / 100.0)
    amount_cents = int(refund_dollars * 100)
```
After Code (`app/routers/bookings.py`):
```python
    refund_amount_cents = int(booking.price_cents * (refund_percent / 100.0) + 0.5)
```
After Code (`app/services/refunds.py`):
```python
    amount_cents = int(booking.price_cents * (percent / 100.0) + 0.5)
```

---

## Bug 12: Pagination Is Sorted Backwards, Offset Skips the First Page and Limit Is Hardcoded

### What is the main problem
The booking list endpoint sorts by descending start time, computes the SQL offset as `page * limit` instead of `(page - 1) * limit`, and ignores the caller's `limit` parameter entirely by hardcoding `.limit(10)`.

### Why it is a problem according to the business rules
Business Rule 11 requires items sorted by ascending `start_time` (ties broken by ascending `id`), page N at limit L returning items at index `(N-1)*L` through `N*L - 1`, with no items skipped or repeated across sequential pages. All three defects in the broken code violate this rule simultaneously.

### Which code was changed in which file
File: `app/routers/bookings.py`, function `list_bookings`, approximate line 128.

### Exact changes
Before Code:
```python
    base = db.query(Booking).filter(Booking.user_id == user.id)
    total = base.count()
    items = (
        base.order_by(Booking.start_time.desc(), Booking.id.asc())
        .offset(page * limit)
        .limit(10)
        .all()
    )
```
After Code:
```python
    base = db.query(Booking).filter(Booking.user_id == user.id)
    total = base.count()
    items = (
        base.order_by(Booking.start_time.asc(), Booking.id.asc())
        .offset((page - 1) * limit)
        .limit(limit)
        .all()
    )
```

---

## Bug 13: Admin CSV Export Leaks Bookings Across Organization Boundaries

### What is the main problem
When `include_all=true` and a `room_id` is provided, the export function fetches all bookings for that room directly by its database ID without verifying that the room belongs to the requesting administrator's organization.

### Why it is a problem according to the business rules
Business Rule 9 requires that all data operations, including administrative exports, are strictly scoped to the caller's organization, and that cross-org resource IDs must behave as non-existent (returning `404 ROOM_NOT_FOUND`). The broken code allows an admin to download the full booking history of any room in the system.

### Which code was changed in which file
File: `app/routers/admin.py`, function `export`, approximate line 72. Also required importing `Room` and `AppError` into the router.

### Exact changes
Before Code:
```python
    csv_body = generate_export(db, admin.org_id, admin.id, room_id, include_all)
    return Response(content=csv_body, media_type="text/csv")
```
After Code:
```python
    if room_id is not None:
        room = db.query(Room).filter(Room.id == room_id, Room.org_id == admin.org_id).first()
        if room is None:
            raise AppError(404, "ROOM_NOT_FOUND", "Room not found")
    csv_body = generate_export(db, admin.org_id, admin.id, room_id, include_all)
    return Response(content=csv_body, media_type="text/csv")
```

---

## Bug 14: Reference Code Counter Has No Concurrency Protection

### What is the main problem
The booking reference code generator reads the current counter value, pauses for 120 milliseconds, then increments and writes it back, with no lock to prevent two simultaneous requests from reading the same starting value.

### Why it is a problem according to the business rules
Business Rule 7 requires that every booking's `reference_code` is unique, including under concurrent creation. Two simultaneous booking requests will both read the same counter value during the sleep and produce identical codes such as `CW-001000`, violating the uniqueness guarantee.

### Which code was changed in which file
File: `app/services/reference.py`, function `next_reference_code`, approximate line 17.

### Exact changes
Before Code:
```python
def next_reference_code() -> str:
    current = _counter["value"]
    _format_pause()
    _counter["value"] = current + 1
    return f"CW-{current:06d}"
```
After Code:
```python
import threading
_ref_lock = threading.Lock()

def next_reference_code() -> str:
    with _ref_lock:
        current = _counter["value"]
        _format_pause()
        _counter["value"] = current + 1
        return f"CW-{current:06d}"
```

---

## Bug 15: Notification Locks Are Acquired in Opposite Order, Causing Deadlock

### What is the main problem
`notify_created` acquires the email lock then the audit lock, while `notify_cancelled` acquires them in the reverse order — audit first, then email — creating a classic circular dependency that will deadlock the API when a booking creation and a booking cancellation are processed concurrently.

### Why it is a problem according to the business rules
Business Rule 16 states that the service must respond to all endpoints at all times and that no combination of concurrent valid requests may hang the service. Two concurrent requests hitting `POST /bookings` and `POST /bookings/{id}/cancel` will mutually block each other forever, violating the liveness guarantee.

### Which code was changed in which file
File: `app/services/notifications.py`, function `notify_cancelled`, approximate line 31.

### Exact changes
Before Code:
```python
def notify_cancelled(booking) -> None:
    with _audit_lock:
        _write_audit("cancelled", booking)
        with _email_lock:
            _send_email("cancelled", booking)
```
After Code:
```python
def notify_cancelled(booking) -> None:
    with _email_lock:
        _send_email("cancelled", booking)
        with _audit_lock:
            _write_audit("cancelled", booking)
```

---

## Bug 16: Rate Limit Buckets Are Updated Without Concurrency Protection

### What is the main problem
The rolling-window rate limiter reads a user's request bucket from a shared dictionary, trims it, sleeps for 100 milliseconds, appends the new timestamp, and writes it back — all without a lock — so concurrent requests see a stale bucket and do not count against each other.

### Why it is a problem according to the business rules
Business Rule 5 limits `POST /bookings` to exactly 20 requests per rolling 60-second window per user, and explicitly states this must hold under concurrent requests. The broken code allows a user to fire dozens of simultaneous requests and each one will observe an empty or near-empty bucket, bypassing the limit entirely.

### Which code was changed in which file
File: `app/services/ratelimit.py`, function `record_and_check`, approximate line 18.

### Exact changes
Before Code:
```python
def record_and_check(user_id: int) -> None:
    now = time.time()
    bucket = _buckets.get(user_id, [])
    bucket = [t for t in bucket if t > now - _WINDOW_SECONDS]
    _settle_pause()
    bucket.append(now)
    _buckets[user_id] = bucket
    if len(bucket) > _MAX_REQUESTS:
        raise AppError(429, "RATE_LIMITED", "Too many booking requests")
```
After Code:
```python
import threading
_limit_lock = threading.Lock()

def record_and_check(user_id: int) -> None:
    with _limit_lock:
        now = time.time()
        bucket = _buckets.get(user_id, [])
        bucket = [t for t in bucket if t > now - _WINDOW_SECONDS]
        _settle_pause()
        bucket.append(now)
        _buckets[user_id] = bucket
        if len(bucket) > _MAX_REQUESTS:
            raise AppError(429, "RATE_LIMITED", "Too many booking requests")
```

---

## Bug 17: Room Statistics Are Updated Without Concurrency Protection

### What is the main problem
The in-memory room statistics tracker reads the current count and revenue, pauses for 100 milliseconds, then writes back the incremented or decremented values — with no lock — causing concurrent bookings or cancellations to overwrite each other's updates.

### Why it is a problem according to the business rules
Business Rule 14 states that `GET /rooms/{id}/stats` must always return values exactly equal to those derivable from the confirmed bookings in the database. Under concurrent load the broken read-modify-write race drops updates, so the stats endpoint permanently diverges from the actual booking totals stored in the database.

### Which code was changed in which file
File: `app/services/stats.py`, functions `record_create` and `record_cancel`, approximate line 15.

### Exact changes
Before Code:
```python
def record_create(room_id: int, price_cents: int) -> None:
    current = _stats.get(room_id, {"count": 0, "revenue": 0})
    count, revenue = current["count"], current["revenue"]
    _aggregate_pause()
    _stats[room_id] = {"count": count + 1, "revenue": revenue + price_cents}


def record_cancel(room_id: int, price_cents: int) -> None:
    current = _stats.get(room_id, {"count": 0, "revenue": 0})
    count, revenue = current["count"], current["revenue"]
    _aggregate_pause()
    _stats[room_id] = {"count": max(0, count - 1), "revenue": revenue - price_cents}
```
After Code:
```python
import threading
_stats_lock = threading.Lock()

def record_create(room_id: int, price_cents: int) -> None:
    with _stats_lock:
        current = _stats.get(room_id, {"count": 0, "revenue": 0})
        count, revenue = current["count"], current["revenue"]
        _aggregate_pause()
        _stats[room_id] = {"count": count + 1, "revenue": revenue + price_cents}


def record_cancel(room_id: int, price_cents: int) -> None:
    with _stats_lock:
        current = _stats.get(room_id, {"count": 0, "revenue": 0})
        count, revenue = current["count"], current["revenue"]
        _aggregate_pause()
        _stats[room_id] = {"count": max(0, count - 1), "revenue": revenue - price_cents}
```
