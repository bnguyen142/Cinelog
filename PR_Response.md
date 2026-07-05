# PR Response Doc — feature/watchlist

Responses to the six review comments left by `@jamjamgobambam` on
[PR #1](https://github.com/jamjamgobambam/ai201-project6-cinelog-starter/pull/1).

---

## 1. Naming convention (`save_to_watchlist` → `add_to_watchlist`)

> `save_to_watchlist()` should follow the project's naming convention. Compare
> with `add_to_collection()` — the pattern here is `verb_to_noun`. Please
> rename to `add_to_watchlist()` and update all call sites.

**Response:**
Renamed `save_to_watchlist()` to `add_to_watchlist()` in `services/watchlist_service.py`
and updated the one call site in `routes/watchlist/watchlist.py` to match the
project's `verb_to_noun` convention (`add_to_collection`, `remove_from_collection`).

**Commit:** _TODO — commit this as `refactor: rename save_to_watchlist to add_to_watchlist`_

---

## 2. Duplicate watchlist entries

> What happens if a user calls this with a film that's already on their
> watchlist? The current implementation would add a duplicate entry. Please
> handle this case.

**Response:**
_TODO — fill in after making the change._

**Commit:** _TODO_

---

## 3. Sort order (alphabetical vs. date added)

> I'd prefer watchlists to default to "date added" order rather than
> alphabetical. Most users want to see what they added recently. I'm open to
> discussion if you see it differently — but let's make a decision and
> document it.

**Decision:**
_TODO — your call: keep alphabetical, switch to date-added, or something else? Say why._

**Commit:** _TODO_

---

## 4. Default visibility (`public=True`)

> I notice watchlists default to `public=True`. We don't have a documented
> decision on default visibility for user lists. Before I can approve this, I
> need you to add a note to your PR description explaining your reasoning. I
> want to make sure we're being intentional here, not just inheriting a
> default.

**Decision:**
_TODO — your call: keep `public=True`, flip to `public=False`, or something else? Say why._

**Commit:** _TODO_

---

## 5. Missing test for nonexistent `film_id`

> Please add a test for the case where `film_id` doesn't exist in the
> database. Look at the existing tests in `test_collection.py` — the pattern
> is there.

**Response:**
_TODO — fill in after adding the test._

**Commit:** _TODO_

---

## 6. Rebase onto `main` (UUID refactor)

> A refactor merged to `main` that changed film IDs from integers to UUIDs.
> Your watchlist code still references integer IDs. Please rebase on `main`
> and update accordingly.

**Response:**
_TODO — fill in after rebasing._

**Commit:** _TODO_
