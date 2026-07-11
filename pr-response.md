# PR Response Doc — feature/watchlist

Responses to the six review comments left by `@jamjamgobambam` on
[PR #1](https://github.com/jamjamgobambam/ai201-project6-cinelog-starter/pull/1).

---

## AI Usage

Used Claude (claude.ai) for two purposes during this project.

First, codebase orientation. Starting from `app.py` and working through models, services, routes, and tests, I asked questions at each layer to confirm my understanding of how the entire codebase runs together — how the Flask factory wires up blueprints, what the SQLAlchemy relationships do in `models.py`, how the service layer separates business logic from routes, and how the test fixtures set up an isolated in-memory database. This gave me enough context to recognize the existing patterns (naming conventions, deduplication approach, test structure) before reading any of the review comments.

Second, as a second reviewer to catch things I missed. For example, during the interactive rebase I accidentally put the new commit message as a comment (`#`) in the todo list instead of in the actual commit message editor — the commit appeared to succeed but the message didn't change. Using Claude as a reviewer caught this immediately when checking the git log, and I was able to redo the rebase correctly.

AI was used to build understanding and catch mistakes — not to write code or generate the design decision responses.

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

**Commit:** `fix: reject duplicate watchlist entries`

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

1. While callers can now set `public=False` explicitly via the visibility toggle, the field is still inert in terms of filtering or sharing — no route surfaces watchlist entries to other users yet. The default optimizes for the social feature when it does land, without forcing users to opt in retroactively.
2. It's a reversible decision per-user going forward, not a one-way door: a
   natural next step is letting users flip individual entries to private, so
   people with something to hide can opt out instead of everyone having to
   opt in.
3. Public-by-default also means we have more watchlist data to build
   recommendations from later (e.g., "users who added X also added Y").
   Defaulting to private would mean most watchlists stay empty of signal
   until users deliberately opt in, which is a much smaller dataset to build
   suggestion features on top of.

As a stretch improvement, an optional `public` parameter was added to the `POST /watchlist/<user_id>/add` endpoint so callers can explicitly override the default per entry — giving users who want privacy an immediate opt-out without changing the default behavior for everyone else.

**Commit:** `docs: document default visibility decision for watchlist`

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

Ran `pytest tests/test_watchlist.py -v` — all 4 pass. Ran the full suite,
`pytest tests/ -v` — all 8 pass, confirming no regressions to `test_collection.py`.

**Commit:** `test: add watchlist tests for duplicate and nonexistent film cases`

---

## 6. Rebase onto `main` (UUID refactor)

> A refactor merged to `main` that changed film IDs from integers to UUIDs.
> Your watchlist code still references integer IDs. Please rebase on `main`
> and update accordingly.

**Response:**
Ran `git fetch origin` then `git rebase origin/main`. Git reported a successful rebase with no textual conflicts — but a manual inspection of `models.py` afterward revealed a **silent semantic conflict**: the `WatchlistEntry` class was missing. The UUID refactor on `main` had changed `models.py` enough that when git replayed the original "added watchlist model and endpoint" commit, the `WatchlistEntry` class didn't apply cleanly. Because there was no direct line-by-line text clash, git didn't pause to ask for resolution — it just silently dropped the class. The `watchlist_entries` relationship on `Film` survived because it was added in a later separate commit.

Fixed by adding `WatchlistEntry` back to `models.py` with `film_id = db.Column(db.String(36), ...)` — updated from the original `Integer` to `String(36)` to match the UUID-based `Film.id` after the refactor. Ran `pytest tests/ -v` to confirm all 8 tests pass.

**What conflicted:** `WatchlistEntry.film_id` was `db.Column(db.Integer, ...)` on the feature branch; main's refactor changed `Film.id` to `db.Column(db.String(36), ...)`. The resolution was updating `film_id` in `WatchlistEntry` to `String(36)` to match.

**Commit:** `fix: restore WatchlistEntry model with UUID film_id after rebase`

---

## PR Description

### Watchlist Feature

Adds a watchlist to CineLog so users can save films they want to watch in the future, separate from their collection of films they've already seen. A user can add a film to their watchlist, view their full watchlist sorted by most recently added, and is prevented from adding the same film twice.

**Endpoints added:**
- `GET /watchlist/<user_id>` — returns the user's watchlist, newest first
- `POST /watchlist/<user_id>/add` — adds a film; accepts optional `"public": false` to mark the entry private; returns 409 if already on the watchlist, 404 if the film doesn't exist
- `DELETE /watchlist/<user_id>/remove` — removes a film from the watchlist; returns 404 if not on the watchlist

**Design decisions:**

- **Default visibility: `public=True`** — CineLog is built around shared film discovery. The whole point of a watchlist in this context is to let others see what you want to watch and find common interests — without that, users would just use a private notes app. Defaulting to public means that social behavior happens naturally without requiring an extra opt-in step that most users would skip. The tradeoff is some users may self-censor, but that's the acceptable cost of optimizing for the platform's core purpose.

- **Sort order: newest first** — Watchlists are append-only (no remove feature yet), so alphabetical order becomes less useful as a list grows. Newest-first keeps the top of the list relevant — recently added films are the ones the user is most actively thinking about.

**How to test manually:**

1. Start the app: `python app.py`
2. In a separate terminal, add a film to a user's watchlist (replace UUIDs with real ones from your database):
   ```bash
   curl -X POST http://127.0.0.1:5000/watchlist/<user_id>/add \
     -H "Content-Type: application/json" \
     -d '{"film_id": "<film_uuid>"}'
   ```
   Expect: `201` with the entry JSON
3. Add the same film again — expect `409`
4. Add a nonexistent film ID — expect `404`
5. View the watchlist:
   ```bash
   curl http://127.0.0.1:5000/watchlist/<user_id>
   ```
   Expect: list of films sorted newest-first
6. Test the visibility toggle — add a film with `"public": false` and confirm the entry's `public` field is `false` in the response:
   ```bash
   curl -X POST http://127.0.0.1:5000/watchlist/<user_id>/add \
     -H "Content-Type: application/json" \
     -d '{"film_id": "<film_uuid>", "public": false}'
   ```
7. Test removing a film from the watchlist:
   ```bash
   curl -X DELETE http://127.0.0.1:5000/watchlist/<user_id>/remove \
     -H "Content-Type: application/json" \
     -d '{"film_id": "<film_uuid>"}'
   ```
   Expect: `200` with a confirmation message. Removing again should return `404`.
8. Run the test suite: `pytest tests/ -v` — all 11 tests should pass

---

## remove_from_watchlist() (Stretch)

Added `remove_from_watchlist(user_id, film_id)` to `services/watchlist_service.py`, following the same pattern as `remove_from_collection()`. It queries for an existing `WatchlistEntry` by `(user_id, film_id)`, raises `NotInWatchlistError` if not found, and deletes and commits if found. Added a `DELETE /watchlist/<user_id>/remove` endpoint in the route that catches `NotInWatchlistError` and returns 404. Two tests cover it: one confirms the entry is gone from the database after removal, one confirms that removing a film not on the watchlist raises `NotInWatchlistError`.

---

## Visibility Toggle Endpoint (Stretch)

Added an optional `public` parameter to `add_to_watchlist(user_id, film_id, public=True)` in `services/watchlist_service.py` and wired it through the `POST /watchlist/<user_id>/add` route. Callers can now set visibility explicitly:

```json
{ "film_id": "<uuid>", "public": false }
```

If `public` is omitted, it defaults to `True` — consistent with the decision documented in Comment 4. The parameter passes through `data.get("public", True)` in the route so omitting it behaves identically to the original endpoint.

---

## Second Test (Stretch)

Beyond the nonexistent `film_id` test required by Comment 3, I also wrote `test_get_watchlist_returns_newest_first` in `tests/test_watchlist.py`.

I chose this case because sort order is a behavioral contract — if the order silently changes, callers will get results in the wrong sequence with no error to signal the problem. A test that asserts position explicitly (`titles[0] == "Blade Runner"`) catches that regression in a way that the happy-path tests don't. It also directly verifies the decision made in Comment 3, so the design choice has test coverage backing it up.

---

## Additional Fix (not a review comment)

While addressing the review comments, a bug was found in `routes/watchlist/watchlist.py`: the `add_film()` endpoint imported `FilmNotFoundError` but never wrapped the `add_to_watchlist()` call in a `try/except`. This meant a duplicate add would raise `AlreadyInWatchlistError` with no handler, returning a 500 instead of a 409.

Fixed by wrapping the call in a `try/except` block mirroring the pattern in `routes/collection.py`, catching both `FilmNotFoundError` (404) and `AlreadyInWatchlistError` (409).
