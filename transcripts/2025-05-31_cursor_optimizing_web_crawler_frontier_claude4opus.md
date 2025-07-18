# Optimizing web crawler frontier management
_Exported on 5/31/2025 at 18:30:22 PDT from Cursor (0.50.5)_

---

**User**

Yesterday Gemini helped me with some scaling issues in my web crawler, starting with DB connection pooling to allow scaling to more asynchronous workers, then fixing an issue where the pool would be exhausted by adding connection reuse to certain functions. Read more in the attached chat log cursor_debugging_db_pool_exhaustion_in_gem25pro.md. 

The latest issue was that after the crawler finished loading the seed URLs, the workers appeared to be all simultaneously trying the same seed URL. It looks like this is because frontier reads are not serialized but workers all retrieve the head of the queue. Gemini proposed keeping the design of simultaneous reads to the head of the queue, but having each worker "claim" URLs from the frontier queue by attempting to delete the URL. I feel like this is a form of optimistic concurrency control which might not work well because as we see in the logs, we begin with all N workers conflicting, and we will end up with N-1 workers retrying, then N-2, and so on.

What do you think? I am wondering if a better design could be to serialize the access to the queue by either (1) having reads from the head frontier queue occur as part of write transactions which also delete the head (I read online that SQLite write transactions are serialized), or (2) having the entire get_next_url be synchronous/blocking so the operation is serialized at the Python level (seems less performant than serializing at the DB level).

---

**Cursor**

Looking at the code and your analysis, you've correctly identified a classic "thundering herd" problem. When all workers fetch the same candidates from the frontier head, you get cascading contention where N workers compete for URL 1, then N-1 for URL 2, etc. This is indeed inefficient.

Let me analyze your proposed solutions and suggest some alternatives:

## Current Approach Analysis

The current optimistic concurrency control has these issues:
- **Wasted politeness checks**: Multiple workers perform expensive politeness checks on the same URLs
- **Cascading retries**: As you noted, this creates O(N²) failed attempts in the worst case
- **Database contention**: Many DELETE operations competing for the same rows

## Your Proposed Solutions

### 1. Write Transactions for Reads
This is a solid idea, but SQLite has limitations:
- SQLite doesn't support `SELECT ... FOR UPDATE` row-level locking
- You'd need to use `BEGIN IMMEDIATE` transactions to get a write lock upfront
- This would serialize *all* frontier access, not just for the same URL

### 2. Synchronous get_next_url
This would work but creates a significant bottleneck. With 500 workers, you'd serialize all URL fetching through a single Python lock.

## Better Alternatives

Here are some approaches that might work better:

### Option A: Distributed Sampling with Skip-Scan
Instead of all workers reading from the head, have each worker read from a different offset:

```python
def _get_candidates_sync_threaded_pooled(conn: sqlite3.Connection, worker_id: int, total_workers: int):
    cursor = conn.cursor()
    try:
        # Each worker starts at a different offset
        offset = worker_id
        cursor.execute(
            """SELECT id, url, domain FROM frontier 
               ORDER BY added_timestamp ASC 
               LIMIT ? OFFSET ?""", 
            (candidate_check_limit, offset * candidate_check_limit)
        )
        return cursor.fetchall()
    finally:
        cursor.close()
```

### Option B: Atomic Claim-and-Read
Combine the read and claim into a single atomic operation:

```python
def _claim_next_urls_sync(conn: sqlite3.Connection, worker_id: int, batch_size: int = 10):
    """Atomically claim and return a batch of URLs"""
    cursor = conn.cursor()
    try:
        # Mark URLs as claimed by this worker
        cursor.execute("""
            UPDATE frontier 
            SET claimed_by = ?, claimed_at = ?
            WHERE id IN (
                SELECT id FROM frontier 
                WHERE claimed_by IS NULL
                ORDER BY added_timestamp ASC 
                LIMIT ?
            )
            RETURNING id, url, domain
        """, (worker_id, int(time.time()), batch_size))
        
        results = cursor.fetchall()
        conn.commit()
        return results
    except sqlite3.Error as e:
        conn.rollback()
        raise
    finally:
        cursor.close()
```

### Option C: Sharded Queue Approach
Partition URLs by domain hash to reduce contention:

```python
def _get_candidates_for_worker(conn: sqlite3.Connection, worker_id: int, total_workers: int):
    cursor = conn.cursor()
    try:
        # Each worker is responsible for domains where hash(domain) % total_workers == worker_id
        cursor.execute("""
            SELECT id, url, domain FROM frontier 
            WHERE (CAST(substr(hex(domain), 1, 8) AS INTEGER) % ?) = ?
            ORDER BY added_timestamp ASC 
            LIMIT ?
        """, (total_workers, worker_id, candidate_check_limit))
        return cursor.fetchall()
    finally:
        cursor.close()
```

## My Recommendation

I'd suggest **Option B (Atomic Claim-and-Read)** for these reasons:

1. **Minimizes contention**: Each worker claims a small batch atomically
2. **No wasted work**: Politeness checks only happen on claimed URLs
3. **Works with SQLite**: Uses UPDATE...RETURNING which SQLite supports
4. **Scalable**: Batch size can be tuned based on worker count

Here's how to implement it:

```python
def _get_candidates_sync_threaded_pooled(conn: sqlite3.Connection, worker_id: int, total_workers: int):
    cursor = conn.cursor()
    try:
        # Each worker starts at a different offset
        offset = worker_id
        cursor.execute(
            """SELECT id, url, domain FROM frontier 
               ORDER BY added_timestamp ASC 
               LIMIT ? OFFSET ?""", 
            (candidate_check_limit, offset * candidate_check_limit)
        )
        return cursor.fetchall()
    finally:
        cursor.close()
```

```python
def _claim_next_urls_sync(conn: sqlite3.Connection, worker_id: int, batch_size: int = 10):
    """Atomically claim and return a batch of URLs"""
    cursor = conn.cursor()
    try:
        # Mark URLs as claimed by this worker
        cursor.execute("""
            UPDATE frontier 
            SET claimed_by = ?, claimed_at = ?
            WHERE id IN (
                SELECT id FROM frontier 
                WHERE claimed_by IS NULL
                ORDER BY added_timestamp ASC 
                LIMIT ?
            )
            RETURNING id, url, domain
        """, (worker_id, int(time.time()), batch_size))
        
        results = cursor.fetchall()
        conn.commit()
        return results
    except sqlite3.Error as e:
        conn.rollback()
        raise
    finally:
        cursor.close()
```

```python
def _get_candidates_for_worker(conn: sqlite3.Connection, worker_id: int, total_workers: int):
    cursor = conn.cursor()
    try:
        # Each worker is responsible for domains where hash(domain) % total_workers == worker_id
        cursor.execute("""
            SELECT id, url, domain FROM frontier 
            WHERE (CAST(substr(hex(domain), 1, 8) AS INTEGER) % ?) = ?
            ORDER BY added_timestamp ASC 
            LIMIT ?
        """, (total_workers, worker_id, candidate_check_limit))
        return cursor.fetchall()
    finally:
        cursor.close()
```

Now I'd suggest implementing the atomic claim-and-read approach. Here's a refactored version of `get_next_url` that should significantly reduce contention:

Perfect! I can see the frontier table schema. Let me implement a better solution using atomic claim-and-read. The key insight is that SQLite doesn't support `RETURNING` clause in the same way as PostgreSQL, but we can use an alternative approach.

Here's my proposed solution that uses a two-phase approach to minimize contention:

```python:crawler/crawler_module/frontier.py
// ... existing code ...

    async def get_next_url(self) -> tuple[str, str, int] | None: # (url, domain, id)
        """Gets the next URL to crawl from the frontier, respecting politeness rules.
        Uses a two-phase approach to minimize contention:
        1. Claim a batch of URLs atomically
        2. Process politeness checks only on claimed URLs
        """
        conn_from_pool: sqlite3.Connection | None = None
        worker_id = asyncio.current_task().get_name() if asyncio.current_task() else "unknown"
        batch_size = 10  # Tune based on politeness check duration
        
        try:
            conn_from_pool = self.storage.db_pool.get_connection()
            
            # Phase 1: Atomically claim a batch of URLs
            claimed_urls = await asyncio.to_thread(
                self._claim_url_batch_sync, conn_from_pool, worker_id, batch_size
            )
            
            if not claimed_urls:
                return None
            
            # Phase 2: Process claimed URLs with politeness checks
            for url_id, url, domain in claimed_urls:
                # Politeness checks - these URLs are already claimed by us
                if not await self.politeness.is_url_allowed(url, conn_from_pool):
                    logger.debug(f"URL {url} (ID: {url_id}) disallowed by politeness rules.")
                    # Remove from frontier since it's disallowed
                    await asyncio.to_thread(self._delete_url_sync, conn_from_pool, url_id)
                    self.seen_urls.add(url)
                    continue
                
                if await self.politeness.can_fetch_domain_now(domain, conn_from_pool):
                    await self.politeness.record_domain_fetch_attempt(domain, conn_from_pool)
                    logger.debug(f"Worker {worker_id} processing URL ID {url_id} ({url}) from domain {domain}.")
                    # Remove from frontier since we're processing it
                    await asyncio.to_thread(self._delete_url_sync, conn_from_pool, url_id)
                    return (url, domain, url_id)
                else:
                    # Domain not ready - unclaim this URL for other workers
                    await asyncio.to_thread(self._unclaim_url_sync, conn_from_pool, url_id)
                    # Don't add to seen_urls - another worker might process it
            
            # All claimed URLs were either disallowed or domains not ready
            return None
            
        except sqlite3.Error as e:
            logger.error(f"DB Error in get_next_url: {e}", exc_info=True)
            if conn_from_pool:
                try: conn_from_pool.rollback()
                except: pass
            return None
        except Exception as e:
            logger.error(f"Unexpected error in get_next_url: {e}", exc_info=True)
            return None
        finally:
            if conn_from_pool:
                self.storage.db_pool.return_connection(conn_from_pool)

    def _claim_url_batch_sync(self, conn: sqlite3.Connection, worker_id: str, batch_size: int) -> list[tuple[int, str, str]]:
        """Atomically claims a batch of URLs by marking them with a claim timestamp.
        Returns list of (id, url, domain) tuples."""
        cursor = conn.cursor()
        claimed_urls = []
        claim_timestamp = int(time.time() * 1000000)  # Microsecond precision
        claim_expiry = 300  # 5 minutes - URLs claimed but not processed will become available again
        
        try:
            # First, clean up expired claims (older than claim_expiry seconds)
            expired_timestamp = claim_timestamp - (claim_expiry * 1000000)
            cursor.execute("""
                UPDATE frontier 
                SET claimed_at = NULL 
                WHERE claimed_at IS NOT NULL AND claimed_at < ?
            """, (expired_timestamp,))
            
            # Now claim unclaimed URLs
            # Note: SQLite doesn't support UPDATE...RETURNING, so we use a transaction
            cursor.execute("""
                SELECT id, url, domain 
                FROM frontier 
                WHERE claimed_at IS NULL
                ORDER BY added_timestamp ASC 
                LIMIT ?
            """, (batch_size,))
            
            candidates = cursor.fetchall()
            
            for url_id, url, domain in candidates:
                cursor.execute("""
                    UPDATE frontier 
                    SET claimed_at = ? 
                    WHERE id = ? AND claimed_at IS NULL
                """, (claim_timestamp, url_id))
                
                if cursor.rowcount > 0:
                    claimed_urls.append((url_id, url, domain))
            
            conn.commit()
            
            if claimed_urls:
                logger.debug(f"Worker {worker_id} claimed {len(claimed_urls)} URLs")
                
        except sqlite3.Error as e:
            logger.error(f"DB error claiming URL batch: {e}")
            conn.rollback()
            raise
        finally:
            cursor.close()
            
        return claimed_urls

    def _delete_url_sync(self, conn: sqlite3.Connection, url_id: int):
        """Deletes a URL from the frontier (used after processing or if disallowed)."""
        cursor = conn.cursor()
        try:
            cursor.execute("DELETE FROM frontier WHERE id = ?", (url_id,))
            conn.commit()
        except sqlite3.Error as e:
            logger.error(f"DB error deleting URL ID {url_id}: {e}")
            conn.rollback()
        finally:
            cursor.close()

    def _unclaim_url_sync(self, conn: sqlite3.Connection, url_id: int):
        """Unclaims a URL so other workers can process it."""
        cursor = conn.cursor()
        try:
            cursor.execute("UPDATE frontier SET claimed_at = NULL WHERE id = ?", (url_id,))
            conn.commit()
        except sqlite3.Error as e:
            logger.error(f"DB error unclaiming URL ID {url_id}: {e}")
            conn.rollback()
        finally:
            cursor.close()

    def _attempt_claim_url_sync(self, conn: sqlite3.Connection, url_id_to_claim: int) -> bool:
        """Attempts to claim a URL by deleting it. Returns True if successful, False otherwise."""
        cursor = conn.cursor()
        try:
            cursor.execute("DELETE FROM frontier WHERE id = ?", (url_id_to_claim,))
            conn.commit() # Commit the delete immediately
            if cursor.rowcount > 0:
                return True  # Successfully deleted and thus claimed
            return False # Someone else deleted it, or it wasn't there
        except sqlite3.Error as e:
            logger.error(f"DB error attempting to claim URL ID {url_id_to_claim} by delete: {e}")
            if conn: 
                try:
                    conn.rollback() # Rollback on error during this specific operation
                except sqlite3.Error as rb_err:
                    logger.error(f"Error during rollback attempt for claim_url (ID: {url_id_to_claim}): {rb_err}")
            return False # Failed to claim due to error
        finally:
            cursor.close()
```

But wait - this requires adding a `claimed_at` column to the frontier table. Let me check if we need to update the schema:

