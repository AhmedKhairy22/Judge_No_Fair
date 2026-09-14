# Deep Dives

For the main components, we looked at the problems we can face when the system grows, the different options, and why we decided to use a specific one.

For the main database, we decided to use PostgreSQL as the source of truth. At 1M+ users and around 100k active jobs, most of the traffic will still be reads, while writes like applications are much lower. We considered read replicas, sharding, and NewSQL databases. Sharding can give us more write scalability, but it makes transactions and things like the `(CandidateID, JobID)` uniqueness constraint more complicated. NewSQL can help with this but adds more operational complexity.

So for now we chose one PostgreSQL primary with read replicas and PgBouncer. From our estimation, the write volume is still far below what PostgreSQL can handle, so sharding now would just add complexity without solving a real problem. We can revisit it if the write volume grows a lot. We can also partition `ApplicationStatusHistory` by time since it is append-only and will grow faster than the other tables.

For communication between the core system and background services, we need an event bus because changes have to reach Matching, Skill-Gap, Notifications, and Search without making the main request wait. We considered dual-write and CDC. With dual-write, the database update can succeed while publishing the event fails, which can leave us with missing events.

We chose CDC so the events come from changes that were actually committed to the database. For the broker, we chose Kafka because it keeps the events and allows consumers to replay them later. This is useful if a service goes down or if we need to rebuild something like the search index.

For search, we don't want to depend only on PostgreSQL. Job and candidate search needs text search and filtering by skills, salary, experience, and location, and the system will have much more reads than writes. So we use OpenSearch and update it asynchronously through the event bus.

We also noticed that our diagram was showing the Skill-Gap Engine updating the search index, which doesn't really make sense. The Skill-Gap Engine should only compare candidate skills with job requirements. So we added a separate Search Indexer that consumes events and updates OpenSearch. We accept some eventual consistency here because a new job might take a short time to appear in search, but this gives us better search performance.

For matching, the main problem is that comparing candidates and jobs is a many-to-many operation. Running this directly in an API request would become very expensive. We decided to use incremental event-driven matching instead. A new job triggers matching only for relevant candidates, while a profile update recalculates recommendations for that candidate. We also run a periodic full batch as a backup in case of missed events or other problems. The recommendation results are cached in Redis because they will be read much more often than they change.

Skill-gap analysis is different. It is basically comparing the candidate's current skills with the skills required for one job, so we kept it synchronous. There is no real reason to precompute it at this stage, and doing it on request means we are using the latest data. If later we need to calculate skill gaps for many candidates at once, we can move that workload to an async process.

For notifications, a popular job can generate many events at the same time and external providers have rate limits. We also need to avoid sending the same notification twice. We decided to store the notification first with an idempotency key like `(UserID, EventID)`, then deliver it asynchronously with retries and backoff. This makes retries safer and gives us better control over the external providers.

We also considered adding a general Redis cache in front of the core services for things like popular jobs. But this would add more cache invalidation and stale-data problems. Since we already have read replicas and don't have evidence that they won't meet the latency target, we decided to leave the general read cache out of v1. If monitoring later shows that database reads are becoming the bottleneck, we can add it without changing the overall architecture.
