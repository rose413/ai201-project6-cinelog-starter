# PR Response Doc — CineLog Watchlist Feature

## AI Usage
I used Claude on Comments 4 and 5 to stress-test my design arguments and ensure they were well-organized and covered potential counterarguments. I also used it to understand the repo — specifically to clarify what was in `models.py` and what each function did across the routes and services.

## Comment 1 — Rename
**What I did:** I renamed `save_to_watchlist()` to `add_to_watchlist()` in `services/watchlist_service.py`. I also updated both call sites in `routes/watchlist/watchlist.py`: the import on line 8 (`from services.watchlist_service import add_to_watchlist, get_watchlist`) and the function call on line 32 (`entry = add_to_watchlist(...)`).
**How I verified:** I used a project-wide search for `save_to_watchlist` to confirm no remaining references. Then, I ran the full test suite to confirm nothing broke.

## Comment 2 — Deduplication
**What I did:** I added a `AlreadyInWatchlistError` exception class to `services/watchlist_service.py`. Inside `add_to_watchlist()`, I added a `WatchlistEntry.query.filter_by(user_id=user_id, film_id=film_id).first()` check before creating a new entry — mirroring the exact pattern used in `add_to_collection()` in `services/collection_service.py`. If an existing entry is found, `AlreadyInWatchlistError` is raised instead of creating a duplicate.
**How I verified:** I ran the full test suite and confirmed all tests passed.

## Comment 3 — Missing test
**What I did:** I created `tests/test_watchlist.py` with `app` and `sample_user` fixtures copied from `tests/test_collection.py`. I wrote `test_add_to_watchlist()` to mirror `test_add_to_collection_nonexistent_film_raises`: it passes a fake UUID (`00000000-0000-0000-0000-000000000000`) to `add_to_watchlist()` and asserts that `FilmNotFoundError` is raised.
**How I verified:** I ran `pytest tests/test_watchlist.py -v` — 1 test collected, 1 passed. Then, I ran the full test suite and confirmed all tests passed.

## Comment 4 — Default visibility
**My position:** The default should be `public=False` (private).
**Reasoning:** I am optimizing a user who knows their watchlist is private will add films freely — including niche, embarrassing, or personal choices — without self-censoring for an audience. A watchlist is a personal planning tool. Defaulting to `public=True` means every entry is shared without the user ever explicitly consenting to that. The burden should be on the user to opt into sharing, not to opt out of privacy.
**Tradeoff acknowledged:** Defaulting to private has a real cost. Social discovery features — finding films through what others want to watch, following users, seeing trending saves — only work if there is public content to browse. A new user on a private-by-default app sees empty social feeds, which suppresses the network effects CineLog depends on for engagement. If social discovery is the primary goal, `public=True` would be recommended. However, I think trust and uninhibited use matter more at this stage, and users who want to share can opt in explicitly.

## Comment 5 — Sort order
**My position:** The watchlist should sort by `date_added` descending (newest first), matching the reviewer's preference and the existing behavior of `get_collection()`.
**Reasoning:** I am optimizing for the moment a user opens their watchlist and asks "what did I want to watch next?" The most recently added film is the one that was top of mind — something they just discovered or someone just recommended. Showing it first reduces the friction between saving a film and actually watching it. Alphabetical order optimizes for scanning a large static library, which is the wrong mental model for a watchlist: users don't browse it like a catalog, they pick from it while deciding what to watch tonight.
**Engagement with reviewer's point:** The maintainer's point is correct, and the current alphabetical sort works against that. There is a reasonable case for alphabetical: a watchlist can grow long and `A→Z` makes it easier to check whether a specific film is already saved. However, it is a search problem, not a sort problem — the right fix is a filter or search endpoint, not changing the default sort order. For the default view, date-added descending is the right choice, and it also makes the two endpoints (`/collection` and `/watchlist`) consistent with each other.

## Comment 6 — Rebase
**What conflicted:** The `.gitignore` file differed between the main branch and `feature/watchlist`. I also noticed that the `WatchlistEntry` class was missing from `models.py`, though this was not a merge conflict — it was deleted on main during the UUID migration.
**How I resolved it:** I resolved it by examining the conflicting lines and saw that both versions were similar, except the main branch had an extra line. I chose to keep the main branch version since it had the more complete content. I also manually restored the `WatchlistEntry` model to `models.py` with `film_id` updated to `String(36)` to match the UUID refactor.
**How I verified no conflict remains:** I verified no conflict remains by using the merge editor to confirm no conflict markers were left in any file, then ran the full test suite (`pytest tests/ -v` — 5 passed) to confirm the branch is stable.

## PR Description

### Overview
This PR adds a watchlist feature to CineLog. Users can save films they intend to watch later, view their full watchlist, and are protected from adding the same film twice. The feature introduces a `WatchlistEntry` model, a `watchlist_service` with `add_to_watchlist()` and `get_watchlist()`, and two REST endpoints registered under `/watchlist`.

### Design Decisions

**Visibility default (`public=True`):** Each `WatchlistEntry` has a `public` boolean field. The current default is `public=True`. I argued in my review response that `public=False` (private by default) is the better choice — users should opt into sharing rather than opt out of it — but I left the existing default unchanged pending maintainer sign-off, since this is a design decision that affects the social feature roadmap.

**Sort order (alphabetical by title):** `get_watchlist()` currently sorts results alphabetically by `Film.title`. I argued in my review response that `date_added` descending is the better default — it surfaces the most recently saved film first, matching user intent and making the endpoint consistent with `get_collection()`. Again, I left the code as-is pending maintainer decision.

### Manual Testing Steps

**Setup:** ensure the app is running locally (`flask run`) and you have a valid `user_id` and at least one `film_id` in the database.

1. **Add a film to the watchlist**
   ```
   POST /watchlist/<user_id>/add
   Body: { "film_id": "<film_uuid>" }
   ```
   Expected: `201` response with the new `WatchlistEntry` as JSON (`id`, `user_id`, `film_id`, `date_added`, `public`).

2. **View the watchlist**
   ```
   GET /watchlist/<user_id>
   ```
   Expected: `200` response with a JSON array of film objects, each including `date_added` and `public` fields.

3. **Verify deduplication**
   Repeat the same `POST` from step 1 with the same `film_id`.
   Expected: the service raises `AlreadyInWatchlistError` — confirm the endpoint returns a `4xx` error rather than creating a duplicate entry.

4. **Verify nonexistent film handling**
   ```
   POST /watchlist/<user_id>/add
   Body: { "film_id": "00000000-0000-0000-0000-000000000000" }
   ```
   Expected: `FilmNotFoundError` is raised — confirm the endpoint returns a `4xx` error.

5. **Run the automated test suite**
   ```
   pytest tests/ -v
   ```
   Expected: 5 passed, 0 failed.