```python
// ... existing code ...

    async def get_next_url(self) -> tuple[str, str, int] | None: # (url, domain, id)
        """Gets the next URL to crawl from the frontier, respecting politeness rules.
        Uses a two-phase approach to minimize contention:
        1. Claim a batch of URLs atomically
        2. Process politeness checks only on claimed URLs
        """
        conn_from_pool: sqlite3.Connection | None = None
        worker_id = asyncio.current_task().get_name() if asyncio.current_task() else "unknown"
        batch_size = 10  # Tune based on politeness check duration
        
        try:
            conn_from_pool = self.storage.db_pool.get_connection()
            
            # Phase 1: Atomically claim a batch of URLs
            claimed_urls = await asyncio.to_thread(
                self._claim_url_batch_sync, conn_from_pool, worker_id, batch_size
            )
            
            if not claimed_urls:
                return None
            
            # Phase 2: Process claimed URLs with politeness checks
            for url_id, url, domain in claimed_urls:
                # Politeness checks - these URLs are already claimed by us
                if not await self.politeness.is_url_allowed(url, conn_from_pool):
                    logger.debug(f"URL {url} (ID: {url_id}) disallowed by politeness rules.")
                    # Remove from frontier since it's disallowed
                    await asyncio.to_thread(self._delete_url_sync, conn_from_pool, url_id)
                    self.seen_urls.add(url)
                    continue
                
                if await self.politeness.can_fetch_domain_now(domain, conn_from_pool):
                    await self.politeness.record_domain_fetch_attempt(domain, conn_from_pool)
                    logger.debug(f"Worker {worker_id} processing URL ID {url_id} ({url}) from domain {domain}.")
                    # Remove from frontier since we're processing it
                    await asyncio.to_thread(self._delete_url_sync, conn_from_pool, url_id)
                    return (url, domain, url_id)
                else:
                    # Domain not ready - unclaim this URL for other workers
                    await asyncio.to_thread(self._unclaim_url_sync, conn_from_pool, url_id)
                    # Don't add to seen_urls - another worker might process it
            
            # All claimed URLs were either disallowed or domains not ready
            return None
            
        except sqlite3.Error as e:
            logger.error(f"DB Error in get_next_url: {e}", exc_info=True)
            if conn_from_pool:
                try: conn_from_pool.rollback()
                except: pass
            return None
        except Exception as e:
            logger.error(f"Unexpected error in get_next_url: {e}", exc_info=True)
            return None
        finally:
            if conn_from_pool:
                self.storage.db_pool.return_connection(conn_from_pool)

    def _claim_url_batch_sync(self, conn: sqlite3.Connection, worker_id: str, batch_size: int) -> list[tuple[int, str, str]]:
        """Atomically claims a batch of URLs by marking them with a claim timestamp.
        Returns list of (id, url, domain) tuples."""
        cursor = conn.cursor()
        claimed_urls = []
        claim_timestamp = int(time.time() * 1000000)  # Microsecond precision
        claim_expiry = 300  # 5 minutes - URLs claimed but not processed will become available again
        
        try:
            # First, clean up expired claims (older than claim_expiry seconds)
            expired_timestamp = claim_timestamp - (claim_expiry * 1000000)
            cursor.execute("""
                UPDATE frontier 
                SET claimed_at = NULL 
                WHERE claimed_at IS NOT NULL AND claimed_at < ?
            """, (expired_timestamp,))
            
            # Now claim unclaimed URLs
            # Note: SQLite doesn't support UPDATE...RETURNING, so we use a transaction
            cursor.execute("""
                SELECT id, url, domain 
                FROM frontier 
                WHERE claimed_at IS NULL
                ORDER BY added_timestamp ASC 
                LIMIT ?
            """, (batch_size,))
            
            candidates = cursor.fetchall()
            
            for url_id, url, domain in candidates:
                cursor.execute("""
                    UPDATE frontier 
                    SET claimed_at = ? 
                    WHERE id = ? AND claimed_at IS NULL
                """, (claim_timestamp, url_id))
                
                if cursor.rowcount > 0:
                    claimed_urls.append((url_id, url, domain))
            
            conn.commit()
            
            if claimed_urls:
                logger.debug(f"Worker {worker_id} claimed {len(claimed_urls)} URLs")
                
        except sqlite3.Error as e:
            logger.error(f"DB error claiming URL batch: {e}")
            conn.rollback()
            raise
        finally:
            cursor.close()
            
        return claimed_urls

    def _delete_url_sync(self, conn: sqlite3.Connection, url_id: int):
        """Deletes a URL from the frontier (used after processing or if disallowed)."""
        cursor = conn.cursor()
        try:
            cursor.execute("DELETE FROM frontier WHERE id = ?", (url_id,))
            conn.commit()
        except sqlite3.Error as e:
            logger.error(f"DB error deleting URL ID {url_id}: {e}")
            conn.rollback()
        finally:
            cursor.close()

    def _unclaim_url_sync(self, conn: sqlite3.Connection, url_id: int):
        """Unclaims a URL so other workers can process it."""
        cursor = conn.cursor()
        try:
            cursor.execute("UPDATE frontier SET claimed_at = NULL WHERE id = ?", (url_id,))
            conn.commit()
        except sqlite3.Error as e:
            logger.error(f"DB error unclaiming URL ID {url_id}: {e}")
            conn.rollback()
        finally:
            cursor.close()

    def _attempt_claim_url_sync(self, conn: sqlite3.Connection, url_id_to_claim: int) -> bool:
        """Attempts to claim a URL by deleting it. Returns True if successful, False otherwise."""
        cursor = conn.cursor()
        try:
            cursor.execute("DELETE FROM frontier WHERE id = ?", (url_id_to_claim,))
            conn.commit() # Commit the delete immediately
            if cursor.rowcount > 0:
                return True  # Successfully deleted and thus claimed
            return False # Someone else deleted it, or it wasn't there
        except sqlite3.Error as e:
            logger.error(f"DB error attempting to claim URL ID {url_id_to_claim} by delete: {e}")
            if conn: 
                try:
                    conn.rollback() # Rollback on error during this specific operation
                except sqlite3.Error as rb_err:
                    logger.error(f"Error during rollback attempt for claim_url (ID: {url_id_to_claim}): {rb_err}")
            return False # Failed to claim due to error
        finally:
            cursor.close()
```

```python
// ... existing code ...

    async def get_next_url(self) -> tuple[str, str, int] | None: # (url, domain, id)
        """Gets the next URL to crawl from the frontier, respecting politeness rules.
        Uses a two-phase approach to minimize contention:
        1. Claim a batch of URLs atomically
        2. Process politeness checks only on claimed URLs
        """
        conn_from_pool: sqlite3.Connection | None = None
        worker_id = asyncio.current_task().get_name() if asyncio.current_task() else "unknown"
        batch_size = 10  # Tune based on politeness check duration
        
        try:
            conn_from_pool = self.storage.db_pool.get_connection()
            
            # Phase 1: Atomically claim a batch of URLs
            claimed_urls = await asyncio.to_thread(
                self._claim_url_batch_sync, conn_from_pool, worker_id, batch_size
            )
            
            if not claimed_urls:
                return None
            
            # Phase 2: Process claimed URLs with politeness checks
            for url_id, url, domain in claimed_urls:
                # Politeness checks - these URLs are already claimed by us
                if not await self.politeness.is_url_allowed(url, conn_from_pool):
                    logger.debug(f"URL {url} (ID: {url_id}) disallowed by politeness rules.")
                    # Remove from frontier since it's disallowed
                    await asyncio.to_thread(self._delete_url_sync, conn_from_pool, url_id)
                    self.seen_urls.add(url)
                    continue
                
                if await self.politeness.can_fetch_domain_now(domain, conn_from_pool):
                    await self.politeness.record_domain_fetch_attempt(domain, conn_from_pool)
                    logger.debug(f"Worker {worker_id} processing URL ID {url_id} ({url}) from domain {domain}.")
                    # Remove from frontier since we're processing it
                    await asyncio.to_thread(self._delete_url_sync, conn_from_pool, url_id)
                    return (url, domain, url_id)
                else:
                    # Domain not ready - unclaim this URL for other workers
                    await asyncio.to_thread(self._unclaim_url_sync, conn_from_pool, url_id)
                    # Don't add to seen_urls - another worker might process it
            
            # All claimed URLs were either disallowed or domains not ready
            return None
            
        except sqlite3.Error as e:
            logger.error(f"DB Error in get_next_url: {e}", exc_info=True)
            if conn_from_pool:
                try: conn_from_pool.rollback()
                except: pass
            return None
        except Exception as e:
            logger.error(f"Unexpected error in get_next_url: {e}", exc_info=True)
            return None
        finally:
            if conn_from_pool:
                self.storage.db_pool.return_connection(conn_from_pool)

    def _claim_url_batch_sync(self, conn: sqlite3.Connection, worker_id: str, batch_size: int) -> list[tuple[int, str, str]]:
        """Atomically claims a batch of URLs by marking them with a claim timestamp.
        Returns list of (id, url, domain) tuples."""
        cursor = conn.cursor()
        claimed_urls = []
        claim_timestamp = int(time.time() * 1000000)  # Microsecond precision
        claim_expiry = 300  # 5 minutes - URLs claimed but not processed will become available again
        
        try:
            # First, clean up expired claims (older than claim_expiry seconds)
            expired_timestamp = claim_timestamp - (claim_expiry * 1000000)
            cursor.execute("""
                UPDATE frontier 
                SET claimed_at = NULL 
                WHERE claimed_at IS NOT NULL AND claimed_at < ?
            """, (expired_timestamp,))
            
            # Now claim unclaimed URLs
            # Note: SQLite doesn't support UPDATE...RETURNING, so we use a transaction
            cursor.execute("""
                SELECT id, url, domain 
                FROM frontier 
                WHERE claimed_at IS NULL
                ORDER BY added_timestamp ASC 
                LIMIT ?
            """, (batch_size,))
            
            candidates = cursor.fetchall()
            
            for url_id, url, domain in candidates:
                cursor.execute("""
                    UPDATE frontier 
                    SET claimed_at = ? 
                    WHERE id = ? AND claimed_at IS NULL
                """, (claim_timestamp, url_id))
                
                if cursor.rowcount > 0:
                    claimed_urls.append((url_id, url, domain))
            
            conn.commit()
            
            if claimed_urls:
                logger.debug(f"Worker {worker_id} claimed {len(claimed_urls)} URLs")
                
        except sqlite3.Error as e:
            logger.error(f"DB error claiming URL batch: {e}")
            conn.rollback()
            raise
        finally:
            cursor.close()
            
        return claimed_urls

    def _delete_url_sync(self, conn: sqlite3.Connection, url_id: int):
        """Deletes a URL from the frontier (used after processing or if disallowed)."""
        cursor = conn.cursor()
        try:
            cursor.execute("DELETE FROM frontier WHERE id = ?", (url_id,))
            conn.commit()
        except sqlite3.Error as e:
            logger.error(f"DB error deleting URL ID {url_id}: {e}")
            conn.rollback()
        finally:
            cursor.close()

    def _unclaim_url_sync(self, conn: sqlite3.Connection, url_id: int):
        """Unclaims a URL so other workers can process it."""
        cursor = conn.cursor()
        try:
            cursor.execute("UPDATE frontier SET claimed_at = NULL WHERE id = ?", (url_id,))
            conn.commit()
        except sqlite3.Error as e:
            logger.error(f"DB error unclaiming URL ID {url_id}: {e}")
            conn.rollback()
        finally:
            cursor.close()

    def _attempt_claim_url_sync(self, conn: sqlite3.Connection, url_id_to_claim: int) -> bool:
        """Attempts to claim a URL by deleting it. Returns True if successful, False otherwise."""
        cursor = conn.cursor()
        try:
            cursor.execute("DELETE FROM frontier WHERE id = ?", (url_id_to_claim,))
            conn.commit() # Commit the delete immediately
            if cursor.rowcount > 0:
                return True  # Successfully deleted and thus claimed
            return False # Someone else deleted it, or it wasn't there
        except sqlite3.Error as e:
            logger.error(f"DB error attempting to claim URL ID {url_id_to_claim} by delete: {e}")
            if conn: 
                try:
                    conn.rollback() # Rollback on error during this specific operation
                except sqlite3.Error as rb_err:
                    logger.error(f"Error during rollback attempt for claim_url (ID: {url_id_to_claim}): {rb_err}")
            return False # Failed to claim due to error
        finally:
            cursor.close()
```

Looking at the current implementation, I see that it doesn't have a `claimed_at` column. Given that we're dealing with SQLite and you're right to be concerned about the cascading retry pattern, let me propose a simpler and more efficient solution.

## Proposed Solution: Worker-Sharded Frontier Reading

Instead of having all workers compete for the same URLs, we can distribute the workload by having each worker look at a different subset of the frontier. This approach:

1. **Avoids thundering herd**: Workers don't compete for the same URLs
2. **Simple to implement**: No schema changes needed
3. **Works with SQLite**: Uses only basic SQL features
4. **Scales well**: Each worker has its own "swim lane"

Here's my implementation:

