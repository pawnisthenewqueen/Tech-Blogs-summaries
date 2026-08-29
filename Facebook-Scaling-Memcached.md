# 📝 Tech Blog Insights: Caching & System Scale
**Case Study:** Scaling Memcached at Facebook (Meta)
**Topic Focus:** Caching, Consistent Hashing, Thundering Herds, and Race Conditions

## 1. The Core Problem: Databases are Slow
Databases (like SQL) read from physical storage (SSDs/Hard Drives). While reliable, they are relatively slow. If 1 billion users open an app simultaneously and hit the database, it will instantly overload and crash. The database is always the bottleneck in a distributed system.

## 2. The Solution: In-Memory Caching (Memcached)
To protect the database, engineers put a **Cache** in front of it. A cache is essentially a massive Key-Value hash table (like a C++ `std::unordered_map`) that stores data entirely in RAM, making it blisteringly fast.
*   **Cache Hit:** If data is in the cache, it is returned instantly. The database does nothing.
*   **Cache Miss:** If data is missing, the system fetches it from the database, returns it to the user, and *writes a copy to the cache* for the next user.

## 3. Scaling the Cache: Consistent Hashing
A single server cannot hold Facebook's petabytes of data in RAM. They use thousands of cache servers. 
*   **The Routing Problem:** How do you know which server has the data? You can't ask all of them.
*   **The Solution:** An algorithmic concept called **Consistent Hashing**. By hashing the key (e.g., `hash("post_123") % 1000`), the system knows exactly which of the 1,000 servers holds that specific piece of data. 

## 4. The Boss Fight: The "Thundering Herd" Problem
Data in a cache has a Time-To-Live (TTL). Eventually, it deletes itself to keep data fresh.
If a viral post's TTL expires, the cache deletes it. In that exact millisecond, 100,000 users might request the post. They all get a Cache Miss, and **all 100,000 requests hit the SQL database at the exact same time**, crashing it instantly. This is a "Thundering Herd".

## 5. Facebook's Fix: Leases (Mutex Locks for the Web)
To solve the Thundering Herd, Facebook modified Memcached to hand out **Lease Tokens**. 
*   When the 100,000 requests hit the empty cache, Memcached gives a Lease Token to the *very first request* and tells it to fetch the data from the DB.
*   The other 99,999 requests are told to wait for a few milliseconds. 
*   The first request updates the cache, and the waiting requests can now safely read from RAM. The database survives.

## 6. The "Edit" Problem & Race Conditions
When a user edits a post, the database updates, but the cache holds the old data. You must explicitly delete the cache entry (Cache Invalidation). 
*   **The "Stale Set" Race Condition:** User A fetches the *old* data from the database. User B edits the post and deletes the cache. User A's slow network finishes, and they blindly insert the *old* data back into the cache, permanently corrupting it.
*   **The Fix:** Leases solve this too. When User B edits the post, Memcached voids User A's Lease Token. When User A tries to insert the old data, Memcached rejects it because the token is invalid.

## 7. High-Level Architectural Takeaways
1.  **Protect the DB at all costs:** In high-scale systems, the database should only handle a tiny fraction of total read traffic.
2.  **Concurrency creates chaos:** When thousands of servers act at the exact same millisecond, simple actions (like a cache expiring) create massive race conditions that require low-level lock mechanics (like Leases) to solve safely.
3.  
