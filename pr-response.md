AI usage:
    I used AI usage to aid in writing out the manual testing and PR description sections.

comment 1 -- rename: Rename save_to_watchlist() to add_to_watchlist() in services/watchlist_service.py and update all call sites 
    Found the code segment from the comment in watchlist_service.py and used vscode's rename symbol feature to change all occurrences.
comment 2 -- Deduplication: Add deduplication logic to add_to_watchlist() in services/watchlist_service.py
    Found the pattern for deduplication in collection.py and reimplemented in test_watchlist.py. Wrote a test in test_watchlist.py using the pattern from test_collection.py to test if the logic works.
comment 3 -- Missing test:
    created a test_add_to_watchlist_nonexistent_film_raises test in test_watchlist.py using the pattern from test_add_to_collection_nonexistent_film_raises.
comment 4 -- Default visibility:
    Watchlists are visible by default because it seemed more fitting for a social app like CineLog. While a private default will better maintain the privacy of users, the app is intended to be social so having the default privacy level reflect that better aligns with that identity. While some users may prefer to use the functionality without it being public, that will likely be the minority and it makes more sense to require those users to change their settings than the majority of users who would like for their watchlists to be public.

comment 5 -- Sort order:
    I agree that alphabetical order does not make sense for the sort order of the watchlist. Users likely will not find value searching through a watchlist of things they may have forgotten the names of in alphabetical order. I also agree that date-added is the most appropriate method to sort the list, but I would like to propose sorting by oldest first. A watchlist is a backlog or list, if a user is using the app to curate a list of things to watch, it would only make sense to present it in a way where things on the list will realistically be completed. If films are presented with the most recent additions first, anything that was already on the list previously will be pushed down. This means that the first thing that is put on a list will never be watched until the entire watchlist has been completed. By sorting by oldest first we ensure that each addition to the list will remain in place on the list. I am open to further discussion and will implement a change once we have agreed upon an answer.

comment 6 -- Rebase:
    Removed local .gitignore and rebased on origin/main. Changed reference to film ID in watchlist to match reference in collection.

---

## PR Description

### What this feature does

This PR adds a **watchlist** to CineLog — a way for users to save films they intend to watch later, separate from their collection of films already watched.

- `POST /watchlist/<user_id>/add` — adds a film to a user's watchlist. Body: `{ "film_id": "<uuid>" }`.
- `GET /watchlist/<user_id>` — returns all films on a user's watchlist, each with `date_added` and `public` attached.

Under the hood, `add_to_watchlist()` in `services/watchlist_service.py` looks up the film, rejects unknown `film_id`s with `FilmNotFoundError`, rejects duplicate entries with `AlreadyInWatchlistError`, and otherwise creates a `WatchlistEntry` row linking the user and film.

### Design decisions

- **Naming**: renamed `save_to_watchlist()` to `add_to_watchlist()` to match the project's `verb_to_noun` convention (`add_to_collection`, `remove_from_collection`, etc.), per comment 1.
- **Deduplication**: a user can only have one watchlist entry per film. Adding a film already on the watchlist raises `AlreadyInWatchlistError` instead of silently creating a duplicate row (comment 2), mirroring how `add_to_collection` handles duplicates.
- **Film ID type**: `WatchlistEntry.film_id` references `Film.id`, which is a UUID string (post-refactor), so watchlist entries use the same ID type as collection entries (comment 6).
- **Default visibility**: `WatchlistEntry.public` defaults to `True`. CineLog is a social app, so watchlists are visible by default; users who want privacy can opt out rather than everyone having to opt in (comment 4).
- **Sort order (open item)**: `get_watchlist()` currently sorts alphabetically by film title. Per the discussion in comment 5, alphabetical order isn't useful for a backlog-style list — date-added is more appropriate, and I've proposed sorting oldest-first so earlier additions aren't perpetually pushed down by newer ones. This change is not yet implemented pending agreement on the approach.

### Manual testing

The app has no endpoint for creating users or films (film data is meant to be seeded, and there's no signup flow yet), so a user and film need to be inserted directly for manual testing.

1. Install dependencies and start the app:
   ```bash
   pip install -r requirements.txt
   python app.py
   ```
   The app runs at `http://localhost:5000` using `cinelog.db`.

2. In a second terminal, open a Python shell in the app context and create a test user and film:
   ```bash
   python
   ```
   ```python
   from app import create_app, db
   from models import User, Film

   app = create_app()
   with app.app_context():
       user = User(username="testuser", email="test@example.com")
       film = Film(title="Paddington 2", year=2017, genre="Comedy")
       db.session.add_all([user, film])
       db.session.commit()
       print("user_id:", user.id)
       print("film_id:", film.id)
   ```
   Copy the printed `user_id` and `film_id` for the requests below.

3. Add the film to the watchlist:
   ```bash
   curl -X POST http://localhost:5000/watchlist/<user_id>/add \
     -H "Content-Type: application/json" \
     -d '{"film_id": "<film_id>"}'
   ```
   Expect `201` with the new watchlist entry (`id`, `user_id`, `film_id`, `date_added`, `public: true`).

4. View the watchlist:
   ```bash
   curl http://localhost:5000/watchlist/<user_id>
   ```
   Expect a `200` with a list containing the film, including `date_added` and `public` fields.

5. Try adding the same film again to confirm deduplication:
   ```bash
   curl -X POST http://localhost:5000/watchlist/<user_id>/add \
     -H "Content-Type: application/json" \
     -d '{"film_id": "<film_id>"}'
   ```
   This currently raises `AlreadyInWatchlistError` uncaught by the route, so expect a `500` rather than a clean `409` — the route doesn't catch service exceptions the way `routes/collection.py` does. Worth fixing in a follow-up.

6. Try adding a nonexistent film to confirm the not-found check:
   ```bash
   curl -X POST http://localhost:5000/watchlist/<user_id>/add \
     -H "Content-Type: application/json" \
     -d '{"film_id": "00000000-0000-0000-0000-000000000000"}'
   ```
   Same caveat as above: expect a `500` (from the uncaught `FilmNotFoundError`) rather than a `404`.

7. Run the automated test suite:
   ```bash
   pytest tests/test_watchlist.py -v
   ```
   All tests (duplicate rejection, nonexistent film rejection) should pass.