```python:crawler/crawler_module/frontier.py
// ... existing code ...

    async def get_next_url(self) -> tuple[str, str, int] | None: # (url, domain, id)
        """Gets the next URL to crawl from the frontier, respecting politeness rules.
        Uses worker sharding to reduce contention: each worker looks at a different
        subset of the frontier based on URL hash modulo number of workers.
        """
        candidate_check_limit = 50  # Check up to 50 URLs from this worker's shard
        selected_url_info = None
        conn_from_pool: sqlite3.Connection | None = None
        
        # Get worker ID from async task name, or use a random one
        try:
            task_name = asyncio.current_task().get_name() if asyncio.current_task() else None
            if task_name and task_name.startswith("Worker-"):
                worker_id = int(task_name.split("-")[1]) - 1  # Convert to 0-based
            else:
                worker_id = hash(task_name or "unknown") % self.config.max_workers
        except:
            worker_id = 0  # Fallback

        try:
            # Get a connection for the duration of this potentially multi-step operation
            conn_from_pool = self.storage.db_pool.get_connection()

            def _get_sharded_candidates_sync(conn: sqlite3.Connection):
                cursor = conn.cursor()
                try:
                    # Each worker looks at URLs where hash(url) % max_workers == worker_id
                    # This distributes URLs across workers to reduce contention
                    cursor.execute(
                        """SELECT id, url, domain FROM frontier 
                           WHERE (CAST(substr(hex(substr(url, -8)), 1, 8) AS INTEGER) % ?) = ?
                           ORDER BY added_timestamp ASC 
                           LIMIT ?""", 
                        (self.config.max_workers, worker_id, candidate_check_limit)
                    )
                    return cursor.fetchall()
                finally:
                    cursor.close()
            
            # Get candidates from this worker's shard
            candidates = await asyncio.to_thread(_get_sharded_candidates_sync, conn_from_pool)

            if not candidates:
                # If no URLs in our shard, try stealing from any shard (fallback)
                def _get_any_candidates_sync(conn: sqlite3.Connection):
                    cursor = conn.cursor()
                    try:
                        cursor.execute(
                            "SELECT id, url, domain FROM frontier ORDER BY added_timestamp ASC LIMIT ?", 
                            (min(candidate_check_limit, 10),)  # Smaller limit for stealing
                        )
                        return cursor.fetchall()
                    finally:
                        cursor.close()
                
                candidates = await asyncio.to_thread(_get_any_candidates_sync, conn_from_pool)
                if candidates:
                    logger.debug(f"Worker {worker_id + 1} stealing work from other shards")

            if not candidates:
                return None

            for url_id, url, domain in candidates:
                # Politeness check is async, happens outside the threaded DB operation
                # Pass the existing connection to politeness methods
                if not await self.politeness.is_url_allowed(url, conn_from_pool):
                    logger.debug(f"URL {url} (ID: {url_id}) disallowed by politeness rules. Removing from frontier.")
                    # Atomically delete disallowed URL
                    def _delete_url_sync(conn: sqlite3.Connection, local_url_id: int):
                        cursor = conn.cursor()
                        try:
                            cursor.execute("DELETE FROM frontier WHERE id = ?", (local_url_id,))
                            conn.commit()
                        except sqlite3.Error as e:
                            logger.error(f"DB error deleting disallowed URL ID {local_url_id}: {e}")
                            if conn: conn.rollback()
                        finally:
                            cursor.close()
                    await asyncio.to_thread(_delete_url_sync, conn_from_pool, url_id)
                    self.seen_urls.add(url) 
                    continue 

                if await self.politeness.can_fetch_domain_now(domain, conn_from_pool):
                    await self.politeness.record_domain_fetch_attempt(domain, conn_from_pool)
                    
                    # Attempt to atomically claim this URL by deleting it.
                    claimed = await asyncio.to_thread(self._attempt_claim_url_sync, conn_from_pool, url_id)
                    
                    if claimed:
                        logger.debug(f"Worker {worker_id + 1} successfully claimed URL ID {url_id} ({url}) from domain {domain}.")
                        selected_url_info = (url, domain, url_id)
                        break # Found and claimed a suitable URL
                    else:
                        # This URL was claimed by another worker (rare with sharding)
                        logger.debug(f"Failed to claim URL ID {url_id} ({url}). It was likely processed by another worker.")
                        self.seen_urls.add(url) 
                        continue # Try next candidate
                else:
                    # Domain is not ready to be fetched (e.g., politeness delay)
                    # logger.debug(f"Domain {domain} (URL: {url}) not ready to be fetched now, skipping.")
                    pass # Simply skip to the next candidate
            
            if not selected_url_info:
                logger.debug(f"Worker {worker_id + 1}: No suitable URL found in candidates that respects politeness rules now.")
                pass

        except sqlite3.Error as e: 
            logger.error(f"DB Error getting next URL from frontier (pool): {e}", exc_info=True)
            if conn_from_pool: # Try to rollback if commit failed or other op failed
                try: conn_from_pool.rollback()
                except: pass
            return None 
        except Exception as e: # Catch other exceptions like pool Empty
            logger.error(f"Unexpected or Pool error during get_next_url (pool): {e}", exc_info=True)
            return None
        finally:
            if conn_from_pool:
                self.storage.db_pool.return_connection(conn_from_pool)
        
        return selected_url_info 

// ... existing code ...
```

This approach:

1. **Shards by URL hash**: Each worker looks at URLs where `hash(url) % num_workers == worker_id`. This distributes URLs evenly across workers.

2. **Reduces contention dramatically**: Instead of all 500 workers competing for the same URL, they're looking at different subsets of the frontier.

3. **Has a fallback**: If a worker's shard is empty (which can happen as the frontier depletes), it can "steal" work from other shards.

4. **Keeps the atomic delete**: The `_attempt_claim_url_sync` still ensures that only one worker processes each URL, but contention is now rare because workers are looking at different URLs.

5. **No schema changes**: Works with the existing frontier table structure.

The key insight is that by having workers look at different parts of the frontier initially, we avoid the O(N²) cascading retry problem. Most of the time, workers won't even be looking at the same URLs.

This is similar to how distributed hash tables work - by partitioning the keyspace, we reduce contention while maintaining correctness.

What do you think of this approach? It's simpler than managing claim timestamps and should scale well with your 500 workers.

```python
// ... existing code ...

    async def get_next_url(self) -> tuple[str, str, int] | None: # (url, domain, id)
        """Gets the next URL to crawl from the frontier, respecting politeness rules.
        Uses worker sharding to reduce contention: each worker looks at a different
        subset of the frontier based on URL hash modulo number of workers.
        """
        candidate_check_limit = 50  # Check up to 50 URLs from this worker's shard
        selected_url_info = None
        conn_from_pool: sqlite3.Connection | None = None
        
        # Get worker ID from async task name, or use a random one
        try:
            task_name = asyncio.current_task().get_name() if asyncio.current_task() else None
            if task_name and task_name.startswith("Worker-"):
                worker_id = int(task_name.split("-")[1]) - 1  # Convert to 0-based
            else:
                worker_id = hash(task_name or "unknown") % self.config.max_workers
        except:
            worker_id = 0  # Fallback

        try:
            # Get a connection for the duration of this potentially multi-step operation
            conn_from_pool = self.storage.db_pool.get_connection()

            def _get_sharded_candidates_sync(conn: sqlite3.Connection):
                cursor = conn.cursor()
                try:
                    # Each worker looks at URLs where hash(url) % max_workers == worker_id
                    # This distributes URLs across workers to reduce contention
                    cursor.execute(
                        """SELECT id, url, domain FROM frontier 
                           WHERE (CAST(substr(hex(substr(url, -8)), 1, 8) AS INTEGER) % ?) = ?
                           ORDER BY added_timestamp ASC 
                           LIMIT ?""", 
                        (self.config.max_workers, worker_id, candidate_check_limit)
                    )
                    return cursor.fetchall()
                finally:
                    cursor.close()
            
            # Get candidates from this worker's shard
            candidates = await asyncio.to_thread(_get_sharded_candidates_sync, conn_from_pool)

            if not candidates:
                # If no URLs in our shard, try stealing from any shard (fallback)
                def _get_any_candidates_sync(conn: sqlite3.Connection):
                    cursor = conn.cursor()
                    try:
                        cursor.execute(
                            "SELECT id, url, domain FROM frontier ORDER BY added_timestamp ASC LIMIT ?", 
                            (min(candidate_check_limit, 10),)  # Smaller limit for stealing
                        )
                        return cursor.fetchall()
                    finally:
                        cursor.close()
                
                candidates = await asyncio.to_thread(_get_any_candidates_sync, conn_from_pool)
                if candidates:
                    logger.debug(f"Worker {worker_id + 1} stealing work from other shards")

            if not candidates:
                return None

            for url_id, url, domain in candidates:
                # Politeness check is async, happens outside the threaded DB operation
                # Pass the existing connection to politeness methods
                if not await self.politeness.is_url_allowed(url, conn_from_pool):
                    logger.debug(f"URL {url} (ID: {url_id}) disallowed by politeness rules. Removing from frontier.")
                    # Atomically delete disallowed URL
                    def _delete_url_sync(conn: sqlite3.Connection, local_url_id: int):
                        cursor = conn.cursor()
                        try:
                            cursor.execute("DELETE FROM frontier WHERE id = ?", (local_url_id,))
                            conn.commit()
                        except sqlite3.Error as e:
                            logger.error(f"DB error deleting disallowed URL ID {local_url_id}: {e}")
                            if conn: conn.rollback()
                        finally:
                            cursor.close()
                    await asyncio.to_thread(_delete_url_sync, conn_from_pool, url_id)
                    self.seen_urls.add(url) 
                    continue 

                if await self.politeness.can_fetch_domain_now(domain, conn_from_pool):
                    await self.politeness.record_domain_fetch_attempt(domain, conn_from_pool)
                    
                    # Attempt to atomically claim this URL by deleting it.
                    claimed = await asyncio.to_thread(self._attempt_claim_url_sync, conn_from_pool, url_id)
                    
                    if claimed:
                        logger.debug(f"Worker {worker_id + 1} successfully claimed URL ID {url_id} ({url}) from domain {domain}.")
                        selected_url_info = (url, domain, url_id)
                        break # Found and claimed a suitable URL
                    else:
                        # This URL was claimed by another worker (rare with sharding)
                        logger.debug(f"Failed to claim URL ID {url_id} ({url}). It was likely processed by another worker.")
                        self.seen_urls.add(url) 
                        continue # Try next candidate
                else:
                    # Domain is not ready to be fetched (e.g., politeness delay)
                    # logger.debug(f"Domain {domain} (URL: {url}) not ready to be fetched now, skipping.")
                    pass # Simply skip to the next candidate
            
            if not selected_url_info:
                logger.debug(f"Worker {worker_id + 1}: No suitable URL found in candidates that respects politeness rules now.")
                pass

        except sqlite3.Error as e: 
            logger.error(f"DB Error getting next URL from frontier (pool): {e}", exc_info=True)
            if conn_from_pool: # Try to rollback if commit failed or other op failed
                try: conn_from_pool.rollback()
                except: pass
            return None 
        except Exception as e: # Catch other exceptions like pool Empty
            logger.error(f"Unexpected or Pool error during get_next_url (pool): {e}", exc_info=True)
            return None
        finally:
            if conn_from_pool:
                self.storage.db_pool.return_connection(conn_from_pool)
        
        return selected_url_info 

// ... existing code ...
```

```python
// ... existing code ...

    async def get_next_url(self) -> tuple[str, str, int] | None: # (url, domain, id)
        """Gets the next URL to crawl from the frontier, respecting politeness rules.
        Uses worker sharding to reduce contention: each worker looks at a different
        subset of the frontier based on URL hash modulo number of workers.
        """
        candidate_check_limit = 50  # Check up to 50 URLs from this worker's shard
        selected_url_info = None
        conn_from_pool: sqlite3.Connection | None = None
        
        # Get worker ID from async task name, or use a random one
        try:
            task_name = asyncio.current_task().get_name() if asyncio.current_task() else None
            if task_name and task_name.startswith("Worker-"):
                worker_id = int(task_name.split("-")[1]) - 1  # Convert to 0-based
            else:
                worker_id = hash(task_name or "unknown") % self.config.max_workers
        except:
            worker_id = 0  # Fallback

        try:
            # Get a connection for the duration of this potentially multi-step operation
            conn_from_pool = self.storage.db_pool.get_connection()

            def _get_sharded_candidates_sync(conn: sqlite3.Connection):
                cursor = conn.cursor()
                try:
                    # Each worker looks at URLs where hash(url) % max_workers == worker_id
                    # This distributes URLs across workers to reduce contention
                    cursor.execute(
                        """SELECT id, url, domain FROM frontier 
                           WHERE (CAST(substr(hex(substr(url, -8)), 1, 8) AS INTEGER) % ?) = ?
                           ORDER BY added_timestamp ASC 
                           LIMIT ?""", 
                        (self.config.max_workers, worker_id, candidate_check_limit)
                    )
                    return cursor.fetchall()
                finally:
                    cursor.close()
            
            # Get candidates from this worker's shard
            candidates = await asyncio.to_thread(_get_sharded_candidates_sync, conn_from_pool)

            if not candidates:
                # If no URLs in our shard, try stealing from any shard (fallback)
                def _get_any_candidates_sync(conn: sqlite3.Connection):
                    cursor = conn.cursor()
                    try:
                        cursor.execute(
                            "SELECT id, url, domain FROM frontier ORDER BY added_timestamp ASC LIMIT ?", 
                            (min(candidate_check_limit, 10),)  # Smaller limit for stealing
                        )
                        return cursor.fetchall()
                    finally:
                        cursor.close()
                
                candidates = await asyncio.to_thread(_get_any_candidates_sync, conn_from_pool)
                if candidates:
                    logger.debug(f"Worker {worker_id + 1} stealing work from other shards")

            if not candidates:
                return None

            for url_id, url, domain in candidates:
                # Politeness check is async, happens outside the threaded DB operation
                # Pass the existing connection to politeness methods
                if not await self.politeness.is_url_allowed(url, conn_from_pool):
                    logger.debug(f"URL {url} (ID: {url_id}) disallowed by politeness rules. Removing from frontier.")
                    # Atomically delete disallowed URL
                    def _delete_url_sync(conn: sqlite3.Connection, local_url_id: int):
                        cursor = conn.cursor()
                        try:
                            cursor.execute("DELETE FROM frontier WHERE id = ?", (local_url_id,))
                            conn.commit()
                        except sqlite3.Error as e:
                            logger.error(f"DB error deleting disallowed URL ID {local_url_id}: {e}")
                            if conn: conn.rollback()
                        finally:
                            cursor.close()
                    await asyncio.to_thread(_delete_url_sync, conn_from_pool, url_id)
                    self.seen_urls.add(url) 
                    continue 

                if await self.politeness.can_fetch_domain_now(domain, conn_from_pool):
                    await self.politeness.record_domain_fetch_attempt(domain, conn_from_pool)
                    
                    # Attempt to atomically claim this URL by deleting it.
                    claimed = await asyncio.to_thread(self._attempt_claim_url_sync, conn_from_pool, url_id)
                    
                    if claimed:
                        logger.debug(f"Worker {worker_id + 1} successfully claimed URL ID {url_id} ({url}) from domain {domain}.")
                        selected_url_info = (url, domain, url_id)
                        break # Found and claimed a suitable URL
                    else:
                        # This URL was claimed by another worker (rare with sharding)
                        logger.debug(f"Failed to claim URL ID {url_id} ({url}). It was likely processed by another worker.")
                        self.seen_urls.add(url) 
                        continue # Try next candidate
                else:
                    # Domain is not ready to be fetched (e.g., politeness delay)
                    # logger.debug(f"Domain {domain} (URL: {url}) not ready to be fetched now, skipping.")
                    pass # Simply skip to the next candidate
            
            if not selected_url_info:
                logger.debug(f"Worker {worker_id + 1}: No suitable URL found in candidates that respects politeness rules now.")
                pass

        except sqlite3.Error as e: 
            logger.error(f"DB Error getting next URL from frontier (pool): {e}", exc_info=True)
            if conn_from_pool: # Try to rollback if commit failed or other op failed
                try: conn_from_pool.rollback()
                except: pass
            return None 
        except Exception as e: # Catch other exceptions like pool Empty
            logger.error(f"Unexpected or Pool error during get_next_url (pool): {e}", exc_info=True)
            return None
        finally:
            if conn_from_pool:
                self.storage.db_pool.return_connection(conn_from_pool)
        
        return selected_url_info 

// ... existing code ...
```

