# Coding / DSA Problems

Pure coding problems from the LinkedIn batch — these are practice problems, not conceptual Q&A. Solve each yourself; bring your solution for review rather than looking for a written answer here. A short hint is included where the "trick" is non-obvious.

## Longest substring without repeating characters
Input example: `"ABCABCBB"`. Find the length/content of the longest substring with no repeated characters.
**Hint:** sliding window with a `HashSet`/`HashMap` tracking the last-seen index of each character; shrink the window from the left when a repeat is found instead of restarting from scratch.

## Design a rate limiter (100 requests/user/minute)
See the full design discussion in `microservices-system-design.md` — implement it: pick fixed-window or sliding-window, make it thread-safe, and note how it'd need to change to work across multiple app instances.

## Cinema scheduling system (Atlassian round)
Operating hours 10 AM–11 PM. Implement `canSchedule(movie, schedule)` without overlapping bookings. Follow-ups: replace a movie in an existing slot, calculate total revenue.
**Hint:** model as interval scheduling — sort by start time, check overlap against already-booked intervals; a `TreeMap<startTime, booking>` makes conflict lookup and the "replace" follow-up easier than a plain list.

## upsertCommodityPrice(timestamp, price) / getMaxCommodityPrice() (Atlassian round)
Must handle out-of-order timestamp updates correctly. Example: after updates, prices are 20, 26, 25 — max is 26.
**Hint:** a `TreeMap<timestamp, price>` for the upserts (handles out-of-order + overwrite-by-timestamp cleanly) plus a separate max-tracking structure (e.g. a `TreeMap<price, count>` or a max-heap with lazy deletion) if `getMaxCommodityPrice()` needs to stay O(log n) after updates/removals.

## Design a popular feed for Confluence (Atlassian round)
Track views/likes/shares, calculate top N pages, support real-time updates and high traffic.
**Hint:** this is a system design question more than pure DSA — think in terms of a write path (increment counters, likely in a fast store like Redis sorted sets) and a separate read path (periodically materialized "top N" rather than computing it from scratch on every read).

## String challenge: duplicate characters + frequency, special character count
Input example: `"dineshs.&*"`. Find duplicate characters and their frequency; separately count only the special characters present.
**Hint:** a single pass building a `HashMap<Character, Integer>` frequency count covers the duplicates; filter the same map (or a second pass) against a definition of "special character" (not alphanumeric) for the count.

## Second-highest (and third-highest) salary per department using Java 8 Streams
Given a list of employees with department and salary, find the second-highest distinct salary within each department, without explicit loops.
**Hint:** `Collectors.groupingBy(Employee::getDepartment, Collectors.mapping(Employee::getSalary, Collectors.toCollection(TreeSet::new)))` gives a per-department sorted, de-duplicated set of salaries; pulling the 2nd-from-top (or 3rd, for the follow-up) from each `TreeSet` answers the question. Handle departments with fewer than 2 (or 3) distinct salaries with `Optional` rather than throwing.
