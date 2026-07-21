# Consistent Hashing — Detailed Notes

## 1. Understanding Hash Functions

- A hash function takes an input and maps it to a specific output.
- It is commonly used for distributing data efficiently in **load balancing**, **caching**, and **sharding**.

## 2. Normal Hashing (Modulo-Based Hashing)

**How it works**

Given `N` servers, data is assigned using:

```
server_index = hash(input) mod N
```

*Example:* With `N = 3` servers and input `X`, compute `hash(X) mod 3`. The result determines which of the 3 servers handles the request.

**Challenges**

- When a new server is added, we now take the modulo by `N + 1` instead of `N`.
- This causes **cache misses**, because:
  - Data previously mapped to one server is now mapped to a different one.
  - Rebalancing is required to reassign data correctly.

## 3. The Problem of Rebalancing

- **Rebalancing** means redistributing data when servers are added or removed.
- **Issue:** with normal hashing, *most* of the data must be reassigned.
- **Solution:** consistent hashing minimizes rebalancing.

## 4. Introduction to Consistent Hashing

The goal of consistent hashing is to **minimize the number of keys that need to be reassigned** when a server is added or removed.

In a well-designed system, only `1/N` of the total data needs to be reassigned on each such change.

### Key Properties

**Minimal rebalancing**

- Only `1/N` of the total data needs reassignment when servers change.
- Rebalancing formula:

  ```
  records_moved = (1 / N) × (total number of records)
  ```

  where `N` is the number of servers.

**Works well for dynamic scaling**

- Suitable when servers are frequently added or removed.
- Ideal for load balancing and database sharding.

## 5. How Consistent Hashing Works

**Virtual ring structure**

- Servers and data are arranged on a circular (virtual) **hash ring**.
- Example: a ring with 12 positions.
- Each server is mapped to specific points on the ring.

**Data assignment**

- A request (or key) is mapped to a position on the ring.
- The request is assigned to the **next available server in the clockwise direction**.

**Adding and removing servers**

- When a new server is added, it takes over responsibility for a small portion of the ring.
- Only the affected portion of data is reassigned — instead of all data, as in normal hashing.

## 6. Virtual Nodes

- **Issue:** uneven data distribution when servers are not evenly spaced on the ring.
- **Solution — virtual nodes:**
  - Instead of placing one server at a single location, each server is mapped to *multiple* points on the ring.
  - This ensures a balanced distribution of requests across servers.

## 7. Use Cases

**Load balancing**

- Distributes incoming requests evenly across multiple app servers.
- Minimizes disruption when scaling up or down.

**Distributed caching**

- Used in Redis and Memcached to efficiently map data to cache nodes.
- Prevents frequent cache misses.

**Database sharding**

- Helps with horizontal sharding by dynamically assigning data to different database partitions.