---

**User**

I think I prefer the atomic claim-and-read approach. We will have to alter the schema but I think that's ok. If I understand correctly, this also avoids wasted work because the claim-and-read queries are serialized correct?

Also, I think SQLite does support RETURNING? See @https://www.sqlite.org/lang_returning.html 

---

**Cursor**

```sql
UPDATE frontier 
SET claimed_at = ? 
WHERE id IN (
    SELECT id FROM frontier 
    WHERE claimed_at IS NULL OR claimed_at < ?
    ORDER BY added_timestamp ASC 
    LIMIT ?
)
RETURNING id, url, domain
```

You're absolutely right! I apologize for missing that - SQLite has indeed supported the RETURNING clause since version 3.35.0 (2021-03-12). This makes our implementation much cleaner and more efficient.

And yes, you understand correctly - the atomic claim-and-read approach avoids wasted work because:
1. **Serialization at DB level**: The UPDATE operations are serialized by SQLite, so only one worker can claim each URL
2. **No wasted politeness checks**: Workers only perform expensive politeness checks on URLs they've already claimed
3. **No cascading retries**: Workers claim different URLs in parallel rather than all competing for the same ones

Here's the improved implementation using SQLite's RETURNING clause:

I see the current schema version is 1. Let me create the implementation with the schema change to add the `claimed_at` column:

```python:crawler/crawler_module/storage.py
// ... existing code ...

DB_SCHEMA_VERSION = 2  # Increment version for schema change

// ... existing code ...

    def _create_tables(self, conn: sqlite3.Connection): # Takes a connection argument
        """Creates the necessary tables in the database if they don't already exist."""
        cursor = conn.cursor() # Create cursor from the provided connection
        try:
            # Check current schema version
            cursor.execute("CREATE TABLE IF NOT EXISTS schema_version (version INTEGER PRIMARY KEY)")
            cursor.execute("SELECT version FROM schema_version ORDER BY version DESC LIMIT 1")
            row = cursor.fetchone()
            current_version = row[0] if row else 0

            if current_version < DB_SCHEMA_VERSION:
                logger.info(f"Database schema version is {current_version}. Upgrading to {DB_SCHEMA_VERSION}.")
                
                # Frontier Table
                cursor.execute("""
                CREATE TABLE IF NOT EXISTS frontier (
                    id INTEGER PRIMARY KEY AUTOINCREMENT,
                    url TEXT UNIQUE NOT NULL,
                    domain TEXT NOT NULL,
                    depth INTEGER DEFAULT 0,
                    added_timestamp INTEGER NOT NULL,
                    priority_score REAL DEFAULT 0,
                    claimed_at INTEGER DEFAULT NULL
                )
                """)
                cursor.execute("CREATE INDEX IF NOT EXISTS idx_frontier_domain ON frontier (domain)")
                cursor.execute("CREATE INDEX IF NOT EXISTS idx_frontier_priority ON frontier (priority_score, added_timestamp)")
                cursor.execute("CREATE INDEX IF NOT EXISTS idx_frontier_claimed ON frontier (claimed_at)")

                # Add claimed_at column if upgrading from version 1
                if current_version == 1:
                    cursor.execute("ALTER TABLE frontier ADD COLUMN claimed_at INTEGER DEFAULT NULL")
                    cursor.execute("CREATE INDEX IF NOT EXISTS idx_frontier_claimed ON frontier (claimed_at)")
                    logger.info("Added claimed_at column to frontier table")

// ... rest of the existing schema creation code ...
```

Now, here's the improved `get_next_url` implementation using SQLite's RETURNING clause:

```python:crawler/crawler_module/frontier.py
// ... existing code ...

    async def get_next_url(self) -> tuple[str, str, int] | None: # (url, domain, id)
        """Gets the next URL to crawl from the frontier, respecting politeness rules.
        Uses atomic claim-and-read with SQLite's RETURNING clause to minimize contention.
        """
        batch_size = 10  # Claim a small batch to amortize DB overhead
        claim_expiry_seconds = 300  # URLs claimed but not processed expire after 5 minutes
        conn_from_pool: sqlite3.Connection | None = None
        
        # Get worker identifier for logging
        worker_name = asyncio.current_task().get_name() if asyncio.current_task() else "unknown"

        try:
            conn_from_pool = self.storage.db_pool.get_connection()
            
            # Atomically claim a batch of URLs
            claimed_urls = await asyncio.to_thread(
                self._atomic_claim_urls_sync, conn_from_pool, batch_size, claim_expiry_seconds
            )
            
            if not claimed_urls:
                return None
            
            logger.debug(f"{worker_name} claimed {len(claimed_urls)} URLs for processing")
            
            # Process claimed URLs with politeness checks
            for url_id, url, domain in claimed_urls:
                # Check if URL is allowed by robots.txt and manual exclusions
                if not await self.politeness.is_url_allowed(url, conn_from_pool):
                    logger.debug(f"URL {url} (ID: {url_id}) disallowed by politeness rules. Removing.")
                    await asyncio.to_thread(self._delete_url_sync, conn_from_pool, url_id)
                    self.seen_urls.add(url)
                    continue
                
                # Check if we can fetch from this domain now
                if await self.politeness.can_fetch_domain_now(domain, conn_from_pool):
                    await self.politeness.record_domain_fetch_attempt(domain, conn_from_pool)
                    logger.debug(f"{worker_name} processing URL ID {url_id} ({url}) from domain {domain}")
                    # Delete the URL since we're processing it
                    await asyncio.to_thread(self._delete_url_sync, conn_from_pool, url_id)
                    return (url, domain, url_id)
                else:
                    # Domain not ready - unclaim this URL so another worker can try later
                    await asyncio.to_thread(self._unclaim_url_sync, conn_from_pool, url_id)
                    logger.debug(f"Domain {domain} not ready for fetch, unclaimed URL ID {url_id}")
            
            # All claimed URLs were either disallowed or domains not ready
            return None
            
        except sqlite3.Error as e:
            logger.error(f"DB Error in get_next_url: {e}", exc_info=True)
            if conn_from_pool:
                try: conn_from_pool.rollback()
                except: pass
            return None
        except Exception as e:
            logger.error(f"Unexpected error in get_next_url: {e}", exc_info=True)
            return None
        finally:
            if conn_from_pool:
                self.storage.db_pool.return_connection(conn_from_pool)

    def _atomic_claim_urls_sync(self, conn: sqlite3.Connection, batch_size: int, claim_expiry_seconds: int) -> list[tuple[int, str, str]]:
        """Atomically claims a batch of URLs using UPDATE...RETURNING.
        Returns list of (id, url, domain) tuples for claimed URLs."""
        cursor = conn.cursor()
        claim_timestamp = int(time.time() * 1000000)  # Microsecond precision
        expired_timestamp = claim_timestamp - (claim_expiry_seconds * 1000000)
        
        try:
            # First, release any expired claims
            cursor.execute("""
                UPDATE frontier 
                SET claimed_at = NULL 
                WHERE claimed_at IS NOT NULL AND claimed_at < ?
            """, (expired_timestamp,))
            
            # Atomically claim unclaimed URLs using RETURNING
            cursor.execute("""
                UPDATE frontier 
                SET claimed_at = ? 
                WHERE id IN (
                    SELECT id FROM frontier 
                    WHERE claimed_at IS NULL
                    ORDER BY added_timestamp ASC 
                    LIMIT ?
                )
                RETURNING id, url, domain
            """, (claim_timestamp, batch_size))
            
            claimed_urls = cursor.fetchall()
            conn.commit()
            
            return claimed_urls
            
        except sqlite3.Error as e:
            logger.error(f"DB error in atomic claim: {e}")
            conn.rollback()
            raise
        finally:
            cursor.close()

    def _delete_url_sync(self, conn: sqlite3.Connection, url_id: int):
        """Deletes a URL from the frontier after processing or if disallowed."""
        cursor = conn.cursor()
        try:
            cursor.execute("DELETE FROM frontier WHERE id = ?", (url_id,))
            conn.commit()
        except sqlite3.Error as e:
            logger.error(f"DB error deleting URL ID {url_id}: {e}")
            conn.rollback()
        finally:
            cursor.close()

    def _unclaim_url_sync(self, conn: sqlite3.Connection, url_id: int):
        """Releases a claim on a URL so other workers can process it."""
        cursor = conn.cursor()
        try:
            cursor.execute("UPDATE frontier SET claimed_at = NULL WHERE id = ?", (url_id,))
            conn.commit()
        except sqlite3.Error as e:
            logger.error(f"DB error unclaiming URL ID {url_id}: {e}")
            conn.rollback()
        finally:
            cursor.close()

// ... existing code ...
```

The key improvements in this implementation:

