## Summary

The main upload and Q&A flows do not work reliably, and question answering is not limited to the signed-in user's documents. Auth is also not enforced after login.

## Bugs

### 1. Uploaded documents are rolled back

`execute_query` in `backend/db.py` commits only when `fetch=False`. The document insert in `backend/app.py` uses `RETURNING` with `fetch=True`, so the connection closes and the insert is rolled back.

Chunk inserts then reference a `doc_id` that was never committed and fail the foreign key. The API returns "Failed to process document."

### 2. Ask query likely fails on the vector type

`/api/ask` runs:

```sql
SELECT chunk_id, content FROM chunks ORDER BY embedding <=> %s LIMIT 2
```

psycopg2 sends a Python list as a numeric array, not a `vector`. Without a `::vector` cast, Postgres typically rejects the `<=>` operator.

### 3. Retrieval is not scoped to the current user

That same query searches every row in `chunks`. Answers can be drawn from other users' notes. Upload and ask also trust a client-supplied `user_id` and never check document ownership.

## Auth gaps

- Login returns only `user_id`. There is no session or token, and later requests are not authenticated.
- `roles` is written at registration and never checked.
- Passwords are salted SHA-256, which is too fast for password storage.

## Other gaps

- `page_number` stores the chunk index, not the PDF page.
- Chunks are hard 500-character slices, so text is often cut mid-sentence.
- The frontend receives `cited_chunk_ids` but never displays them.
- CORS is open, and the server runs with `debug=True`.
- Login and ask index into the JSON body without checking that it exists.
- There is no README, tests, or schema runner.

## Suggested fix order

1. Commit `INSERT ... RETURNING` in `execute_query` (or commit before returning fetched rows).
2. Cast the query embedding to `vector` and filter chunks by the owning user.
3. Issue a real session or token and stop trusting a client-supplied `user_id`.
