# Back-of-the-Envelope Estimation

We start with the main assumptions from our Non-Functional Requirements: around 1M registered users, 100k active jobs, and roughly a 100:1 read-to-write ratio.

If we assume around 10% of users are active each day, we get about 100k Daily Active Users. If each user performs around 8 searches per day, that gives us around 800k searches/day. Using the 100:1 search-to-application ratio, this is around 8k applications/day.

That gives us roughly 9 Queries Per Second for search on average. If we assume around 3x peak traffic during busy hours, it can reach around 28 Queries Per Second. For applications, the average is only around 0.09 QPS and around 1 QPS at peak.

This is also one reason we don't think we need database sharding from the beginning. Even at peak, the write traffic is very small for a PostgreSQL primary. The bigger concern is the search workload and keeping filtered and ranked queries within the 300ms 95th Percentile target.

For storage, over around 2–3 years, we estimate roughly 1 GB for users, 3 GB for candidate profiles, around 2 GB for jobs, around 3 GB for applications, and around 7 GB for application status history. So the core relational data should be around 15–20 GB. This is still very manageable for a single PostgreSQL instance, and again the main pressure point is not storage or write throughput.

For notifications, 8k applications per day with around 4 status notifications each gives us around 32k notifications. Adding match notifications, we estimate around 60k notifications per day in total, which is roughly 1 QPS on average. So our own infrastructure should handle this easily, and the bigger limitation will probably be the rate limits of the external email or push providers during bursts.

Overall, these numbers don't really justify sharding, moving to NewSQL, or adding aggressive caching from day one. The main thing we need to optimize is search: around 9–28 QPS of filtered and ranked queries over 100k+ jobs while keeping the 300ms p95 target. This is why we put more design effort into OpenSearch and asynchronous indexing rather than making the transactional database more complicated.