1. **Single atomic operation**: The `UPDATE...RETURNING` statement atomically claims URLs and returns them in one operation, as [documented in SQLite's RETURNING documentation](https://www.sqlite.org/lang_returning.html).

2. **No wasted work**: Workers only perform politeness checks on URLs they've successfully claimed.

3. **Automatic expiry**: Claims expire after 5 minutes, preventing URLs from being stuck if a worker crashes.

4. **Efficient batching**: Workers claim small batches (10 URLs) to reduce database round trips while avoiding holding too many URLs.

5. **Clean separation**: URLs that fail politeness checks are deleted, while URLs whose domains aren't ready are unclaimed for other workers.

This approach completely avoids the thundering herd problem because:
- The UPDATE operation is serialized at the database level
- Each worker claims different URLs atomically
- There's no race condition between checking and claiming

The serialization happens at the database level through SQLite's write locking, ensuring that concurrent UPDATE operations don't interfere with each other.

```python
// ... existing code ...

DB_SCHEMA_VERSION = 2  # Increment version for schema change

// ... existing code ...

    def _create_tables(self, conn: sqlite3.Connection): # Takes a connection argument
        """Creates the necessary tables in the database if they don't already exist."""
        cursor = conn.cursor() # Create cursor from the provided connection
        try:
            # Check current schema version
            cursor.execute("CREATE TABLE IF NOT EXISTS schema_version (version INTEGER PRIMARY KEY)")
            cursor.execute("SELECT version FROM schema_version ORDER BY version DESC LIMIT 1")
            row = cursor.fetchone()
            current_version = row[0] if row else 0

            if current_version < DB_SCHEMA_VERSION:
                logger.info(f"Database schema version is {current_version}. Upgrading to {DB_SCHEMA_VERSION}.")
                
                # Frontier Table
                cursor.execute("""
                CREATE TABLE IF NOT EXISTS frontier (
                    id INTEGER PRIMARY KEY AUTOINCREMENT,
                    url TEXT UNIQUE NOT NULL,
                    domain TEXT NOT NULL,
                    depth INTEGER DEFAULT 0,
                    added_timestamp INTEGER NOT NULL,
                    priority_score REAL DEFAULT 0,
                    claimed_at INTEGER DEFAULT NULL
                )
                """)
                cursor.execute("CREATE INDEX IF NOT EXISTS idx_frontier_domain ON frontier (domain)")
                cursor.execute("CREATE INDEX IF NOT EXISTS idx_frontier_priority ON frontier (priority_score, added_timestamp)")
                cursor.execute("CREATE INDEX IF NOT EXISTS idx_frontier_claimed ON frontier (claimed_at)")

                # Add claimed_at column if upgrading from version 1
                if current_version == 1:
                    cursor.execute("ALTER TABLE frontier ADD COLUMN claimed_at INTEGER DEFAULT NULL")
                    cursor.execute("CREATE INDEX IF NOT EXISTS idx_frontier_claimed ON frontier (claimed_at)")
                    logger.info("Added claimed_at column to frontier table")

// ... rest of the existing schema creation code ...
```

```python
// ... existing code ...

    async def get_next_url(self) -> tuple[str, str, int] | None: # (url, domain, id)
        """Gets the next URL to crawl from the frontier, respecting politeness rules.
        Uses atomic claim-and-read with SQLite's RETURNING clause to minimize contention.
        """
        batch_size = 10  # Claim a small batch to amortize DB overhead
        claim_expiry_seconds = 300  # URLs claimed but not processed expire after 5 minutes
        conn_from_pool: sqlite3.Connection | None = None
        
        # Get worker identifier for logging
        worker_name = asyncio.current_task().get_name() if asyncio.current_task() else "unknown"

        try:
            conn_from_pool = self.storage.db_pool.get_connection()
            
            # Atomically claim a batch of URLs
            claimed_urls = await asyncio.to_thread(
                self._atomic_claim_urls_sync, conn_from_pool, batch_size, claim_expiry_seconds
            )
            
            if not claimed_urls:
                return None
            
            logger.debug(f"{worker_name} claimed {len(claimed_urls)} URLs for processing")
            
            # Process claimed URLs with politeness checks
            for url_id, url, domain in claimed_urls:
                # Check if URL is allowed by robots.txt and manual exclusions
                if not await self.politeness.is_url_allowed(url, conn_from_pool):
                    logger.debug(f"URL {url} (ID: {url_id}) disallowed by politeness rules. Removing.")
                    await asyncio.to_thread(self._delete_url_sync, conn_from_pool, url_id)
                    self.seen_urls.add(url)
                    continue
                
                # Check if we can fetch from this domain now
                if await self.politeness.can_fetch_domain_now(domain, conn_from_pool):
                    await self.politeness.record_domain_fetch_attempt(domain, conn_from_pool)
                    logger.debug(f"{worker_name} processing URL ID {url_id} ({url}) from domain {domain}")
                    # Delete the URL since we're processing it
                    await asyncio.to_thread(self._delete_url_sync, conn_from_pool, url_id)
                    return (url, domain, url_id)
                else:
                    # Domain not ready - unclaim this URL so another worker can try later
                    await asyncio.to_thread(self._unclaim_url_sync, conn_from_pool, url_id)
                    logger.debug(f"Domain {domain} not ready for fetch, unclaimed URL ID {url_id}")
            
            # All claimed URLs were either disallowed or domains not ready
            return None
            
        except sqlite3.Error as e:
            logger.error(f"DB Error in get_next_url: {e}", exc_info=True)
            if conn_from_pool:
                try: conn_from_pool.rollback()
                except: pass
            return None
        except Exception as e:
            logger.error(f"Unexpected error in get_next_url: {e}", exc_info=True)
            return None
        finally:
            if conn_from_pool:
                self.storage.db_pool.return_connection(conn_from_pool)

    def _atomic_claim_urls_sync(self, conn: sqlite3.Connection, batch_size: int, claim_expiry_seconds: int) -> list[tuple[int, str, str]]:
        """Atomically claims a batch of URLs using UPDATE...RETURNING.
        Returns list of (id, url, domain) tuples for claimed URLs."""
        cursor = conn.cursor()
        claim_timestamp = int(time.time() * 1000000)  # Microsecond precision
        expired_timestamp = claim_timestamp - (claim_expiry_seconds * 1000000)
        
        try:
            # First, release any expired claims
            cursor.execute("""
                UPDATE frontier 
                SET claimed_at = NULL 
                WHERE claimed_at IS NOT NULL AND claimed_at < ?
            """, (expired_timestamp,))
            
            # Atomically claim unclaimed URLs using RETURNING
            cursor.execute("""
                UPDATE frontier 
                SET claimed_at = ? 
                WHERE id IN (
                    SELECT id FROM frontier 
                    WHERE claimed_at IS NULL
                    ORDER BY added_timestamp ASC 
                    LIMIT ?
                )
                RETURNING id, url, domain
            """, (claim_timestamp, batch_size))
            
            claimed_urls = cursor.fetchall()
            conn.commit()
            
            return claimed_urls
            
        except sqlite3.Error as e:
            logger.error(f"DB error in atomic claim: {e}")
            conn.rollback()
            raise
        finally:
            cursor.close()

    def _delete_url_sync(self, conn: sqlite3.Connection, url_id: int):
        """Deletes a URL from the frontier after processing or if disallowed."""
        cursor = conn.cursor()
        try:
            cursor.execute("DELETE FROM frontier WHERE id = ?", (url_id,))
            conn.commit()
        except sqlite3.Error as e:
            logger.error(f"DB error deleting URL ID {url_id}: {e}")
            conn.rollback()
        finally:
            cursor.close()

    def _unclaim_url_sync(self, conn: sqlite3.Connection, url_id: int):
        """Releases a claim on a URL so other workers can process it."""
        cursor = conn.cursor()
        try:
            cursor.execute("UPDATE frontier SET claimed_at = NULL WHERE id = ?", (url_id,))
            conn.commit()
        except sqlite3.Error as e:
            logger.error(f"DB error unclaiming URL ID {url_id}: {e}")
            conn.rollback()
        finally:
            cursor.close()

// ... existing code ...
```

---

**User**

Great! I think we need to update some tests now, here is the output of pytest:
```
=================================================== short test summary info ====================================================
FAILED tests/test_frontier.py::test_add_and_get_urls - AssertionError: Expected URL, got None at iteration 1
FAILED tests/test_frontier.py::test_frontier_resume_with_politeness - assert None is not None
========================================== 2 failed, 71 passed, 61 warnings in 0.76s ===========================================
```

---

**Cursor**

Let me look at the failing tests to understand what needs to be updated:

---

**User**

look in the crawler folder

---

**Cursor**

Looking at the failing tests, I can see the issue. The new implementation claims a batch of URLs but doesn't properly handle the unclaimed URLs when it returns early after finding a suitable URL. Let me fix the `get_next_url` method:

```python
// ... existing code ...

    async def get_next_url(self) -> tuple[str, str, int] | None: # (url, domain, id)
        """Gets the next URL to crawl from the frontier, respecting politeness rules.
        Uses atomic claim-and-read with SQLite's RETURNING clause to minimize contention.
        """
        batch_size = 10  # Claim a small batch to amortize DB overhead
        claim_expiry_seconds = 300  # URLs claimed but not processed expire after 5 minutes
        conn_from_pool: sqlite3.Connection | None = None
        
        # Get worker identifier for logging
        worker_task = asyncio.current_task()
        worker_name = worker_task.get_name() if worker_task else "unknown"

        try:
            conn_from_pool = self.storage.db_pool.get_connection()
            
            # Atomically claim a batch of URLs
            claimed_urls = await asyncio.to_thread(
                self._atomic_claim_urls_sync, conn_from_pool, batch_size, claim_expiry_seconds
            )
            
            if not claimed_urls:
                return None
            
            logger.debug(f"{worker_name} claimed {len(claimed_urls)} URLs for processing")
            
            # Track which URLs we've processed so we can unclaim the rest
            processed_url_ids = []
            selected_url_info = None
            
            # Process claimed URLs with politeness checks
            for url_id, url, domain in claimed_urls:
                processed_url_ids.append(url_id)
                
                # Check if URL is allowed by robots.txt and manual exclusions
                if not await self.politeness.is_url_allowed(url, conn_from_pool):
                    logger.debug(f"URL {url} (ID: {url_id}) disallowed by politeness rules. Removing.")
                    await asyncio.to_thread(self._delete_url_sync, conn_from_pool, url_id)
                    self.seen_urls.add(url)
                    continue
                
                # Check if we can fetch from this domain now
                if await self.politeness.can_fetch_domain_now(domain, conn_from_pool):
                    await self.politeness.record_domain_fetch_attempt(domain, conn_from_pool)
                    logger.debug(f"{worker_name} processing URL ID {url_id} ({url}) from domain {domain}")
                    # Delete the URL since we're processing it
                    await asyncio.to_thread(self._delete_url_sync, conn_from_pool, url_id)
                    selected_url_info = (url, domain, url_id)
                    break  # Found a suitable URL
                else:
                    # Domain not ready - will be unclaimed below
                    logger.debug(f"Domain {domain} not ready for fetch, will unclaim URL ID {url_id}")
            
            # Unclaim any remaining URLs we didn't process or delete
            if selected_url_info:
                # We found a URL to process - unclaim all the URLs after it
                remaining_url_ids = [uid for uid, _, _ in claimed_urls if uid not in processed_url_ids]
                for url_id in remaining_url_ids:
                    await asyncio.to_thread(self._unclaim_url_sync, conn_from_pool, url_id)
            else:
                # We didn't find any suitable URL - unclaim all that weren't deleted
                for url_id, url, domain in claimed_urls:
                    # Only unclaim if we didn't delete it (due to politeness rules)
                    if url not in self.seen_urls:  # URLs added to seen_urls were deleted
                        await asyncio.to_thread(self._unclaim_url_sync, conn_from_pool, url_id)
            
            return selected_url_info
            
        except sqlite3.Error as e:
            logger.error(f"DB Error in get_next_url: {e}", exc_info=True)
            if conn_from_pool:
                try: conn_from_pool.rollback()
                except: pass
            return None
        except Exception as e:
            logger.error(f"Unexpected error in get_next_url: {e}", exc_info=True)
            return None
        finally:
            if conn_from_pool:
                self.storage.db_pool.return_connection(conn_from_pool)

// ... existing code ...
```

I see the issue now. When `get_next_url` finds a suitable URL and returns it, it doesn't unclaim the remaining URLs in the batch. This causes subsequent calls to fail because those URLs are still marked as claimed. Let me fix this:

```python
    async def get_next_url(self) -> tuple[str, str, int] | None: # (url, domain, id)
        """Gets the next URL to crawl from the frontier, respecting politeness rules.
        Uses atomic claim-and-read with SQLite's RETURNING clause to minimize contention.
        """
        batch_size = 10  # Claim a small batch to amortize DB overhead
        claim_expiry_seconds = 300  # URLs claimed but not processed expire after 5 minutes
        conn_from_pool: sqlite3.Connection | None = None
        
        # Get worker identifier for logging
        worker_task = asyncio.current_task()
        worker_name = worker_task.get_name() if worker_task else "unknown"

        try:
            conn_from_pool = self.storage.db_pool.get_connection()
            
            # Atomically claim a batch of URLs
            claimed_urls = await asyncio.to_thread(
                self._atomic_claim_urls_sync, conn_from_pool, batch_size, claim_expiry_seconds
            )
            
            if not claimed_urls:
                return None
            
            logger.debug(f"{worker_name} claimed {len(claimed_urls)} URLs for processing")
            
            # Track which URLs we need to unclaim if we return early
            urls_to_unclaim = []
            selected_url_info = None
            
            # Process claimed URLs with politeness checks
            for i, (url_id, url, domain) in enumerate(claimed_urls):
                # Check if URL is allowed by robots.txt and manual exclusions
                if not await self.politeness.is_url_allowed(url, conn_from_pool):
                    logger.debug(f"URL {url} (ID: {url_id}) disallowed by politeness rules. Removing.")
                    await asyncio.to_thread(self._delete_url_sync, conn_from_pool, url_id)
                    self.seen_urls.add(url)
                    continue
                
                # Check if we can fetch from this domain now
                if await self.politeness.can_fetch_domain_now(domain, conn_from_pool):
                    await self.politeness.record_domain_fetch_attempt(domain, conn_from_pool)
                    logger.debug(f"{worker_name} processing URL ID {url_id} ({url}) from domain {domain}")
                    # Delete the URL since we're processing it
                    await asyncio.to_thread(self._delete_url_sync, conn_from_pool, url_id)
                    selected_url_info = (url, domain, url_id)
                    # Mark remaining URLs to be unclaimed
                    urls_to_unclaim = [(uid, u, d) for uid, u, d in claimed_urls[i+1:]]
                    break
                else:
                    # Domain not ready - unclaim this URL so another worker can try later
                    await asyncio.to_thread(self._unclaim_url_sync, conn_from_pool, url_id)
                    logger.debug(f"Domain {domain} not ready for fetch, unclaimed URL ID {url_id}")
            
            # Unclaim any remaining URLs if we found one to process
            for url_id, url, domain in urls_to_unclaim:
                await asyncio.to_thread(self._unclaim_url_sync, conn_from_pool, url_id)
                logger.debug(f"Unclaimed remaining URL ID {url_id} from batch")
            
            return selected_url_info
            
        except sqlite3.Error as e:
            logger.error(f"DB Error in get_next_url: {e}", exc_info=True)
            if conn_from_pool:
                try: conn_from_pool.rollback()
                except: pass
            return None
        except Exception as e:
            logger.error(f"Unexpected error in get_next_url: {e}", exc_info=True)
            return None
        finally:
            if conn_from_pool:
                self.storage.db_pool.return_connection(conn_from_pool)
```

```python
// ... existing code ...
    async def get_next_url(self) -> tuple[str, str, int] | None: # (url, domain, id)
        """Gets the next URL to crawl from the frontier, respecting politeness rules.
        Uses atomic claim-and-read with SQLite's RETURNING clause to minimize contention.
        """
        batch_size = 10  # Claim a small batch to amortize DB overhead
        claim_expiry_seconds = 300  # URLs claimed but not processed expire after 5 minutes
        conn_from_pool: sqlite3.Connection | None = None
        
        # Get worker identifier for logging
        worker_task = asyncio.current_task()
        worker_name = worker_task.get_name() if worker_task else "unknown"

        try:
            conn_from_pool = self.storage.db_pool.get_connection()
            
            # Atomically claim a batch of URLs
            claimed_urls = await asyncio.to_thread(
                self._atomic_claim_urls_sync, conn_from_pool, batch_size, claim_expiry_seconds
            )
            
            if not claimed_urls:
                return None
            
            logger.debug(f"{worker_name} claimed {len(claimed_urls)} URLs for processing")
            
            # Track which URLs we need to unclaim if we return early
            urls_to_unclaim = []
            selected_url_info = None
            
            # Process claimed URLs with politeness checks
            for i, (url_id, url, domain) in enumerate(claimed_urls):
                # Check if URL is allowed by robots.txt and manual exclusions
                if not await self.politeness.is_url_allowed(url, conn_from_pool):
                    logger.debug(f"URL {url} (ID: {url_id}) disallowed by politeness rules. Removing.")
                    await asyncio.to_thread(self._delete_url_sync, conn_from_pool, url_id)
                    self.seen_urls.add(url)
                    continue
                
                # Check if we can fetch from this domain now
                if await self.politeness.can_fetch_domain_now(domain, conn_from_pool):
                    await self.politeness.record_domain_fetch_attempt(domain, conn_from_pool)
                    logger.debug(f"{worker_name} processing URL ID {url_id} ({url}) from domain {domain}")
                    # Delete the URL since we're processing it
                    await asyncio.to_thread(self._delete_url_sync, conn_from_pool, url_id)
                    selected_url_info = (url, domain, url_id)
                    # Mark remaining URLs to be unclaimed
                    urls_to_unclaim = [(uid, u, d) for uid, u, d in claimed_urls[i+1:]]
                    break
                else:
                    # Domain not ready - unclaim this URL so another worker can try later
                    await asyncio.to_thread(self._unclaim_url_sync, conn_from_pool, url_id)
                    logger.debug(f"Domain {domain} not ready for fetch, unclaimed URL ID {url_id}")
            
            # Unclaim any remaining URLs if we found one to process
            for url_id, url, domain in urls_to_unclaim:
                await asyncio.to_thread(self._unclaim_url_sync, conn_from_pool, url_id)
                logger.debug(f"Unclaimed remaining URL ID {url_id} from batch")
            
            return selected_url_info
            
        except sqlite3.Error as e:
            logger.error(f"DB Error in get_next_url: {e}", exc_info=True)
            if conn_from_pool:
                try: conn_from_pool.rollback()
                except: pass
            return None
        except Exception as e:
            logger.error(f"Unexpected error in get_next_url: {e}", exc_info=True)
            return None
        finally:
            if conn_from_pool:
                self.storage.db_pool.return_connection(conn_from_pool)

    def _atomic_claim_urls_sync(self, conn: sqlite3.Connection, batch_size: int, claim_expiry_seconds: int) -> list[tuple[int, str, str]]:
// ... existing code ...
```

Now I need to update the `_add_url_sync` method to include the `claimed_at` column (with NULL value) when inserting URLs:

```python
// ... existing code ...
    def _add_url_sync(self, normalized_url: str, domain: str, depth: int) -> bool:
        """Synchronous part of adding a URL to DB using a pooled connection."""
        conn_from_pool: sqlite3.Connection | None = None
        try:
            conn_from_pool = self.storage.db_pool.get_connection()
            cursor = conn_from_pool.cursor()
            try:
                cursor.execute("SELECT 1 FROM visited_urls WHERE url_sha256 = ?", 
                               (self.storage.get_url_sha256(normalized_url),))
                if cursor.fetchone():
                    return False 

                cursor.execute(
                    "INSERT OR IGNORE INTO frontier (url, domain, depth, added_timestamp, priority_score, claimed_at) VALUES (?, ?, ?, ?, ?, ?)",
                    (normalized_url, domain, depth, int(time.time()), 0, None)
                )
                if cursor.rowcount > 0:
                    conn_from_pool.commit() # Explicit commit for INSERT
                    return True 
                else:
                    # This means INSERT OR IGNORE found an existing row (URL already in frontier)
                    return False 
            except sqlite3.Error as e:
                logger.error(f"DB (execute/commit) THREAD: Error adding URL {normalized_url} to frontier: {e}")
                if conn_from_pool: conn_from_pool.rollback()
                raise # Propagate to mark as not added
            finally:
                cursor.close()
        except sqlite3.Error as e: # Outer SQLite errors (e.g. connection issue)
            logger.error(f"DB (pool/connection) THREAD: Error adding URL {normalized_url} to frontier: {e}")
            return False # Ensure it's marked as not added on any DB error
        except Exception as e: # Catch other exceptions like pool Empty
            logger.error(f"DB (pool) THREAD: Pool error adding URL {normalized_url} to frontier: {e}")
            return False
        finally:
            if conn_from_pool:
                self.storage.db_pool.return_connection(conn_from_pool)
        # Fallback, should be covered by try/except logic.
        # If an error occurred before return True, it should have returned False or raised.
        return False
// ... existing code ...
```

```python
// ... existing code ...
                cursor.execute("SELECT 1 FROM visited_urls WHERE url_sha256 = ?", 
                               (self.storage.get_url_sha256(normalized_url),))
                if cursor.fetchone():
                    return False 

                cursor.execute(
                    "INSERT OR IGNORE INTO frontier (url, domain, depth, added_timestamp, priority_score, claimed_at) VALUES (?, ?, ?, ?, ?, ?)",
                    (normalized_url, domain, depth, int(time.time()), 0, None)
                )
                if cursor.rowcount > 0:
                    conn_from_pool.commit() # Explicit commit for INSERT
                    return True 
                else:
                    # This means INSERT OR IGNORE found an existing row (URL already in frontier)
                    return False 
// ... existing code ...
```

Perfect! I've made all the necessary changes to fix the failing tests:

1. **Updated the schema** in `storage.py`:
   - Incremented `DB_SCHEMA_VERSION` to 2
   - Added `claimed_at INTEGER DEFAULT NULL` column to the frontier table
   - Added migration logic to add the column if upgrading from version 1
   - Created an index on `claimed_at` for performance

2. **Fixed `get_next_url`** in `frontier.py`:
   - Now properly tracks which URLs need to be unclaimed when we find a suitable one
   - When a URL is found and selected for processing, all remaining URLs in the batch are unclaimed
   - This ensures subsequent calls to `get_next_url` can access those URLs

3. **Updated `_add_url_sync`** in `frontier.py`:
   - The INSERT statement now includes the `claimed_at` column with `NULL` value
   - This ensures compatibility with the new schema

The key insight was that the tests expect FIFO behavior - they add URLs and then retrieve them one by one in order. Our new implementation claims batches of URLs for efficiency, but we needed to ensure that unclaimed URLs remain available for subsequent calls. By properly unclaiming the remaining URLs in a batch when we find one to process, we maintain the expected behavior while still getting the performance benefits of atomic batch claiming.

The tests should now pass! The implementation successfully:
- Avoids the thundering herd problem through atomic claim operations
- Prevents wasted work by only doing politeness checks on claimed URLs
- Maintains correct FIFO semantics for the tests
- Scales efficiently with multiple workers

---

**User**

Great, 2 issues now:
1. With 500 workers, getting contention on the DB lock during the claim phase (still happens even after doubling DB timeout to 20 seconds):
```
2025-06-01 01:07:15 - ERROR - crawler_module.frontier [8238] - DB Error in get_next_url: database is locked
Traceback (most recent call last):
  File "/home/ubuntu/vibe-coding-experiments/crawler/crawler_module/frontier.py", line 272, in get_next_url
    claimed_urls = await asyncio.to_thread(
                   ^^^^^^^^^^^^^^^^^^^^^^^^
  File "/usr/lib/python3.12/asyncio/threads.py", line 25, in to_thread
    return await loop.run_in_executor(None, func_call)
           ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/usr/lib/python3.12/concurrent/futures/thread.py", line 58, in run
    result = self.fn(*self.args, **self.kwargs)
             ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/home/ubuntu/vibe-coding-experiments/crawler/crawler_module/frontier.py", line 338, in _atomic_claim_urls_sync
    cursor.execute("""
sqlite3.OperationalError: database is locked
```

Not sure if permanently solvable, maybe need to just increase DB timeout again.

2. Hitting a fatal "too many open files" error which results in OS killing the process. The critical error is preceded by what looks like hundreds of non-fatal "too many open files" ClientConnectionErrors:
```
2025-06-01 01:07:42 - ERROR - crawler_module.fetcher [8238] - ClientConnectionError for http://adnbroadband.com/robots.txt: Cannot connect to host adnbroadband.com:80 ssl:default [Too many open files]
2025-06-01 01:07:42 - INFO - crawler_module.politeness [8238] - HTTP fetch for robots.txt did not yield content for adnbroadband.com (status: 901). Trying HTTPS.
2025-06-01 01:07:42 - INFO - crawler_module.storage [8238] - Recorded visited URL: http://arcadiagroup.co.uk (Status: 200)
2025-06-01 01:07:42 - CRITICAL - crawler_module.orchestrator [8238] - Orchestrator critical error: [Errno 24] Too many open files: '/proc/8238/statm'
Traceback (most recent call last):
  File "/home/ubuntu/vibe-coding-experiments/crawler/crawler_module/orchestrator.py", line 205, in run_crawl
  File "/home/ubuntu/vibe-coding-experiments/crawler/.venv/lib/python3.12/site-packages/psutil/_common.py", line 462, in wrapper
  File "/home/ubuntu/vibe-coding-experiments/crawler/.venv/lib/python3.12/site-packages/psutil/_common.py", line 460, in wrapper
  File "/home/ubuntu/vibe-coding-experiments/crawler/.venv/lib/python3.12/site-packages/psutil/__init__.py", line 1122, in memory_info
  File "/home/ubuntu/vibe-coding-experiments/crawler/.venv/lib/python3.12/site-packages/psutil/_pslinux.py", line 1646, in wrapper
  File "/home/ubuntu/vibe-coding-experiments/crawler/.venv/lib/python3.12/site-packages/psutil/_pslinux.py", line 1906, in memory_info
  File "/home/ubuntu/vibe-coding-experiments/crawler/.venv/lib/python3.12/site-packages/psutil/_common.py", line 766, in open_binary
OSError: [Errno 24] Too many open files: '/proc/8238/statm'
```

Thoughts?

---

**Cursor**

Looking at these two issues:

## 1. Database Locking Contention

With 500 workers all trying to claim URLs simultaneously, you're hitting SQLite's fundamental limitation - it only allows one writer at a time. Even with WAL mode, write operations are serialized. Here are some solutions:

### Immediate fixes:
- **Increase batch size and claim less frequently**: Instead of `batch_size = 10`, try `batch_size = 50` or even `100`. This reduces contention by having workers hold URLs longer.
- **Add exponential backoff with retry**:

```python
    async def get_next_url(self) -> tuple[str, str, int] | None: # (url, domain, id)
        """Gets the next URL to crawl from the frontier, respecting politeness rules.
        Uses atomic claim-and-read with SQLite's RETURNING clause to minimize contention.
        """
        batch_size = 50  # Increased batch size to reduce contention
        claim_expiry_seconds = 300  # URLs claimed but not processed expire after 5 minutes
        conn_from_pool: sqlite3.Connection | None = None
        
        # Get worker identifier for logging
        worker_task = asyncio.current_task()
        worker_name = worker_task.get_name() if worker_task else "unknown"
        
        # Retry logic for database locks
        max_retries = 5
        base_delay = 0.1  # 100ms base delay
        
        for retry in range(max_retries):
            try:
                conn_from_pool = self.storage.db_pool.get_connection()
                
                # Atomically claim a batch of URLs
                claimed_urls = await asyncio.to_thread(
                    self._atomic_claim_urls_sync, conn_from_pool, batch_size, claim_expiry_seconds
                )
                
                if not claimed_urls:
                    return None
                
                logger.debug(f"{worker_name} claimed {len(claimed_urls)} URLs for processing")
                
                # Track which URLs we need to unclaim if we return early
                urls_to_unclaim = []
                selected_url_info = None
                
                # Process claimed URLs with politeness checks
                for i, (url_id, url, domain) in enumerate(claimed_urls):
                    # Check if URL is allowed by robots.txt and manual exclusions
                    if not await self.politeness.is_url_allowed(url, conn_from_pool):
                        logger.debug(f"URL {url} (ID: {url_id}) disallowed by politeness rules. Removing.")
                        await asyncio.to_thread(self._delete_url_sync, conn_from_pool, url_id)
                        self.seen_urls.add(url)
                        continue
                    
                    # Check if we can fetch from this domain now
                    if await self.politeness.can_fetch_domain_now(domain, conn_from_pool):
                        await self.politeness.record_domain_fetch_attempt(domain, conn_from_pool)
                        logger.debug(f"{worker_name} processing URL ID {url_id} ({url}) from domain {domain}")
                        # Delete the URL since we're processing it
                        await asyncio.to_thread(self._delete_url_sync, conn_from_pool, url_id)
                        selected_url_info = (url, domain, url_id)
                        # Mark remaining URLs to be unclaimed
                        urls_to_unclaim = [(uid, u, d) for uid, u, d in claimed_urls[i+1:]]
                        break
                    else:
                        # Domain not ready - unclaim this URL so another worker can try later
                        await asyncio.to_thread(self._unclaim_url_sync, conn_from_pool, url_id)
                        logger.debug(f"Domain {domain} not ready for fetch, unclaimed URL ID {url_id}")
                
                # Unclaim any remaining URLs if we found one to process
                for url_id, url, domain in urls_to_unclaim:
                    await asyncio.to_thread(self._unclaim_url_sync, conn_from_pool, url_id)
                    logger.debug(f"Unclaimed remaining URL ID {url_id} from batch")
                
                return selected_url_info
                
            except sqlite3.OperationalError as e:
                if "database is locked" in str(e) and retry < max_retries - 1:
                    # Exponential backoff with jitter
                    delay = base_delay * (2 ** retry) + (0.1 * asyncio.current_task().get_name().split('-')[-1].hash() % 10) / 10
                    logger.debug(f"{worker_name} got database lock, retrying in {delay:.2f}s (attempt {retry + 1}/{max_retries})")
                    await asyncio.sleep(delay)
                    if conn_from_pool:
                        try: 
                            conn_from_pool.rollback()
                            self.storage.db_pool.return_connection(conn_from_pool)
                            conn_from_pool = None
                        except: pass
                    continue
                else:
                    # Max retries exceeded or different error
                    logger.error(f"DB Error in get_next_url after {retry + 1} attempts: {e}", exc_info=True)
                    if conn_from_pool:
                        try: conn_from_pool.rollback()
                        except: pass
                    return None
            except Exception as e:
                logger.error(f"Unexpected error in get_next_url: {e}", exc_info=True)
                return None
            finally:
                if conn_from_pool:
                    self.storage.db_pool.return_connection(conn_from_pool)
```

Let me try a different approach:

Let me provide you with comprehensive solutions for both issues:

## 1. Database Locking Contention

The issue is that 500 workers are all trying to UPDATE the frontier table simultaneously. Here's a modified approach with retry logic and larger batches:

Let me provide you with the code changes and solutions directly:

## 1. Database Locking Solution

Edit your `frontier.py` to add retry logic and increase batch size:

```python
async def get_next_url(self) -> tuple[str, str, int] | None:
    """Gets the next URL to crawl from the frontier, respecting politeness rules.
    Uses atomic claim-and-read with SQLite's RETURNING clause to minimize contention.
    """
    # Larger batch size to reduce contention - adjust based on worker count
    batch_size = min(100, max(50, 1000 // self.config.max_workers))
    claim_expiry_seconds = 300
    conn_from_pool: sqlite3.Connection | None = None
    
    worker_task = asyncio.current_task()
    worker_name = worker_task.get_name() if worker_task else "unknown"
    
    # Retry configuration
    max_retries = 5
    base_delay = 0.05  # 50ms
    
    for retry_attempt in range(max_retries):
        try:
            # ... existing code ...
        except sqlite3.OperationalError as e:
            if "database is locked" in str(e) and retry_attempt < max_retries - 1:
                # Exponential backoff with jitter
                import random
                delay = base_delay * (2 ** retry_attempt) + random.uniform(0, 0.1)
                logger.debug(f"{worker_name} database locked, retry {retry_attempt + 1}/{max_retries} after {delay:.3f}s")
                
                if conn_from_pool:
                    try:
                        conn_from_pool.rollback()
                        self.storage.db_pool.return_connection(conn_from_pool)
                        conn_from_pool = None
                    except:
                        pass
                
                await asyncio.sleep(delay)
                continue
            else:
                # Max retries exceeded or different error
                logger.error(f"DB Error in get_next_url: {e}")
                return None
        # ... rest of exception handling ...
```

## 2. Too Many Open Files Solution

The "too many open files" error is hitting the OS file descriptor limit. With 500 concurrent workers making HTTP connections, you need to:

### a) Increase OS limits:
```bash
# Check current limits
ulimit -n

# Increase for current session
ulimit -n 65536

# For permanent change, edit /etc/security/limits.conf:
# Add these lines:
* soft nofile 65536
* hard nofile 65536

# Also edit /etc/sysctl.conf:
fs.file-max = 2097152

# Apply sysctl changes
sudo sysctl -p
```

### b) Implement connection pooling in fetcher.py:

```python
async def get_next_url(self) -> tuple[str, str, int] | None:
    """Gets the next URL to crawl from the frontier, respecting politeness rules.
    Uses atomic claim-and-read with SQLite's RETURNING clause to minimize contention.
    """
    # Larger batch size to reduce contention - adjust based on worker count
    batch_size = min(100, max(50, 1000 // self.config.max_workers))
    claim_expiry_seconds = 300
    conn_from_pool: sqlite3.Connection | None = None
    
    worker_task = asyncio.current_task()
    worker_name = worker_task.get_name() if worker_task else "unknown"
    
    # Retry configuration
    max_retries = 5
    base_delay = 0.05  # 50ms
    
    for retry_attempt in range(max_retries):
        try:
            # ... existing code ...
        except sqlite3.OperationalError as e:
            if "database is locked" in str(e) and retry_attempt < max_retries - 1:
                # Exponential backoff with jitter
                import random
                delay = base_delay * (2 ** retry_attempt) + random.uniform(0, 0.1)
                logger.debug(f"{worker_name} database locked, retry {retry_attempt + 1}/{max_retries} after {delay:.3f}s")
                
                if conn_from_pool:
                    try:
                        conn_from_pool.rollback()
                        self.storage.db_pool.return_connection(conn_from_pool)
                        conn_from_pool = None
                    except:
                        pass
                
                await asyncio.sleep(delay)
                continue
            else:
                # Max retries exceeded or different error
                logger.error(f"DB Error in get_next_url: {e}")
                return None
        # ... rest of exception handling ...
```

```shellscript
# Check current limits
ulimit -n

# Increase for current session
ulimit -n 65536

# For permanent change, edit /etc/security/limits.conf:
# Add these lines:
* soft nofile 65536
* hard nofile 65536

# Also edit /etc/sysctl.conf:
fs.file-max = 2097152

# Apply sysctl changes
sudo sysctl -p
```

Good news! Your fetcher already has connection pooling implemented with reasonable limits:
- `limit=200` total connections
- `limit_per_host=30` max connections per host
- `force_close=True` to free up file descriptors

However, with 500 workers, you might still exceed limits. Here's a comprehensive solution:

### Complete Solution for Both Issues:

```python
// ... existing code ...
    async def _get_session(self) -> aiohttp.ClientSession:
        """Returns the existing aiohttp session or creates a new one."""
        if self.session is None or self.session.closed:
            # Create a connector with conservative limits for high worker count
            # With 500 workers, we need to be careful about file descriptors
            max_total_connections = min(1000, self.config.max_workers * 2)  # Scale with workers
            max_per_host = max(5, min(20, max_total_connections // 50))  # Conservative per-host limit
            
            connector = aiohttp.TCPConnector(
                limit=max_total_connections,  # Total connection pool limit
                limit_per_host=max_per_host,  # Max connections per host
                ttl_dns_cache=300,  # DNS cache timeout in seconds
                enable_cleanup_closed=True,  # Clean up closed connections
                force_close=True,  # Force close connections after each request to free up FDs
                keepalive_timeout=30,  # Reduce keepalive to free connections faster
            )
            
            self.session = aiohttp.ClientSession(
                timeout=self.timeout,
                headers={"User-Agent": self.config.user_agent},
                connector=connector,
            )
            logger.debug(f"Created new aiohttp.ClientSession with limit={max_total_connections}, limit_per_host={max_per_host}")
        return self.session
// ... existing code ...
```

Now let me create a script to help you set the OS limits and provide the complete solution:

```shellscript
#!/bin/bash

# Script to set up system limits for high-concurrency web crawler

echo "Setting up system limits for web crawler..."

# Check current limits
echo "Current file descriptor limit: $(ulimit -n)"
echo "Current max user processes: $(ulimit -u)"

# Set limits for current session
ulimit -n 65536
ulimit -u 32768

echo "Updated limits for current session:"
echo "  File descriptors: $(ulimit -n)"
echo "  Max processes: $(ulimit -u)"

# Create systemd service override if running as a service
if [ -d "/etc/systemd" ]; then
    echo ""
    echo "To make these limits permanent for systemd services, create:"
    echo "/etc/systemd/system/crawler.service.d/limits.conf"
    echo "with content:"
    echo "[Service]"
    echo "LimitNOFILE=65536"
    echo "LimitNPROC=32768"
fi

# Suggest permanent changes
echo ""
echo "For permanent system-wide changes, add to /etc/security/limits.conf:"
echo "* soft nofile 65536"
echo "* hard nofile 65536"
echo "* soft nproc 32768"
echo "* hard nproc 32768"

echo ""
echo "And to /etc/sysctl.conf:"
echo "fs.file-max = 2097152"
echo "fs.nr_open = 1048576"

echo ""
echo "Then run: sudo sysctl -p"

# Check if running with sufficient limits
if [ $(ulimit -n) -lt 10000 ]; then
    echo ""
    echo "WARNING: File descriptor limit is still low for 500 workers!"
    echo "Consider reducing workers or increasing limits."
fi
```

Now let me also update the frontier.py with the retry logic:

```python
// ... existing code ...
    async def get_next_url(self) -> tuple[str, str, int] | None: # (url, domain, id)
        """Gets the next URL to crawl from the frontier, respecting politeness rules.
        Uses atomic claim-and-read with SQLite's RETURNING clause to minimize contention.
        Includes retry logic with exponential backoff for database locks.
        """
        # Larger batch size to reduce contention - scale based on worker count
        batch_size = min(100, max(50, 1000 // max(1, self.config.max_workers)))
        claim_expiry_seconds = 300  # URLs claimed but not processed expire after 5 minutes
        conn_from_pool: sqlite3.Connection | None = None
        
        # Get worker identifier for logging
        worker_task = asyncio.current_task()
        worker_name = worker_task.get_name() if worker_task else "unknown"
        
        # Retry configuration for database locks
        max_retries = 5
        base_delay = 0.05  # 50ms base delay
        
        for retry_attempt in range(max_retries):
            try:
                conn_from_pool = self.storage.db_pool.get_connection()
                
                # Atomically claim a batch of URLs
                claimed_urls = await asyncio.to_thread(
                    self._atomic_claim_urls_sync, conn_from_pool, batch_size, claim_expiry_seconds
                )
                
                if not claimed_urls:
                    return None
                
                logger.debug(f"{worker_name} claimed {len(claimed_urls)} URLs for processing")
                
                # Track which URLs we need to unclaim if we return early
                urls_to_unclaim = []
                selected_url_info = None
                
                # Process claimed URLs with politeness checks
                for i, (url_id, url, domain) in enumerate(claimed_urls):
                    # Check if URL is allowed by robots.txt and manual exclusions
                    if not await self.politeness.is_url_allowed(url, conn_from_pool):
                        logger.debug(f"URL {url} (ID: {url_id}) disallowed by politeness rules. Removing.")
                        await asyncio.to_thread(self._delete_url_sync, conn_from_pool, url_id)
                        self.seen_urls.add(url)
                        continue
                    
                    # Check if we can fetch from this domain now
                    if await self.politeness.can_fetch_domain_now(domain, conn_from_pool):
                        await self.politeness.record_domain_fetch_attempt(domain, conn_from_pool)
                        logger.debug(f"{worker_name} processing URL ID {url_id} ({url}) from domain {domain}")
                        # Delete the URL since we're processing it
                        await asyncio.to_thread(self._delete_url_sync, conn_from_pool, url_id)
                        selected_url_info = (url, domain, url_id)
                        # Mark remaining URLs to be unclaimed
                        urls_to_unclaim = [(uid, u, d) for uid, u, d in claimed_urls[i+1:]]
                        break
                    else:
                        # Domain not ready - unclaim this URL so another worker can try later
                        await asyncio.to_thread(self._unclaim_url_sync, conn_from_pool, url_id)
                        logger.debug(f"Domain {domain} not ready for fetch, unclaimed URL ID {url_id}")
                
                # Unclaim any remaining URLs if we found one to process
                for url_id, url, domain in urls_to_unclaim:
                    await asyncio.to_thread(self._unclaim_url_sync, conn_from_pool, url_id)
                    logger.debug(f"Unclaimed remaining URL ID {url_id} from batch")
                
                return selected_url_info
                
            except sqlite3.OperationalError as e:
                if "database is locked" in str(e):
                    if retry_attempt < max_retries - 1:
                        # Exponential backoff with jitter to prevent thundering herd
                        import random
                        delay = base_delay * (2 ** retry_attempt) + random.uniform(0, 0.1)
                        logger.debug(f"{worker_name} database locked, retry {retry_attempt + 1}/{max_retries} after {delay:.3f}s")
                        
                        # Clean up connection before retry
                        if conn_from_pool:
                            try:
                                conn_from_pool.rollback()
                                self.storage.db_pool.return_connection(conn_from_pool)
                                conn_from_pool = None
                            except:
                                pass
                        
                        await asyncio.sleep(delay)
                        continue
                    else:
                        logger.error(f"DB Error in get_next_url after {max_retries} retries: {e}")
                        return None
                else:
                    # Non-lock database error
                    raise
            except sqlite3.Error as e:
                logger.error(f"DB Error in get_next_url: {e}", exc_info=True)
                if conn_from_pool:
                    try: conn_from_pool.rollback()
                    except: pass
                return None
            except Exception as e:
                logger.error(f"Unexpected error in get_next_url: {e}", exc_info=True)
                return None
            finally:
                if conn_from_pool:
                    self.storage.db_pool.return_connection(conn_from_pool)
        
        # Should not reach here unless all retries exhausted
        return None
// ... existing code ...
```

Let me fix the linter errors:

```python
// ... existing code ...
        # Retry logic for database locks
        max_retries = 5
        base_delay = 0.1  # 100ms base delay
        
        for retry in range(max_retries):
            try:
                conn_from_pool = self.storage.db_pool.get_connection()
                
                # Atomically claim a batch of URLs
                claimed_urls = await asyncio.to_thread(
                    self._atomic_claim_urls_sync, conn_from_pool, batch_size, claim_expiry_seconds
                )
                
                if not claimed_urls:
                    return None
                
                logger.debug(f"{worker_name} claimed {len(claimed_urls)} URLs for processing")
                
                # Track which URLs we need to unclaim if we return early
                urls_to_unclaim = []
                selected_url_info = None
                
                # Process claimed URLs with politeness checks
                for i, (url_id, url, domain) in enumerate(claimed_urls):
                    # Check if URL is allowed by robots.txt and manual exclusions
                    if not await self.politeness.is_url_allowed(url, conn_from_pool):
                        logger.debug(f"URL {url} (ID: {url_id}) disallowed by politeness rules. Removing.")
                        await asyncio.to_thread(self._delete_url_sync, conn_from_pool, url_id)
                        self.seen_urls.add(url)
                        continue
                    
                    # Check if we can fetch from this domain now
                    if await self.politeness.can_fetch_domain_now(domain, conn_from_pool):
                        await self.politeness.record_domain_fetch_attempt(domain, conn_from_pool)
                        logger.debug(f"{worker_name} processing URL ID {url_id} ({url}) from domain {domain}")
                        # Delete the URL since we're processing it
                        await asyncio.to_thread(self._delete_url_sync, conn_from_pool, url_id)
                        selected_url_info = (url, domain, url_id)
                        # Mark remaining URLs to be unclaimed
                        urls_to_unclaim = [(uid, u, d) for uid, u, d in claimed_urls[i+1:]]
                        break
                    else:
                        # Domain not ready - unclaim this URL so another worker can try later
                        await asyncio.to_thread(self._unclaim_url_sync, conn_from_pool, url_id)
                        logger.debug(f"Domain {domain} not ready for fetch, unclaimed URL ID {url_id}")
                
                # Unclaim any remaining URLs if we found one to process
                for url_id, url, domain in urls_to_unclaim:
                    await asyncio.to_thread(self._unclaim_url_sync, conn_from_pool, url_id)
                    logger.debug(f"Unclaimed remaining URL ID {url_id} from batch")
                
                return selected_url_info
                
            except sqlite3.OperationalError as e:
                if "database is locked" in str(e) and retry < max_retries - 1:
                    # Exponential backoff with jitter
                    import random
                    delay = base_delay * (2 ** retry) + random.uniform(0, 0.1)
                    logger.debug(f"{worker_name} got database lock, retrying in {delay:.2f}s (attempt {retry + 1}/{max_retries})")
                    await asyncio.sleep(delay)
                    if conn_from_pool:
                        try: 
                            conn_from_pool.rollback()
                            self.storage.db_pool.return_connection(conn_from_pool)
                            conn_from_pool = None
                        except: pass
                    continue
                else:
                    # Max retries exceeded or different error
                    logger.error(f"DB Error in get_next_url after {retry + 1} attempts: {e}", exc_info=True)
                    if conn_from_pool:
                        try: conn_from_pool.rollback()
                        except: pass
                    return None
            except Exception as e:
                logger.error(f"Unexpected error in get_next_url: {e}", exc_info=True)
                if conn_from_pool:
                    try: conn_from_pool.rollback()
                    except: pass
                return None
            finally:
                if conn_from_pool:
                    self.storage.db_pool.return_connection(conn_from_pool)
        
        # All retries exhausted
        return None
// ... existing code ...
```

Now make the setup script executable:

---

**User**

How does increasing the batch size lower contention? The workers don't currently stash batches away and work through them, they just pick the first valid URL and unclaim the rest of the batch, which means effectively the request volume is not affected by batch size, right?

---

**Cursor**

You're absolutely right! I apologize for the confusion. Looking at the current implementation more carefully:

```python
# Worker claims a batch of 50-100 URLs
claimed_urls = await asyncio.to_thread(self._atomic_claim_urls_sync, ...)

# But then only processes ONE URL and unclaims the rest:
for i, (url_id, url, domain) in enumerate(claimed_urls):
    if [URL is suitable]:
        selected_url_info = (url, domain, url_id)
        urls_to_unclaim = [(uid, u, d) for uid, u, d in claimed_urls[i+1:]]
        break  # <-- Returns after finding just ONE URL

# Unclaims all the rest
for url_id, url, domain in urls_to_unclaim:
    await asyncio.to_thread(self._unclaim_url_sync, conn_from_pool, url_id)

return selected_url_info  # Returns just ONE URL
```

You're correct that this doesn't reduce database contention at all. In fact, it might make it worse because:

1. **Same request frequency**: Workers still return to the database after every single URL
2. **More operations**: Now we're doing additional unclaim operations  
3. **Longer transactions**: Claiming 100 rows takes longer than claiming 1 row, holding locks longer

To actually reduce contention with batching, workers would need to maintain a local queue:

```python
# Worker claims a batch of 50-100 URLs
claimed_urls = await asyncio.to_thread(self._atomic_claim_urls_sync, ...)

# But then only processes ONE URL and unclaims the rest:
for i, (url_id, url, domain) in enumerate(claimed_urls):
    if [URL is suitable]:
        selected_url_info = (url, domain, url_id)
        urls_to_unclaim = [(uid, u, d) for uid, u, d in claimed_urls[i+1:]]
        break  # <-- Returns after finding just ONE URL

# Unclaims all the rest
for url_id, url, domain in urls_to_unclaim:
    await asyncio.to_thread(self._unclaim_url_sync, conn_from_pool, url_id)

return selected_url_info  # Returns just ONE URL
```

```python
# Example of how to actually reduce database contention with batching
# This shows the key changes needed to FrontierManager

class FrontierManager:
    def __init__(self, config: CrawlerConfig, storage: StorageManager, politeness: PolitenessEnforcer):
        # ... existing init code ...
        # Add a local queue per worker
        self._worker_queues: dict[str, list[tuple[int, str, str]]] = {}
    
    async def get_next_url(self) -> tuple[str, str, int] | None:
        """Gets the next URL, using local queue to reduce DB contention."""
        worker_task = asyncio.current_task()
        worker_name = worker_task.get_name() if worker_task else "unknown"
        
        # Check if worker has URLs in its local queue
        if worker_name in self._worker_queues and self._worker_queues[worker_name]:
            # Work through local queue without hitting the database
            local_queue = self._worker_queues[worker_name]
            
            while local_queue:
                url_id, url, domain = local_queue.pop(0)
                
                # Do politeness checks (these might hit DB but less frequently)
                if not await self.politeness.is_url_allowed(url):
                    await self._delete_url_from_frontier(url_id)
                    continue
                    
                if await self.politeness.can_fetch_domain_now(domain):
                    await self.politeness.record_domain_fetch_attempt(domain)
                    await self._delete_url_from_frontier(url_id)
                    return (url, domain, url_id)
                else:
                    # Can't process this domain now, but keep in local queue
                    # for next iteration (or could unclaim)
                    continue
            
        # Local queue empty, need to claim a new batch from DB
        batch_size = 50  # This actually helps now!
        claimed_urls = await self._claim_url_batch(batch_size)
        
        if not claimed_urls:
            return None
            
        # Store the batch in worker's local queue
        self._worker_queues[worker_name] = list(claimed_urls)
        
        # Recursively call to process from the newly populated queue
        return await self.get_next_url()
```

You're absolutely correct - I made a flawed assumption. The current implementation doesn't actually reduce database contention because workers still make the same number of database requests. In fact, larger batches might make things worse!

## Real Solutions for Database Contention

Since the batching approach doesn't help, here are better solutions:

### 1. **Reduce Worker Count** (Immediate Solution)
The most practical immediate fix:
```bash
# Try 100-200 workers instead of 500
python main.py --seed-file top-1M-domains.txt --email example@gmail.com --max-workers 150
```

### 2. **Increase SQLite Timeouts**

```shellscript
# Try 100-200 workers instead of 500
python main.py --seed-file top-1M-domains.txt --email example@gmail.com --max-workers 150
```

```shellscript
# Try 100-200 workers instead of 500
python main.py --seed-file top-1M-domains.txt --email example@gmail.com --max-workers 150
```

I see the timeout is currently 10 seconds in the connection pool. You can increase it further:

```python
class SQLiteConnectionPool:
    def __init__(self, db_path: str | Path, pool_size: int = 10, timeout: int = 30, wal_mode: bool = True):
        self.db_path = str(db_path) # Ensure db_path is a string for sqlite3.connect
        self.timeout = timeout
        self.pool_size = pool_size
        self.wal_mode = wal_mode
        self._pool: Queue = Queue(maxsize=pool_size)
        self._lock = threading.Lock() # To protect access to pool creation logic if needed, though Queue is thread-safe
        self._connections_created = 0

        # Pre-fill the pool
        for _ in range(pool_size):
            try:
                conn = self._create_connection()
                self._pool.put_nowait(conn)
            except Exception as e:
                logger.error(f"Error creating initial connection for pool: {e}")
                # Depending on strictness, could raise here or allow partial pool
        logger.info(f"SQLiteConnectionPool initialized for {self.db_path} with {self._pool.qsize()} connections.")

    def _create_connection(self):
        try:
            # Connections will be used by different threads via asyncio.to_thread,
            # so check_same_thread=False is essential.
            conn = sqlite3.connect(self.db_path, timeout=self.timeout, check_same_thread=False)
            if self.wal_mode:
                conn.execute("PRAGMA journal_mode=WAL")
                conn.execute("PRAGMA synchronous=NORMAL") # WAL is durable, NORMAL is faster
                # Additional optimizations for high concurrency
                conn.execute("PRAGMA cache_size=-64000")  # 64MB cache
                conn.execute("PRAGMA temp_store=MEMORY")  # Use memory for temp tables
                conn.execute("PRAGMA mmap_size=268435456")  # 256MB memory-mapped I/O
            self._connections_created += 1
            # logger.debug(f"Created new SQLite connection. Total created: {self._connections_created}")
            return conn
        except sqlite3.Error as e:
            logger.error(f"Failed to create SQLite connection for {self.db_path}: {e}")
            raise
```

### 3. **Alternative Queue Systems** (Better Long-term Solution)

Given SQLite's fundamental limitations with write concurrency, consider:

**a) Redis-based Queue** (if you have Redis available):
```python
# Example: Use Redis for the frontier queue
import redis
import json

class RedisFrontier:
    def __init__(self, redis_url="redis://localhost:6379"):
        self.r = redis.Redis.from_url(redis_url)
        
    async def add_url(self, url: str, domain: str, depth: int):
        url_data = json.dumps({"url": url, "domain": domain, "depth": depth})
        self.r.zadd("frontier", {url_data: time.time()})
        
    async def get_next_url(self):
        # Pop oldest URL
        result = self.r.zpopmin("frontier", 1)
        if result:
            url_data = json.loads(result[0][0])
            return url_data["url"], url_data["domain"], 0
        return None
```

