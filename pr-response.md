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
to match the project's `verb_to_noun` convention (`add_to_collection`,
`remove_from_collection`). Used VS Code's "Rename Symbol" (F2) so the editor's
language server would update every reference automatically rather than relying
on manual find-and-replace. Confirmed no call sites were missed by running
`grep -rn "save_to_watchlist" .` afterward — the only remaining hits were the
quoted comment text in this doc, not live code. The one real call site was
`routes/watchlist/watchlist.py` (the import and the call inside `add_film()`),
and both updated correctly.

**Commit:** `refactor: rename save_to_watchlist to add_to_watchlist`

---

## 2. Duplicate watchlist entries

> What happens if a user calls this with a film that's already on their
> watchlist? The current implementation would add a duplicate entry. Please
> handle this case.

**Response:**
Modeled this on `add_to_collection()` in `services/collection_service.py`, which
queries for an existing `CollectionEntry` with the same `(user_id, film_id)`
before inserting, and raises `AlreadyInCollectionError` if one is found. Added
the watchlist equivalent: a new `AlreadyInWatchlistError` exception, and a
`WatchlistEntry.query.filter_by(user_id=user_id, film_id=film_id).first()`
check in `add_to_watchlist()` before creating the entry. If a match exists,
`add_to_watchlist()` now raises `AlreadyInWatchlistError` instead of silently
inserting a second row.

Verified manually first with `python -c "from services.watchlist_service import
add_to_watchlist, get_watchlist, AlreadyInWatchlistError"` to confirm the module
imports cleanly, and confirmed the existing suite (`test_collection.py`) still
passes with no regressions. Full behavioral verification comes from the
dedicated test in Comment 3 (`test_add_to_watchlist_duplicate_raises`), modeled
on `test_add_to_collection_duplicate_raises` in `test_collection.py`.

**Commit:** _TODO — commit as `fix: reject duplicate watchlist entries`_

---

## 3. Sort order (alphabetical vs. date added)

> I'd prefer watchlists to default to "date added" order rather than
> alphabetical. Most users want to see what they added recently. I'm open to
> discussion if you see it differently — but let's make a decision and
> document it.

**Decision:**
Agreed with the maintainer — switched to date-added, newest first
(`WatchlistEntry.date_added.desc()`), matching the pattern `get_collection()`
already uses for the collection feature.

The reasoning goes further than "most users want recency," though. This app
currently has no `remove_from_watchlist()` — nothing ever gets pruned from a
watchlist, only added. That makes alphabetical order actively worse over time:
as a list grows, new additions get buried alphabetically among everything
you've ever added, with no way to distinguish "I just decided I want this" from
"I added this eight months ago and forgot about it." Newest-first keeps the top
of the list meaningful regardless of how large the list grows, since the
most-recently-added film is also, almost by definition, one you haven't
watched yet — if you had, you wouldn't still need it on the watchlist.

Alphabetical does have a real use case — scanning/searching a large list for a
specific title — but that's a "browse mode," not the default view. A future
enhancement could let users toggle to alphabetical or oldest-first as an
option, but newest-first should remain the default.

Wrote `test_get_watchlist_returns_newest_first` in `tests/test_watchlist.py`
(mirroring `test_get_collection_returns_newest_first`) to verify the new order.
Running it surfaced an unrelated pre-existing bug: `Film` only declared a
relationship back to `CollectionEntry` (`backref="film"`), never to
`WatchlistEntry`, so `entry.film` inside `get_watchlist()` had never actually
worked — it just went unnoticed because no test had ever called
`get_watchlist()` before. Fixed by adding the missing
`watchlist_entries = db.relationship("WatchlistEntry", backref="film", lazy=True)`
to `Film` in `models.py`, mirroring the existing `collection_entries` line.
Ran `pytest tests/ -v` afterward — all 8 pass.

**Commits:**

- `fix: add missing Film-to-WatchlistEntry relationship`
- `fix: sort watchlist by date added instead of alphabetical`

---

## 4. Default visibility (`public=True`)

> I notice watchlists default to `public=True`. We don't have a documented
> decision on default visibility for user lists. Before I can approve this, I
> need you to add a note to your PR description explaining your reasoning. I
> want to make sure we're being intentional here, not just inheriting a
> default.

**Decision:**
Keeping `public=True` as the default.

CineLog's value proposition is social discovery: people use a watchlist app to
share what they want to watch and find new films through friends with similar
taste. If a film is on your public watchlist and a friend with overlapping
interests sees it, that's a real discovery path — arguably the main reason to
build a "list" feature at all instead of just personal notes. Defaulting to
`public=True` optimizes for that behavior happening automatically, without
requiring users to take an extra opt-in step that most of them would skip
(defaults are sticky — most users never change them).

The tradeoff we're accepting: some users will self-censor. If everything you
add is visible by default, you may not add a film you're embarrassed about
wanting to watch, which shrinks the pool of what actually gets logged. That's
a real cost — it's not free to pick `True`.

We're accepting that cost for now because:

1. Right now `public` is inert — no route or service function in this
   codebase actually filters or shares watchlist entries based on it. There's
   no "friends" feature yet, so the immediate downside is close to zero;
   we're setting a default in anticipation of a feature, not for one that
   exists today.
2. It's a reversible decision per-user going forward, not a one-way door: a
   natural next step is letting users flip individual entries to private, so
   people with something to hide can opt out instead of everyone having to
   opt in.
3. Public-by-default also means we have more watchlist data to build
   recommendations from later (e.g., "users who added X also added Y").
   Defaulting to private would mean most watchlists stay empty of signal
   until users deliberately opt in, which is a much smaller dataset to build
   suggestion features on top of.

**Commit:** _TODO — this is a docs-only change (PR description / pr-response.md), no code change needed since `public=True` was already the model's default._

---

## 5. Missing test for nonexistent `film_id`

> Please add a test for the case where `film_id` doesn't exist in the
> database. Look at the existing tests in `test_collection.py` — the pattern
> is there.

**Response:**
Created `tests/test_watchlist.py`, mirroring `tests/test_collection.py`'s fixtures
(`app`, `sample_user`, `sample_film`) and its three-test pattern from
`CONTRIBUTING.md` (happy path, duplicate/conflict, nonexistent ID). Used
`test_add_to_collection_nonexistent_film_raises` as the direct model for
`test_add_to_watchlist_nonexistent_film_raises` — same fake UUID
(`00000000-0000-0000-0000-000000000000`), same `pytest.raises(...)` structure,
swapped `FilmNotFoundError`'s source import from `collection_service`. Also
added `test_add_to_watchlist_creates_entry` (happy path) and
`test_add_to_watchlist_duplicate_raises` (verifies Comment 2's fix), so the
watchlist service now has the same three-test coverage as the collection service.

Ran `pytest tests/test_watchlist.py -v` — all 3 pass. Ran the full suite,
`pytest tests/ -v` — all 7 pass, confirming no regressions to `test_collection.py`.

**Commit:** _TODO — commit as `test: add watchlist tests for duplicate and nonexistent film cases`_

---

## 6. Rebase onto `main` (UUID refactor)

> A refactor merged to `main` that changed film IDs from integers to UUIDs.
> Your watchlist code still references integer IDs. Please rebase on `main`
> and update accordingly.

**Response:**
_TODO — fill in after rebasing._

**Commit:** _TODO_
