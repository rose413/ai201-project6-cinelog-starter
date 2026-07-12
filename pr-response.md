# PR Response Doc — CineLog Watchlist Feature

## AI Usage
<!-- Fill in at the end — how you used AI tools during this project -->

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
**My position:**
**Reasoning:**
**Tradeoff acknowledged:**

## Comment 5 — Sort order
**My position:**
**Reasoning:**
**Engagement with reviewer's point:**

## Comment 6 — Rebase
**What conflicted:**
**How I resolved it:**
**How I verified no conflict remains:**

## PR Description
<!-- Written at the end — feature overview, design decisions, manual testing steps -->