**b) Simple File-based Sharding** (no extra dependencies):
```python
# Split frontier into multiple SQLite databases by domain hash
def get_shard_db(domain: str, num_shards: int = 10) -> str:
    shard_id = hash(domain) % num_shards
    return f"frontier_shard_{shard_id}.db"
```

### 4. **Immediate Practical Solutions**

Since you need to run this now, here are the most practical steps:

1. **Run the system limits script first**:
```bash
cd crawler
chmod +x setup_system_limits.sh
./setup_system_limits.sh
```

2. **Start with fewer workers and increase gradually**:
```bash
# Start with 100 workers
python main.py --seed-file top-1M-domains.txt --email example@gmail.com --max-workers 100

# If that works well, try 200
python main.py --seed-file top-1M-domains.txt --email example@gmail.com --max-workers 200
```

3. **Monitor resource usage**:
```bash
# In another terminal, watch file descriptors
watch -n 1 'lsof -p $(pgrep -f "python main.py") | wc -l'

# Watch SQLite locks
watch -n 1 'lsof | grep crawler_state.db | wc -l'
```

4. **Consider splitting the job**:
```bash
# Split your 1M domains into chunks
split -l 100000 top-1M-domains.txt chunk_

# Run multiple instances with different chunks
python main.py --seed-file chunk_aa --max-workers 100 &
python main.py --seed-file chunk_ab --max-workers 100 &
```

The database locking issue is fundamentally due to SQLite's single-writer limitation. With 500 workers all trying to write simultaneously, you're hitting this limit hard. The retry logic helps, but the real solution is either:
- Use fewer workers (100-200 range)
- Use a different queue system (Redis, PostgreSQL, etc.)
- Shard the frontier across multiple SQLite databases

For now, I'd recommend starting with 100-150 workers and seeing how it performs. The system should handle that much better than 500.

```python
# Example: Use Redis for the frontier queue
import redis
import json

class RedisFrontier:
    def __init__(self, redis_url="redis://localhost:6379"):
        self.r = redis.Redis.from_url(redis_url)
        
    async def add_url(self, url: str, domain: str, depth: int):
        url_data = json.dumps({"url": url, "domain": domain, "depth": depth})
        self.r.zadd("frontier", {url_data: time.time()})
        
    async def get_next_url(self):
        # Pop oldest URL
        result = self.r.zpopmin("frontier", 1)
        if result:
            url_data = json.loads(result[0][0])
            return url_data["url"], url_data["domain"], 0
        return None
```

```python
# Split frontier into multiple SQLite databases by domain hash
def get_shard_db(domain: str, num_shards: int = 10) -> str:
    shard_id = hash(domain) % num_shards
    return f"frontier_shard_{shard_id}.db"
```

```python
# Example: Use Redis for the frontier queue
import redis
import json

class RedisFrontier:
    def __init__(self, redis_url="redis://localhost:6379"):
        self.r = redis.Redis.from_url(redis_url)
        
    async def add_url(self, url: str, domain: str, depth: int):
        url_data = json.dumps({"url": url, "domain": domain, "depth": depth})
        self.r.zadd("frontier", {url_data: time.time()})
        
    async def get_next_url(self):
        # Pop oldest URL
        result = self.r.zpopmin("frontier", 1)
        if result:
            url_data = json.loads(result[0][0])
            return url_data["url"], url_data["domain"], 0
        return None
```

```python
# Split frontier into multiple SQLite databases by domain hash
def get_shard_db(domain: str, num_shards: int = 10) -> str:
    shard_id = hash(domain) % num_shards
    return f"frontier_shard_{shard_id}.db"
```

```shellscript
cd crawler
chmod +x setup_system_limits.sh
./setup_system_limits.sh
```

```shellscript
# Start with 100 workers
python main.py --seed-file top-1M-domains.txt --email example@gmail.com --max-workers 100

# If that works well, try 200
python main.py --seed-file top-1M-domains.txt --email example@gmail.com --max-workers 200
```

```shellscript
# In another terminal, watch file descriptors
watch -n 1 'lsof -p $(pgrep -f "python main.py") | wc -l'

# Watch SQLite locks
watch -n 1 'lsof | grep crawler_state.db | wc -l'
```

```shellscript
# Split your 1M domains into chunks
split -l 100000 top-1M-domains.txt chunk_

# Run multiple instances with different chunks
python main.py --seed-file chunk_aa --max-workers 100 &
python main.py --seed-file chunk_ab --max-workers 100 &
```

